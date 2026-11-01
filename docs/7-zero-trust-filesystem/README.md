# Filesystem Zero Trust

In this module you will prove that the sandbox has **limited filesystem access**, enforced by **Landlock LSM** — a Linux kernel security module. The agent can only read and write to paths explicitly allowed by the policy.

## What Is Landlock LSM?

**Landlock** is a security feature built into the **Linux kernel** (available since Linux 5.13). LSM stands for **Linux Security Module** — a framework for plugging security policies into the kernel itself.

Why does this matter? Because **kernel enforcement cannot be bypassed from userspace**:

| Enforcement layer | Can a root process bypass it? | Can a container escape bypass it? |
|-------------------|------------------------------|-----------------------------------|
| Application-level checks | ✅ Yes — the app can be patched or tricked | ✅ Yes |
| Container filesystem (overlay) | ✅ Yes — with `CAP_SYS_ADMIN` or escape | ✅ Yes |
| **Landlock LSM (kernel)** | ❌ No — the kernel enforces it | ❌ No — it's below the container boundary |

Even if an attacker gains root inside the sandbox, Landlock still blocks access to restricted paths. The kernel simply returns `EACCES` (Permission denied) for any disallowed operation.

## The Filesystem Policy

Here is the actual policy deployed by the Helm chart:

```yaml
filesystem_policy:
  include_workdir: true
  read_only:
    - /bin
    - /usr
    - /lib
    - /proc
    - /dev/urandom
    - /app
    - /etc
    - /var/log
  read_write:
    - /sandbox
    - /tmp
    - /dev/null
```

**Understanding the two access levels:**

| Path | Access | Why the agent needs it |
|------|--------|----------------------|
| `/bin`, `/usr`, `/lib` | **Read-only** | System binaries and shared libraries — the agent needs to *run* `node`, `npm`, `curl`, etc. but must never *modify* them |
| `/etc` | **Read-only** | Configuration files and TLS certificates — needed for HTTPS connections but must not be writable |
| `/proc` | **Read-only** | Process information (e.g., `/proc/self/status`) — read-only process introspection |
| `/sandbox` | **Read + Write** | The agent's working directory (HOME). Pi writes code here. This is the only place the agent can create files. |
| `/tmp` | **Read + Write** | Temporary files, npm cache, build artifacts |

**Everything else is inaccessible.** This includes:
- `/home` — the traditional home directory doesn't exist in the agent's world
- `/root` — root's home directory
- `/var/run` — runtime data, including Kubernetes service account tokens
- `/` (root) — the agent cannot even *list* what's at the top of the filesystem

## Step 1 — Enter the Sandbox

```bash
openshell sandbox exec pi-agent-demo -- bash
```

## Step 2 — Test ALLOWED: Write to /sandbox

```bash
echo "This is allowed" > /sandbox/test.txt
cat /sandbox/test.txt
```

Expected:

```
This is allowed
```

✅ **Allowed** — `/sandbox` is the agent's working directory and is in the `read_write` list. This is where Pi creates all its files.

## Step 3 — Test ALLOWED: Write to /tmp

```bash
echo "Temp file" > /tmp/test.txt
cat /tmp/test.txt
```

Expected:

```
Temp file
```

✅ **Allowed** — `/tmp` is in the `read_write` list. Build tools (npm, node) use `/tmp` for caches and temporary files. Without `/tmp` access, `npm install` would fail.

## Step 4 — Test ALLOWED: Read System Binaries

```bash
ls /usr/bin/node
```

Expected:

```
/usr/bin/node
```

✅ **Allowed** — `/usr` is in the `read_only` list. The agent can **execute** programs (Node.js, npm, curl) but cannot **modify** them. This prevents a supply-chain attack where the agent replaces a binary with a malicious version.

## Step 5 — Test ALLOWED: Read /etc (read-only)

```bash
cat /etc/hostname
```

Expected: the container hostname is printed (e.g., `default--pi-agent-demo`).

✅ **Allowed** — `/etc` is in the `read_only` list. The agent can read configuration files (like TLS certificates in `/etc/ssl/certs`) which are needed for HTTPS connections to work.

## Step 6 — Test DENIED: Write to /etc

```bash
echo "hacked" > /etc/test 2>&1
```

Expected:

```
bash: /etc/test: Permission denied
```

