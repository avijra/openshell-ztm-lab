# Prerequisites

Before starting this lab, verify you have the following.

## What Your Instructor Provides

| Item | Example |
|------|---------|
| Bastion host | `bastion.wjw4q.sandbox3490.opentlc.com` |
| SSH password | provided verbally |
| ROSA cluster domain | `apps.rosa.rosa-wjw4q.8pwg.p3.openshiftapps.com` |
| DeepSeek API key | `sk-...` (get yours at [platform.deepseek.com](https://platform.deepseek.com)) |

## ROSA Cluster with ZTVP

The cluster is pre-deployed with:

- **ROSA** (Red Hat OpenShift on AWS) — OCP 4.22.x
- **Zero Trust Validated Pattern** — fully deployed and Healthy

The ZTVP provides these components, which are already running:

| Component | Status | Verification |
|-----------|--------|--------------|
| HashiCorp Vault | ✅ Deployed | `https://vault.<CLUSTER_DOMAIN>` |
| External Secrets Operator | ✅ Deployed | namespace `external-secrets-operator` |
| Keycloak | ✅ Deployed | `https://keycloak.<CLUSTER_DOMAIN>` |
| SPIFFE/SPIRE | ✅ Deployed | namespace `spire-server` |
| ACS (StackRox) | ✅ Deployed | `https://central-stackrox.<CLUSTER_DOMAIN>` |

> You do not need to deploy or configure any ZTVP components. The pattern is already running.

## SSH Client

You need an SSH client to connect to the bastion host. macOS and Linux have one built-in. On Windows, use PowerShell, WSL, or PuTTY.

## Fill In Your Config

Before proceeding, scroll to the **top of this page** and fill in the form with:

1. **Bastion Host** — the SSH hostname
2. **SSH Password** — the SSH password
3. **Cluster Domain** — the `apps.rosa.rosa-*` domain
4. **DeepSeek API Key** — your personal key (starts with `sk-`)

Click **Save**. All commands in this lab will auto-populate with your values.

## Verify: SSH to the Bastion

Open a terminal and confirm SSH access:

```bash
ssh rosa@<BASTION_HOST>
```

When prompted for the password, enter: `<SSH_PASSWORD>`

You should see a shell prompt like:

```
[rosa@bastion ~]$
```

> If you cannot connect, check with your instructor that the bastion is running and your network allows SSH (port 22).

## Verify: OpenShift CLI on Bastion

The bastion already has `oc` (OpenShift CLI) installed and authenticated:

```bash
oc whoami
```

Expected output:

```
cluster-admin
```

If you see a login error, run:

```bash
oc login https://api.rosa-wjw4q.8pwg.p3.openshiftapps.com:6443 --username cluster-admin --password <SSH_PASSWORD>
```

## Ready?

Once you have:

- [x] SSH access to the bastion
- [x] `oc whoami` returning a valid user
- [x] Your DeepSeek API key ready

Proceed to the next module: **Deploy from Catalog** →
