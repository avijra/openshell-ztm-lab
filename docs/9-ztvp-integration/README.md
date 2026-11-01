# ZTVP Integration

In this module you will explore how the Zero Trust Validated Pattern components work together with OpenShell to form a defense-in-depth security architecture.

## The Five Pillars

The ZTVP provides five security capabilities that integrate with OpenShell:

```mermaid
graph TB
    subgraph ZTVP["Zero Trust Validated Pattern"]
        V["🔐 Vault<br/>Secret Management"]
        E["🔄 External Secrets<br/>K8s Secret Sync"]
        S["🪪 SPIFFE/SPIRE<br/>Workload Identity"]
        A["🛡️ ACS / StackRox<br/>Image Security"]
        K["🔑 Keycloak<br/>User Identity"]
    end
    
    subgraph OS["OpenShell"]
        GW["Gateway"]
        SB["Sandbox"]
    end
    
    V -->|"API keys stored"| E
    E -->|"synced to K8s Secret"| GW
    S -->|"issues SVID to"| GW
    A -->|"scans image of"| SB
    K -->|"authenticates"| GW
    
    style V fill:#bee3f8,color:#2a4365
    style E fill:#bee3f8,color:#2a4365
    style S fill:#c6f6d5,color:#22543d
    style A fill:#fed7d7,color:#742a2a
    style K fill:#fefcbf,color:#744210
```

## Pillar 1: Vault — Secret Management

Vault stores the DeepSeek API key. The Helm chart's pre-install hook seeds it automatically.

### Verify the Secret Reached Vault

The Helm chart stores the API key in Vault at path `hub/infra/openshell/deepseek`. You can confirm Vault is running:

```bash
oc get pods -n vault -l app.kubernetes.io/name=vault
```

Expected:

```
NAME      READY   STATUS    RESTARTS   AGE
vault-0   1/1     Running   0          ...
```

You can also verify the ESO-synced K8s Secret exists (which proves Vault has the key):

```bash
oc get secret deepseek-api-key -n openshell-system -o jsonpath='{.data.api_key}' | base64 -d | head -c 10
echo "..."
```

This should print the first 10 characters of your DeepSeek key, confirming the Vault → ESO → K8s Secret chain works.

### The Vault Path

| Path | Purpose |
|------|---------|
| `hub/infra/openshell/deepseek` | DeepSeek API key (`api_key` field) |

The Vault auth method is Kubernetes auth mounted at path `hub/`. The Helm chart's setup job authenticates using its ServiceAccount token to write the key.

## Pillar 2: External Secrets Operator — Secret Sync

ESO watches `ExternalSecret` resources and syncs them from Vault to Kubernetes Secrets.

### Verify the ExternalSecret

```bash
oc get externalsecrets -n openshell-system
```

Expected:

```
NAME                  STORETYPE            STORE           REFRESH INTERVAL   STATUS         READY
deepseek-api-key-es   ClusterSecretStore   vault-backend   1h                 SecretSynced   True
```

### Verify the K8s Secret

```bash
oc get secret deepseek-api-key -n openshell-system -o jsonpath='{.data.api_key}' | base64 -d | head -c 10
echo "..."
```

**Why:** This shows that ESO has successfully synced the key from Vault into a Kubernetes Secret. We only print the first 10 characters to confirm it's the right key without exposing the full value.

### How ESO Syncs

```mermaid
graph LR
    ES["ExternalSecret CR<br/>(declares what to sync)"]
    ESO["ESO watches CR<br/>reads from Vault every 1h"]
    V["🔐 Vault KV"]
    KS["K8s Secret<br/>deepseek-api-key"]

    ES --> ESO
    V -->|"ESO reads"| ESO
    ESO -->|"creates/updates"| KS
```

## Pillar 3: SPIFFE/SPIRE — Workload Identity

SPIFFE (Secure Production Identity Framework for Everyone) provides cryptographic identities to workloads. Each OpenShell component gets a **SVID** (SPIFFE Verifiable Identity Document) — an X.509 certificate proving its identity without shared secrets.

### Verify the SPIFFE Registration

```bash
oc get clusterspiffeids | grep openshell
```

Expected:

```
openshell-gateway   ...
```

**Why:** `ClusterSPIFFEID` resources tell SPIRE which SPIFFE IDs to issue to which pods. SPIRE runs in the `zero-trust-workload-identity-manager` namespace.

