# Client Setup

In this module you will install the OpenShell CLI on the bastion and configure it to talk to the Gateway.

## Why the Bastion?

The OpenShell Gateway is exposed via an OpenShift Route inside the cluster. The bastion host has direct network access to the cluster and already has `oc` authenticated. This makes it the ideal client.

## Step 1 — SSH to the Bastion

```bash
ssh rosa@<BASTION_HOST>
```

## Step 2 — Check the OpenShell CLI

The OpenShell CLI should already be installed by the lab provisioning. Verify:

```bash
openshell --version
```

Expected output:

```
openshell 0.1.6
```

> If `openshell` is not found, download it:
> ```bash
> curl -fsSL https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
> ```

## Step 3 — Configure the Gateway Endpoint

The CLI needs to know the Gateway's address. Get the Route hostname and set it with the `https://` prefix:

```bash
export OPENSHELL_GATEWAY_ENDPOINT=https://$(oc get route openshell-ztm-demo -n openshell-system -o jsonpath='{.spec.host}')
echo "Gateway: $OPENSHELL_GATEWAY_ENDPOINT"
```

Expected output:

```
Gateway: https://openshell-ztm-demo-openshell-system.apps.rosa.rosa-wjw4q.8pwg.p3.openshiftapps.com
```

**Why:** `OPENSHELL_GATEWAY_ENDPOINT` tells the CLI where to send gRPC requests. The `https://` prefix is required — the CLI rejects bare hostnames.

Since the OpenShift Route uses a cluster-internal CA certificate, set the insecure flag:

```bash
export OPENSHELL_GATEWAY_INSECURE=true
```

**Why:** The Gateway uses TLS (via the OpenShift Route), but the certificate is issued by the cluster's internal CA. Setting `OPENSHELL_GATEWAY_INSECURE=true` tells the CLI to skip certificate validation. In production, you would configure a trusted certificate instead.

## Step 4 — Verify Connectivity

List the sandboxes registered with the Gateway:

```bash
openshell sandbox list
```

Expected output:

```
NAME           CREATED              PHASE
pi-agent-demo  2026-xx-xx xx:xx:xx  Ready
```

**What this tells you:**

| Field | Meaning |
|-------|---------|
| `NAME` | The sandbox identifier — the container running Pi |
| `CREATED` | When the sandbox was created |
| `PHASE` | `Ready` means the pod is running and the supervisor is connected |

## Step 5 — Make the Config Persistent

Add the environment variables to your shell profile so they survive new sessions:

```bash
cat >> ~/.bashrc <<'EOF'

# OpenShell CLI configuration
export OPENSHELL_GATEWAY_ENDPOINT=https://$(oc get route openshell-ztm-demo -n openshell-system -o jsonpath='{.spec.host}')
export OPENSHELL_GATEWAY_INSECURE=true
EOF

source ~/.bashrc
```

**Why:** Without this, you would need to re-export the variables every time you open a new SSH session.

## Connection Architecture

Here is how the CLI communicates with the sandbox:

```mermaid
graph TD
    CLI["💻 OpenShell CLI<br/>(on Bastion)"]
    RT["🔗 OpenShift Route<br/>TLS termination"]
    GW["⚡ Gateway Pod<br/>authenticates, routes"]
    SV["👁️ Supervisor<br/>enforces boundary"]
    SB["📦 Sandbox Pod<br/>runs Pi Agent"]

    CLI -->|"gRPC over HTTPS"| RT
    RT -->|"plaintext inside cluster"| GW
    GW -->|"forward exec"| SV
    SV -->|"run command"| SB
    SB -->|"result returns<br/>same path back"| CLI
```

## Quick Reference

Set these two environment variables on any machine with network access to the cluster:

```bash
export OPENSHELL_GATEWAY_ENDPOINT=https://openshell-ztm-demo-openshell-system.<CLUSTER_DOMAIN>
export OPENSHELL_GATEWAY_INSECURE=true
```

Then you can use any `openshell` command.

## Next Step

The CLI is configured and connected to the Gateway. Next, you'll run the Pi coding agent inside the sandbox.

Proceed to: **AI Agent Coding** →