❌ **Denied** — `/etc` is **read-only**. The agent can read configuration files but cannot modify them. An attacker cannot modify `/etc/resolv.conf` to redirect DNS, or modify `/etc/hosts` to redirect traffic.

> **Why `2>&1`?** This redirects stderr to stdout so we can see the error message. Without it, the error would appear on stderr and might be missed in some terminal configurations.

## Step 7 — Test DENIED: Overwrite a Binary

```bash
cp /dev/null /usr/bin/node 2>&1
```

Expected:

```
cp: cannot create regular file '/usr/bin/node': Permission denied
```

❌ **Denied** — `/usr` is **read-only**. The agent can *run* Node.js but cannot *replace* it with a malicious binary. This is a critical protection against binary planting attacks.

## Step 8 — Test DENIED: List Root Directory

```bash
ls / 2>&1
```

Expected:

```
ls: cannot open directory '/': Permission denied
```

❌ **Denied** — the root directory `/` is **not in any allowlist**. The agent cannot even **enumerate** what exists on the filesystem. It has no way to discover attack surfaces.

> **Why is this important?** An attacker's first step is usually reconnaissance — listing directories to understand the environment. With Landlock, the agent can't even `ls /` to see what's available. It's operating in a minimal, pre-defined world.

## Step 9 — Test DENIED: Create /home/sandbox

```bash
mkdir -p /home/sandbox 2>&1
```

Expected:

```
mkdir: cannot create directory '/home/sandbox': Permission denied
```

❌ **Denied** — `/home` is not in any allowlist. The working directory is `/sandbox`, not `/home/sandbox`. An attacker cannot create hidden directories outside the allowed paths.

## Step 10 — Test: Kubernetes Service Account Token

This is the most security-critical test:

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token 2>&1
```

Expected:

```
cat: /var/run/secrets/kubernetes.io/serviceaccount/token: No such file or directory
```

❌ **Not accessible** — the Kubernetes service account token **does not exist** inside the sandbox. OpenShell does not mount it.

> **Why does this matter?** In a standard Kubernetes pod, a service account token is automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token lets a process authenticate to the Kubernetes API and potentially read Secrets, list pods, or even escalate privileges. By not mounting it, OpenShell ensures the AI agent has **zero access** to the Kubernetes API — it can't discover the cluster, read secrets, or affect other workloads.

## Step 11 — Exit the Sandbox

```bash
exit
```

## Results Summary

| Test | Path | Operation | Expected | Enforcement |
|------|------|-----------|----------|-------------|
| Write /sandbox | `/sandbox/test.txt` | Write | ✅ Success | `read_write` |
| Write /tmp | `/tmp/test.txt` | Write | ✅ Success | `read_write` |
| Read binaries | `/usr/bin/node` | Read | ✅ Success | `read_only` |
| Read /etc | `/etc/hostname` | Read | ✅ Success | `read_only` |
| Write /etc | `/etc/test` | Write | ❌ Permission denied | `read_only` (write blocked) |
| Overwrite binary | `/usr/bin/node` | Write | ❌ Permission denied | `read_only` (write blocked) |
| List root | `/` | Read | ❌ Permission denied | Not in any list |
| Create /home | `/home/sandbox` | Write | ❌ Permission denied | Not in any list |
| K8s SA token | `/var/run/secrets/...` | Read | ❌ No such file | Not mounted |

## Why This Matters

| Attack | Without Landlock | With Landlock |
|--------|-----------------|---------------|
| Read system config + modify it | ✅ Possible | ❌ Read-only or blocked |
| Replace binaries with backdoors | ✅ Possible | ❌ /usr is read-only |
| Access Kubernetes secrets via SA token | ✅ Possible | ❌ Token not mounted |
| Write to cron for persistence | ✅ Possible | ❌ /var not accessible |
| Enumerate filesystem for recon | ✅ Possible | ❌ Can't even `ls /` |
| Escape via writable root fs | ✅ Possible | ❌ Kernel-enforced boundaries |

**The bottom line:** Landlock is enforced by the Linux kernel. No amount of userspace trickery — not even root access inside the container — can bypass it.

## Next Step

Network is locked. Filesystem is locked. But what about **credentials**? The agent needs an API key to call DeepSeek — how is that key protected?

Proceed to: **Credential Security** →
