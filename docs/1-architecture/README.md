# Architecture

Before we start deploying, let's understand how all the pieces fit together. This module is **read-only** — no commands to run, just context that will make every later step make sense.

## The Big Picture

The demo runs on an **OpenShift cluster on AWS** with two layers:

1. **Foundation layer: ZTVP** — The Zero Trust Validated Pattern provides five security services (Vault, ESO, Keycloak, SPIFFE/SPIRE, ACS) that are already deployed and running. Think of ZTVP as the "security platform" that any application can plug into.

2. **Application layer: OpenShell** — NVIDIA's AI agent framework, deployed as a Helm chart on top of ZTVP. OpenShell adds the sandbox, the policy engine, and the credential resolver.

```mermaid
graph TB
        subgraph OCP["OpenShift Cluster (AWS)"]
        subgraph ZTVP["Zero Trust Validated Pattern"]
            Vault["🔐 Vault"]
            ESO["🔄 External Secrets"]
            Keycloak["🔑 Keycloak"]
            SPIRE["🪪 SPIFFE / SPIRE"]
            ACS["🛡️ ACS"]
        end
        subgraph OS["OpenShell"]
            GW["Gateway"]
            SV["Supervisor"]
            SB["Sandbox"]
        end
    end
    User["👤 User"] -->|CLI| GW
    GW --> SV --> SB
    SB -->|"✅ allowed"| MAAS["MAAS API"]
    SB -->|"✅ allowed"| NPM["npm"]
    SB -.-x|"❌ blocked"| Bad["google.com"]
    Vault -.->|secret sync| GW
    SPIRE -.->|identity| GW
    ACS -.->|image scan| SB
```

**Reading the diagram:**
- The **User** (you, on the bastion) sends commands via the OpenShell CLI to the **Gateway**
- The Gateway forwards to the **Supervisor**, which manages the **Sandbox** (where Pi runs)
- The Sandbox can reach **MAAS** and **npm** (green arrows) but **not** google.com (red X)
- ZTVP components (dotted arrows) provide secrets, identity, and image scanning behind the scenes

## What Is a "Sandbox"?

In OpenShell, a **sandbox** is an isolated container where an AI agent runs. It's similar to a Kubernetes pod, but with extra security layers:

| Feature | Regular K8s Pod | OpenShell Sandbox |
|---------|----------------|-------------------|
| Network access | Unrestricted (unless NetworkPolicy) | **Default-deny** — only explicit allowlist |
| Filesystem access | Full container filesystem | **Landlock LSM** — only listed paths |
| Credentials | Mounted as env vars or files (real values) | **Resolver tokens** — real keys never enter |
| Binary restrictions | None | Per-endpoint binary allowlists |
| Policy updates | Requires pod restart | **Live updates** via CLI |

The sandbox is where Pi lives. Everything Pi does — every file it writes, every network call it makes — is governed by the sandbox's policy.

## Request Flow

When you ask Pi to write code, here's what happens step by step:

```mermaid
graph TD
    A["👤 User on Bastion<br/><i>openshell sandbox exec ... pi -p 'Write code'</i>"]
    B["🔗 Gateway<br/>Authenticates the request<br/>routes to correct sandbox"]
    C["👁️ Supervisor<br/>Enforces boundary policy<br/>monitors the sandbox"]
    D["📦 Sandbox (Pi Agent)<br/>Runs prompt through Pi<br/>Pi calls MAAS for code"]
    E["🔒 Boundary Check<br/>Is maas-rhdp.apps.maas.redhatworkshops.io in allowlist? → ✅ Yes<br/>Resolve credential token → real API key"]
    F["🤖 MAAS API<br/>Returns generated code"]

    A -->|"gRPC over TLS"| B
    B -->|"forward"| C
    C -->|"exec in sandbox"| D
    D -->|"HTTPS request"| E
    E -->|"real API key injected"| F
    F -->|"response flows back same path"| A

    style E fill:#fff3cd,color:#856404
    style F fill:#d4edda,color:#155724
```

**Walking through it:**

