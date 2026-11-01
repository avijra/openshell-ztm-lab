# Architecture

## Overall System Architecture

The demo runs on a ROSA cluster with the Zero Trust Validated Pattern (ZTVP) as the foundation. OpenShell is deployed as a Helm chart on top.

```mermaid
graph TB
    subgraph ROSA["ROSA Cluster (AWS)"]
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
    SB -->|"✅ allowed"| DS["DeepSeek API"]
    SB -->|"✅ allowed"| NPM["npm"]
    SB -.-x|"❌ blocked"| Bad["google.com"]
    Vault -.->|secret sync| GW
    SPIRE -.->|identity| GW
    ACS -.->|image scan| SB
```

## Request Flow

When a user asks Pi to write code, the request passes through four layers before reaching the LLM.

```mermaid
graph TD
    A["👤 User on Bastion<br/><i>openshell sandbox exec ... pi -p 'Write code'</i>"]
    B["🔗 Gateway<br/>Authenticates the request<br/>routes to correct sandbox"]
    C["👁️ Supervisor<br/>Enforces boundary policy<br/>monitors the sandbox"]
    D["📦 Sandbox (Pi Agent)<br/>Runs prompt through Pi<br/>Pi calls DeepSeek for code"]
    E["🔒 Boundary Check<br/>Is api.deepseek.com in allowlist? → ✅ Yes<br/>Resolve credential token → real API key"]
    F["🤖 DeepSeek API<br/>Returns generated code"]

    A -->|"gRPC over TLS"| B
    B -->|"forward"| C
    C -->|"exec in sandbox"| D
    D -->|"HTTPS request"| E
    E -->|"real API key injected"| F
    F -->|"response flows back same path"| A

    style E fill:#fff3cd,color:#856404
    style F fill:#d4edda,color:#155724
```

**Key point:** The sandbox never sends the real API key. The Boundary intercepts the request, swaps the resolver token for the real key, and forwards it. The response travels back the same path.

## Credential Flow

API keys are **never visible** inside the sandbox. They pass through three phases:

**Phase 1 — Install time** (Helm chart hooks)

```mermaid
graph LR
    U["👤 User<br/>provides API key"] -->|"helm install --set deepseek.apiKey=sk-..."| VJ["Vault Seed Job"]
    VJ -->|"stores key at"| V["🔐 Vault<br/>secret/openshell/deepseek-api-key"]
    VJ -->|"also creates"| KS["K8s Secret<br/>(fallback)"]
```

**Phase 2 — Runtime sync** (ESO keeps secrets fresh)

```mermaid
graph LR
    V["🔐 Vault"] -->|"ESO reads every 1h"| ESO["🔄 External Secrets Operator"]
    ESO -->|"writes to"| KS["K8s Secret"]
    KS -->|"Gateway reads"| GW["Gateway<br/>registers credential<br/>with provider"]
```

**Phase 3 — Every API call** (Boundary resolves at the edge)

```mermaid
graph LR
    SB["📦 Sandbox<br/>sees: openshell:resolve:env:v1:..."]
    SB -->|"sends resolver token"| BND["🔒 Boundary"]
    BND -->|"swaps token for real key"| DS["🤖 DeepSeek API<br/>receives: sk-abc123..."]

    style SB fill:#f8d7da,color:#721c24
    style BND fill:#fff3cd,color:#856404
    style DS fill:#d4edda,color:#155724
```

> The sandbox **never** handles the real key. Even if compromised, the resolver token is useless outside the OpenShell boundary.

## Zero Trust Policy Enforcement

Every outbound request is checked. There is no default-allow.

```mermaid
graph TD
    P["Process in Sandbox<br/>makes a network request"] --> Q{"Is the destination host<br/>in the allowlist?"}
    Q -->|"Yes"| A["✅ Forward request"]
    Q -->|"No"| D["❌ Silently drop<br/>(connection times out)"]

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

## Components Summary

| Component | Role | Namespace |
|-----------|------|-----------|
| **OpenShell Gateway** | gRPC API, policy engine, credential resolver | `openshell-system` |
| **Supervisor** | Monitors sandbox, enforces boundary | `openshell-system` |
| **Sandbox** | Isolated container running Pi agent | `openshell-system` |
| **Vault** | Stores API keys, issues dynamic secrets | `vault` |
| **ESO** | Syncs Vault secrets → K8s Secrets | `external-secrets-operator` |
| **SPIFFE/SPIRE** | Issues cryptographic workload identities (SVIDs) | `spire-server` |
| **ACS** | Scans container images for CVEs, enforces policies | `stackrox` |
| **Keycloak** | SSO / OIDC identity provider | `keycloak-system` |
