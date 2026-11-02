# ZTVP Integration

In this module you will explore how the **Zero Trust Validated Pattern** components work together with OpenShell to form a **defense-in-depth** security architecture. Each component addresses a different attack vector — together, they create multiple layers that an attacker would need to penetrate simultaneously.

> **What is defense-in-depth?** It's a security strategy where multiple independent layers protect the system. If one layer fails, the others still hold. Think of a castle with a moat, a wall, a gate, and armed guards — compromising any one layer doesn't give you full access.

## The Five Pillars

The ZTVP provides five security capabilities. OpenShell plugs into all of them:

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

**Reading the diagram:** Each ZTVP component connects to OpenShell in a different way:
- **Vault + ESO** → stores and syncs the MAAS API key to the Gateway
- **SPIFFE/SPIRE** → gives the Gateway a cryptographic identity
- **ACS** → scans the sandbox container image for vulnerabilities
- **Keycloak** → authenticates users who access the Gateway

## Pillar 1: Vault — Secret Management

> **What is Vault?** [HashiCorp Vault](https://www.vaultproject.io/) is a secret management system. It stores sensitive data (API keys, passwords, certificates) in encrypted storage with fine-grained access control. Instead of putting secrets in Kubernetes Secrets (which are base64-encoded, not encrypted), you put them in Vault where they're encrypted at rest, access-logged, and rotatable.

The Helm chart's pre-install hook stores your MAAS API key in Vault automatically.

### Verify Vault Is Running

```bash
oc get pods -n vault -l app.kubernetes.io/name=vault
```

Expected:

```
NAME      READY   STATUS    RESTARTS   AGE
vault-0   1/1     Running   0          ...
```

### Verify the Secret Chain Works

You can verify the Vault → ESO → K8s Secret chain by checking the synced K8s Secret:

```bash
oc get secret maas-api-key -n openshell-system -o jsonpath='{.data.api_key}' | base64 -d | head -c 10
echo "..."
```

This should print the first 10 characters of your MAAS key, confirming the entire chain works: Vault has the key → ESO synced it → it's available as a K8s Secret.

> **Why only 10 characters?** We don't want to expose the full key in the terminal. The first 10 chars are enough to confirm it's the right key.

### The Vault Path

| Path | Purpose |
|------|---------|
| `hub/infra/openshell/maas` | MAAS API key (`api_key` field) |

> **What does `hub/` mean?** The ZTVP configures Vault with a Kubernetes auth method mounted at path `hub/`. This means pods in the cluster can authenticate to Vault using their ServiceAccount tokens — no static credentials needed. The Helm chart's setup job uses this auth method to write the key.

## Pillar 2: External Secrets Operator — Secret Sync

> **What is ESO?** The [External Secrets Operator](https://external-secrets.io/) watches `ExternalSecret` custom resources and syncs them from external sources (Vault, AWS Secrets Manager, etc.) into Kubernetes Secrets. It's the bridge between "secrets stored in Vault" and "secrets available to pods."

### Verify the ExternalSecret

```bash
oc get externalsecrets -n openshell-system
```

Expected:

```
NAME              STORETYPE            STORE           REFRESH INTERVAL   STATUS         READY   LAST SYNC
maas-api-key-es   ClusterSecretStore   vault-backend   1h                 SecretSynced   True    ...
```

> **Reading this output:**
> - **NAME**: `maas-api-key-es` — the ExternalSecret resource created by the Helm chart
> - **STORETYPE**: `ClusterSecretStore` — it uses a cluster-wide secret store (not namespace-scoped)
> - **STORE**: `vault-backend` — the store is backed by Vault
> - **REFRESH INTERVAL**: `1h` — ESO checks Vault for updates every hour
> - **STATUS**: `SecretSynced` — the K8s Secret was successfully created/updated from Vault
> - **READY**: `True` — everything is healthy

### How ESO Syncs

```mermaid
graph LR
    ES["ExternalSecret CR<br/>(declares what to sync)"]
    ESO["ESO watches CR<br/>reads from Vault every 1h"]
    V["🔐 Vault KV"]
    KS["K8s Secret<br/>maas-api-key"]

    ES --> ESO
    V -->|"ESO reads"| ESO
    ESO -->|"creates/updates"| KS
```

> **Why is this valuable?** If you rotate the API key in Vault, ESO will automatically sync the new value to the K8s Secret within 1 hour. The Gateway picks up the new secret, and Pi's next API call uses the new key — no redeployment needed.

## Pillar 3: SPIFFE/SPIRE — Workload Identity

> **What is SPIFFE?** [SPIFFE](https://spiffe.io/) (Secure Production Identity Framework for Everyone) is a standard for issuing cryptographic identities to workloads. Instead of saying "this pod is in namespace X, therefore it's trusted," SPIFFE says "this pod has a cryptographically signed certificate (called an SVID) that proves exactly who it is."

> **What is SPIRE?** SPIRE is the reference implementation of SPIFFE. It runs as a DaemonSet (agent on every node) and issues SVIDs (SPIFFE Verifiable Identity Documents) — X.509 certificates that prove a workload's identity.

### Verify the SPIFFE Registration

```bash
oc get clusterspiffeids | grep openshell
```

Expected:

```
openshell-gateway   ...
```

> **What does this mean?** A `ClusterSPIFFEID` resource tells SPIRE: "Issue a SPIFFE identity to pods matching this selector." The `openshell-gateway` entry means SPIRE issues an SVID to the Gateway pod.

### What SPIFFE Provides

| Property | Value |
|----------|-------|
| Identity format | `spiffe://cluster.local/ns/openshell-system/sa/openshell-ztm-demo` |
| Certificate type | X.509 SVID (a standard TLS certificate with a SPIFFE ID) |
| Rotation | Automatic — SPIRE rotates certificates before they expire |
| Trust domain | `cluster.local` |

> **Why does this matter?** In zero trust, you can't assume "same network = trusted." SPIFFE gives each workload a cryptographic passport. The Gateway can prove to Vault, to Keycloak, or to any other service that it is who it claims to be — without shared secrets or static credentials.

## Pillar 4: ACS (StackRox) — Image Security

> **What is ACS?** [Red Hat Advanced Cluster Security](https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes) for Kubernetes (formerly StackRox) continuously scans container images and running deployments for security issues.

ACS scans for:
- **Known CVEs** (Common Vulnerabilities and Exposures) — known bugs in libraries
- **Sensitive data** — embedded secrets, private keys, passwords in the image
- **Misconfigurations** — running as root, writable root filesystem, privilege escalation
- **Compliance violations** — PCI-DSS, HIPAA, NIST standards

### Verify ACS Is Running

```bash
oc get pods -n stackrox -l app=central --no-headers
```

Expected:

```
central-xxxxxxxxx-xxxxx   1/1   Running   0   ...
```

You can also open the ACS Console in your browser:

```
https://central-stackrox.<CLUSTER_DOMAIN>
```

> **Note:** This lab does not configure image-specific scan policies for OpenShell. ACS is deployed by ZTVP and continuously monitors the cluster — scanning deployed images for CVEs, detecting misconfigurations, and enforcing security policies. In a production deployment, you would configure ACS admission control policies to block images with critical CVEs from being deployed.

> **Why does ACS matter in a zero-trust architecture?** Even with network and filesystem enforcement, a compromised container image (e.g., one with a backdoored dependency) could attack from *inside* the sandbox. ACS provides the image-level assurance: it verifies that the sandbox image itself is safe before and after deployment.

## Pillar 5: Keycloak — User Identity

> **What is Keycloak?** [Keycloak](https://www.keycloak.org/) is an open-source identity and access management (IAM) system. It provides SSO (Single Sign-On), OIDC (OpenID Connect), and user federation. In this lab, Keycloak authenticates users who access the OpenShell Gateway.

The ZTVP deploys a `ztvp` realm in Keycloak with pre-configured clients.

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

> **What does this prove?** The `openshell-gateway` OIDC client exists in Keycloak's `ztvp` realm. The Gateway is configured to validate JWT tokens issued by this Keycloak realm. In production, users would log in through Keycloak to get a JWT, and the Gateway would verify it before allowing access.

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

> **What is a JWT?** A JSON Web Token is a signed, base64-encoded JSON document. Keycloak signs it with its private key. The Gateway verifies the signature with Keycloak's public key. If the signature is valid, the token hasn't been tampered with. JWTs contain claims like "who is this user," "what are they allowed to do," and "when does this token expire."

## Defense in Depth Summary

Each pillar addresses a different attack vector. An attacker would need to defeat **all of them simultaneously** to fully compromise the system:

| Attack Vector | Mitigation | ZTVP Pillar |
|--------|-----------|-------------|
| Stolen MAAS API key | Key in Vault, resolver token in sandbox | **Vault + ESO** |
| Impersonated workload | Cryptographic SPIFFE identity (X.509 SVID) | **SPIRE** |
| Vulnerable container image | Continuous CVE scanning + admission control | **ACS** |
| Unauthorized user access | OIDC JWT validation via Keycloak | **Keycloak** |
| Network exfiltration | Allowlist-only network policy with binary enforcement | **OpenShell** |
| Filesystem escape | Kernel-enforced Landlock LSM restrictions | **OpenShell** |
| Credential leakage | Boundary resolver tokens (never expose real keys) | **OpenShell** |

> **This is zero trust in action.** No single component is sufficient alone. Vault without network policy means keys are safe but data can be exfiltrated. Network policy without Landlock means you can't reach the internet but can modify local binaries. Each layer compensates for the others' blind spots.

## Next Step

You've seen all five pillars of the ZTVP working with OpenShell. Next, you'll explore how **user authentication** works — from Keycloak OIDC to JWT tokens to what the Gateway validates.

Proceed to: **User Authentication** →
