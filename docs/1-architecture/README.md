# Architecture

## Overall System Architecture

The demo runs on a ROSA cluster with the Zero Trust Validated Pattern (ZTVP) as the foundation. OpenShell is deployed as a Helm chart on top.

```mermaid
graph TB
    subgraph ROSA["ROSA Cluster (AWS)"]
        subgraph ZTVP["Zero Trust Validated Pattern"]
            Vault["🔐 HashiCorp Vault<br/>Secret Management"]
            ESO["🔄 External Secrets Operator<br/>Secret Sync"]
            Keycloak["🔑 Keycloak<br/>Identity & SSO"]
            SPIRE["🪪 SPIFFE/SPIRE<br/>Workload Identity"]
            ACS["🛡️ Red Hat ACS<br/>Image Scanning"]
        end
        subgraph OS["OpenShell (openshell-system)"]
            GW["⚡ Gateway<br/>gRPC + Policy Engine"]
            SV["👁️ Supervisor<br/>Boundary Enforcement"]
            SB["📦 Sandbox<br/>Pi Agent + Node.js"]
        end
    end
    User["👤 User (Bastion)"] -->|OpenShell CLI| GW
    GW -->|creates & manages| SV
    SV -->|enforces policy on| SB
    SB -->|allowed by policy| DS["🤖 DeepSeek API"]
    SB -->|allowed by policy| GH["🐙 GitHub"]
    SB -->|allowed by policy| NPM["📦 npm"]
    SB -.-x|BLOCKED| Google["❌ google.com"]
    SB -.-x|BLOCKED| Evil["❌ evil-server.com"]
    GW -->|reads secrets| Vault
    ESO -->|syncs| Vault
    ACS -->|scans| SB
    SPIRE -->|issues SVID| GW
```

## Request Flow: User → Agent → LLM

When a user asks Pi to write code, this is the complete request flow:

```mermaid
sequenceDiagram
    participant U as 👤 User (Bastion)
    participant CLI as OpenShell CLI
    participant GW as Gateway
    participant SV as Supervisor
    participant SB as Sandbox (Pi)
    participant BN as Boundary
    participant DS as DeepSeek API

    U->>CLI: openshell sandbox exec pi-agent-demo -- pi -p "Write code"
    CLI->>GW: gRPC ExecInSandbox (TLS)
    GW->>GW: Authenticate request
    GW->>SV: Forward exec to supervisor
    SV->>SB: Execute command in sandbox
    SB->>SB: Pi processes prompt
    SB->>BN: HTTPS to api.deepseek.com
    BN->>BN: Check network policy ✅ ALLOWED
    BN->>BN: Resolve credential token → real API key
    BN->>DS: HTTPS with real API key
    DS-->>BN: LLM response
    BN-->>SB: Response (credential stripped)
    SB->>SB: Pi writes code, runs it
    SB-->>SV: Output
    SV-->>GW: Output
    GW-->>CLI: gRPC response
    CLI-->>U: Display result
```

## Credential Flow: API Key → Vault → Sandbox

API keys are **never visible** inside the sandbox. Here is how they flow:

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant Helm as Helm Chart
    participant Vault as 🔐 Vault
    participant ESO as 🔄 ESO
    participant K8s as K8s Secret
    participant GW as Gateway
    participant SB as Sandbox

    User->>Helm: Install chart with --set deepseek.apiKey=sk-xxx
    Helm->>Vault: Pre-install hook: store key in Vault
    Helm->>K8s: Also create K8s Secret (fallback)
    ESO->>Vault: Sync secret from Vault path
    ESO->>K8s: Update K8s Secret
    GW->>K8s: Read API key from Secret
    GW->>GW: Register as OpenShell Provider credential
    Note over GW: Real key lives here only
    GW->>SB: Inject env: DEEPSEEK_API_KEY=openshell:resolve:env:...
    Note over SB: Sandbox sees resolver TOKEN, never the real key
    SB->>SB: Pi uses $DEEPSEEK_API_KEY in HTTP request
    SB->>GW: Request with resolver token in Authorization header
    GW->>GW: Boundary intercepts, replaces token with real key
    GW->>DS: Forward request with real API key
```

## Zero Trust Policy Enforcement

Every network request and filesystem access is checked against the policy:

```mermaid
flowchart LR
    SB["Sandbox Process"] --> BN{"Boundary<br/>Policy Check"}
    BN -->|"host in allowlist?"| ALLOW["✅ Forward Request"]
    BN -->|"host NOT in allowlist?"| DENY["❌ Silent Drop"]
    BN -->|"file in read_only list?"| RO["✅ Read Allowed"]
    BN -->|"file in read_write list?"| RW["✅ Read/Write Allowed"]
    BN -->|"file NOT in any list?"| FD["❌ Permission Denied"]
    
    style ALLOW fill:#c6f6d5,color:#22543d
    style RO fill:#c6f6d5,color:#22543d
    style RW fill:#c6f6d5,color:#22543d
    style DENY fill:#fed7d7,color:#742a2a
    style FD fill:#fed7d7,color:#742a2a
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