### What SPIFFE Provides

| Property | Value |
|----------|-------|
| Identity format | `spiffe://cluster.local/ns/openshell-system/sa/openshell-ztm-demo` |
| Certificate type | X.509 SVID |
| Rotation | Automatic (SPIRE agent handles renewal) |
| Trust domain | `cluster.local` |

SPIFFE ensures that the Gateway can prove its identity to other services without static credentials. This is foundational to zero trust: **identity is cryptographic, not network-based**.

## Pillar 4: ACS (StackRox) — Image Security

Red Hat Advanced Cluster Security for Kubernetes (ACS) continuously scans container images for:

- Known CVEs (Common Vulnerabilities and Exposures)
- Sensitive data (embedded secrets, private keys)
- Misconfigurations (running as root, writable filesystem)
- Compliance violations (PCI-DSS, HIPAA, NIST)

### Check the Sandbox Image Scan

Open ACS in your browser:

```
https://central-stackrox.<CLUSTER_DOMAIN>
```

Navigate to: **Vulnerability Management → Images** and search for `openshell`.

Alternatively, from the bastion:

```bash
oc get image-vulnerabilities -n openshell-system 2>/dev/null || echo "Check ACS Console for scan results"
```

### What ACS Enforces

| Check | OpenShell Sandbox |
|-------|-------------------|
| Critical CVEs | Scanned on every pull |
| Run as root | Denied by policy |
| Writable root filesystem | Controlled by Landlock |
| Network exposure | Limited by OpenShell policy |

## Pillar 5: Keycloak — User Identity

Keycloak provides SSO and OIDC for user authentication. The ZTVP deploys a `ztvp` realm with pre-configured clients.

### Verify the Keycloak OIDC Client

```bash
KEYCLOAK_URL=https://keycloak.<CLUSTER_DOMAIN>
ADMIN_PASSWORD=$(oc get secret keycloak-admin-user -n keycloak-system -o jsonpath='{.data.password}' | base64 -d)

TOKEN=$(curl -sk "$KEYCLOAK_URL/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=$ADMIN_PASSWORD" \
  -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -sk "$KEYCLOAK_URL/admin/realms/ztvp/clients" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys, json
clients = json.load(sys.stdin)
for c in clients:
    if 'openshell' in c.get('clientId',''):
        print(f\"Client: {c['clientId']}\")
        print(f\"Enabled: {c['enabled']}\")
        print(f\"Protocol: {c.get('protocol','')}\")
"
```

Expected:

```
Client: openshell-gateway
Enabled: True
Protocol: openid-connect
```

**Why:** This confirms the `openshell-gateway` OIDC client exists in Keycloak's `ztvp` realm. The Gateway is configured to validate JWT tokens issued by this Keycloak realm.

### Authentication Flow

```mermaid
graph TD
    U["👤 User authenticates<br/>with Keycloak (ztvp realm)"]
    JWT["Keycloak issues JWT<br/>audience: openshell-gateway"]
    REQ["User sends gRPC request<br/>to Gateway with JWT attached"]
    VAL["Gateway validates JWT:<br/>✅ Signature valid?<br/>✅ Issuer = keycloak.../realms/ztvp?<br/>✅ Audience = openshell-gateway?<br/>✅ Not expired?"]
    OK["✅ Request authorized"]

    U --> JWT --> REQ --> VAL --> OK
```

## Defense in Depth Summary

Each ZTVP pillar addresses a different attack vector:

| Attack | Mitigation | ZTVP Pillar |
|--------|-----------|-------------|
| Stolen API key | Key in Vault, resolver token in sandbox | Vault + ESO |
| Impersonated workload | Cryptographic SPIFFE identity | SPIRE |
| Vulnerable container image | Continuous CVE scanning | ACS |
| Unauthorized user access | OIDC JWT validation | Keycloak |
| Network exfiltration | Allowlist-only network policy | OpenShell |
| Filesystem escape | Landlock LSM restrictions | OpenShell |
| Credential leakage | Boundary resolver tokens | OpenShell |

## Next Step

You've seen all five pillars of the ZTVP working with OpenShell. Next, you'll learn how to dynamically update the zero-trust policy.

Proceed to: **Dynamic Policy** →