1. **You type a command** on the bastion: `openshell sandbox exec pi-agent-demo -- pi -p "Write code"`
2. The CLI sends a **gRPC request** (a high-performance RPC protocol) over **TLS** (encrypted) to the Gateway
3. The **Gateway** authenticates your request and finds the right sandbox
4. The **Supervisor** — a process that runs *outside* the sandbox and monitors it — receives the command
5. The Supervisor **executes** your command inside the sandbox. Pi starts running.
6. Pi calls MAAS. The HTTPS request hits the **Boundary** (the Supervisor's network filter)
7. The Boundary checks: "Is `maas-rhdp.apps.maas.redhatworkshops.io` in the allowlist?" → **Yes**. It also swaps the resolver token for the real API key.
8. The request reaches **MAAS**, which returns generated code
9. The response flows back the same path to your terminal

**The critical insight:** The sandbox never sends the real API key. The Boundary intercepts the request, swaps the resolver token for the real key, and forwards it. If the sandbox were compromised, the attacker would only have a useless token.

## Credential Flow

API keys are **never visible** inside the sandbox. They pass through three phases:

**Phase 1 — Install time** (when you install the Helm chart)

```mermaid
graph LR
    U["👤 User<br/>provides API key"] -->|"helm install --set maas.apiKey=sk-..."| VJ["Vault Seed Job"]
    VJ -->|"stores key at"| V["🔐 Vault<br/>secret/openshell/maas-api-key"]
    VJ -->|"also creates"| KS["K8s Secret<br/>(fallback)"]
```

Your MAAS API key goes into **Vault** (HashiCorp's secret management system) via a Kubernetes Job that runs before anything else starts. Vault encrypts it at rest and controls who can access it.

**Phase 2 — Runtime sync** (ESO keeps secrets fresh)

```mermaid
graph LR
    V["🔐 Vault"] -->|"ESO reads every 1h"| ESO["🔄 External Secrets Operator"]
    ESO -->|"writes to"| KS["K8s Secret"]
    KS -->|"Gateway reads"| GW["Gateway<br/>registers credential<br/>with provider"]
```

The **External Secrets Operator** (ESO) watches Vault and automatically syncs the key into a Kubernetes Secret. The Gateway reads this Secret and registers it as a "provider credential." This sync happens every hour, so if you rotate the key in Vault, it propagates automatically.

**Phase 3 — Every API call** (Boundary resolves at the edge)

```mermaid
graph LR
    SB["📦 Sandbox<br/>sees: openshell:resolve:env:v1:..."]
    SB -->|"sends resolver token"| BND["🔒 Boundary"]
    BND -->|"swaps token for real key"| MAAS["🤖 MAAS API<br/>receives: sk-abc123..."]

    style SB fill:#f8d7da,color:#721c24
    style BND fill:#fff3cd,color:#856404
    style MAAS fill:#d4edda,color:#155724
```

When Pi makes an API call, the `Authorization` header contains a **resolver token** (like `openshell:resolve:env:v123_MAAS_API_KEY`). The Boundary intercepts this, looks up the real key, and replaces the token before the request leaves the cluster. The sandbox never touches the real key.

> **Why this matters:** Even if an attacker compromises the sandbox, they get a resolver token that is (a) bound to a specific endpoint, (b) only resolvable inside the cluster boundary, and (c) useless anywhere else.

## Zero Trust Policy Enforcement

Every outbound request is checked. There is no default-allow.

```mermaid
graph TD
    P["Process in Sandbox<br/>makes a network request"] --> Q{"Is the destination host<br/>in the allowlist?"}
    Q -->|"Yes"| A["✅ Forward request"]
    Q -->|"No"| D["❌ Instantly denied<br/>(connection refused in ~2ms)"]

    P2["Process in Sandbox<br/>opens a file"] --> Q2{"Is the path in<br/>read_only or read_write?"}
    Q2 -->|"read_only"| R["✅ Read allowed, write denied"]
    Q2 -->|"read_write"| RW["✅ Read + write allowed"]
    Q2 -->|"Neither"| FD["❌ Permission denied (EACCES)"]

    style A fill:#d4edda,color:#155724
    style R fill:#d4edda,color:#155724
    style RW fill:#d4edda,color:#155724
    style D fill:#f8d7da,color:#721c24
    style FD fill:#f8d7da,color:#721c24
```

**Two enforcement mechanisms work together:**

1. **Network policy** (enforced by the Supervisor/Boundary): Controls which hosts the sandbox can reach, down to the specific binary making the call. This is *not* a Kubernetes NetworkPolicy — it's enforced by OpenShell at the process level.

2. **Filesystem policy** (enforced by Landlock LSM): Controls which paths the sandbox can read/write. **Landlock** is a Linux kernel security module — it's enforced by the kernel itself, not by userspace code. This means even if the sandboxed process has root privileges, it *still* cannot access restricted paths.

## Components Summary

| Component | What it is | Role in this demo | Namespace |
|-----------|-----------|-------------------|-----------|
| **OpenShell Gateway** | gRPC API server | Receives CLI commands, resolves credentials, applies policy | `openshell-system` |
| **Supervisor** | Sidecar process | Runs outside the sandbox, enforces the network boundary, monitors the agent | `openshell-system` |
| **Sandbox** | Isolated container | Where Pi runs — the agent's entire world | `openshell-system` |
| **Vault** | Secret management (HashiCorp) | Stores and encrypts the MAAS API key | `vault` |
| **ESO** | External Secrets Operator | Syncs secrets from Vault → Kubernetes Secrets automatically | `external-secrets` |
| **SPIFFE/SPIRE** | Workload identity framework | Issues cryptographic certificates (SVIDs) to prove workload identity | `zero-trust-workload-identity-manager` |
| **ACS** | Advanced Cluster Security (StackRox) | Scans container images for CVEs, enforces security policies | `stackrox` |
| **Keycloak** | Identity provider (SSO/OIDC) | Authenticates users, issues JWT tokens | `keycloak-system` |

## What Makes This "Zero Trust"?

Traditional security trusts the network perimeter. Zero Trust trusts nothing:

| Traditional | Zero Trust (this lab) |
|------------|----------------------|
| "Inside the cluster = trusted" | Every request is verified |
| Credentials in env vars | Credentials never enter the sandbox |
| Network access: allow by default | Network access: deny by default |
| Filesystem: full access | Filesystem: kernel-enforced allowlist |
| Identity: "you're in the right namespace" | Identity: cryptographic SPIFFE certificates |
| Image security: scan once at build | Image security: continuous scanning by ACS |

## Next Step

Now that you understand the architecture, let's verify your environment is ready.

Proceed to: **Prerequisites** →
