# Prerequisites

Before starting this lab, let's verify you have everything you need. This module takes about 3 minutes.

## What Your Instructor Provides

Your instructor will give you four pieces of information. Enter them in the **configuration bar** at the top of this page:

| Item | Example | What it is |
|------|---------|-----------|
| **Bastion host** | `bastion.wjw4q.sandbox3490.opentlc.com` | A jump server with network access to the cluster. You'll SSH here to run all commands. |
| **SSH password** | provided verbally | The password for the `lab-user` user on the bastion |
| **Cluster domain** | `apps.ocp.<GUID>.sandbox<ID>.opentlc.com` | The domain suffix for all cluster routes (URLs). Every service is at `https://something.<this-domain>` |
| **MAAS API key** | `sk-...` | Your personal key for the Red Hat MAAS (Model-as-a-Service) LLM API. Pi uses this to generate code via gpt-oss-120b. |

> **What is a bastion host?** A bastion (or jump box) is a hardened server that sits between you and the internal network. You SSH into it, and from there you can access cluster resources. It already has `oc` (the OpenShift CLI) and `openshell` (the OpenShell CLI) installed and authenticated. You never need to install anything on your laptop.

## OpenShift Cluster with ZTVP

The cluster is pre-deployed with:

- **OpenShift on AWS** — OCP 4.x, running on worker nodes in AWS
- **Zero Trust Validated Pattern** — fully deployed, all components Healthy

The ZTVP provides these components, which are **already running**. You do not need to deploy or configure any of them:

| Component | What it does | Status | How to verify |
|-----------|-------------|--------|--------------|
| **HashiCorp Vault** | Stores secrets (API keys, certificates) in encrypted storage with access control | ✅ Deployed | `https://vault-vault.<CLUSTER_DOMAIN>` |
| **External Secrets Operator (ESO)** | Automatically syncs secrets from Vault into Kubernetes Secrets | ✅ Deployed | Namespace `external-secrets` |
| **Keycloak** | Provides single sign-on (SSO) and OIDC authentication for users and services | ✅ Deployed | `https://keycloak.<CLUSTER_DOMAIN>` |
| **SPIFFE/SPIRE** | Issues cryptographic workload identities — like passports for pods | ✅ Deployed | Namespace `zero-trust-workload-identity-manager` |
| **ACS (StackRox)** | Scans container images for vulnerabilities and enforces security policies | ✅ Deployed | `https://central-stackrox.<CLUSTER_DOMAIN>` |

> **Why are these already running?** The ZTVP is designed to be deployed once per cluster as a foundation. Applications (like OpenShell) then plug into it. This mirrors real-world production: the security platform team deploys ZTVP, and application teams consume its services.

## SSH Client

You need an SSH client to connect to the bastion host:
- **macOS / Linux**: Built-in — just open Terminal and type `ssh`
- **Windows**: Use PowerShell, Windows Subsystem for Linux (WSL), or [PuTTY](https://www.putty.org/)

## Fill In Your Config

Before proceeding, scroll to the **top of this page** and fill in the form with:

1. **Bastion Host** — the SSH hostname
2. **SSH Password** — the SSH password
3. **Cluster Domain** — the `apps.ocp.*` domain
4. **MAAS API Key** — your personal key (starts with `sk-`)

Click **Save**. All commands in this lab will auto-populate with your values.

## Verify: SSH to the Bastion

Open a terminal and confirm SSH access:

```bash
ssh lab-user@<BASTION_HOST>
```

When prompted for the password, enter: `<SSH_PASSWORD>`

You should see a shell prompt like:

```
[lab-user@bastion ~]$
```

> **What does this mean?** You're now logged into the bastion server as the `lab-user` user. This machine has direct network access to the OpenShift cluster, so all `oc` and `openshell` commands will work from here.

> **Troubleshooting:** If you cannot connect, check with your instructor that (1) the bastion is running, (2) your network allows outbound SSH on port 22, and (3) you have the correct hostname.

## Verify: OpenShift CLI on Bastion

The bastion already has `oc` (OpenShift CLI) installed and authenticated as a cluster admin:

```bash
oc whoami
```

Expected output:

```
system:admin
```

> **What is `oc`?** It's the OpenShift command-line tool — like `kubectl` for Kubernetes, but with extra OpenShift features. `system:admin` means you have full administrative access to the cluster via the kubeconfig. On some clusters you may see `cluster-admin` instead — both indicate full admin privileges.

If you see a login error, ask your instructor for the **cluster-admin password** (this is different from the SSH password) and run:

```bash
oc login https://api.<CLUSTER_DOMAIN_BASE>:6443 --username cluster-admin --password <CLUSTER_ADMIN_PASSWORD>
```

> **Note:** The cluster-admin password is separate from the SSH password. The bastion is pre-configured with a valid `oc` session, so this step is rarely needed.

## Verify: Helm CLI

The lab uses Helm to install the OpenShell chart. Check if Helm is installed:

```bash
helm version --short
```

If `helm` is not found, install it:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Verify: OpenShell CLI

Check if the OpenShell CLI is installed:

```bash
openshell --version
```

Expected output: `openshell 0.1.6` (or later).

If `openshell` is not found, install it:

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

> **Note:** The install script places the binary in `~/.local/bin`. You may need to add it to your PATH: `export PATH=$HOME/.local/bin:$PATH`

## Ready?

Once you have:

- [x] SSH access to the bastion
- [x] `oc whoami` returning `cluster-admin`
- [x] Your MAAS API key ready

You're all set. Let's deploy OpenShell.

Proceed to the next module: **Deploy from Catalog** →
