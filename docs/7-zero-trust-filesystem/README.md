# Filesystem Zero Trust

In this module you will prove that the sandbox has **limited filesystem access**, enforced by Linux Landlock LSM. The agent can only read and write to paths explicitly allowed by the policy.

## The Filesystem Policy

The actual policy from the cluster:

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

| Path | Access | Why |
|------|--------|-----|
| `/bin`, `/usr`, `/lib` | Read-only | System binaries and libraries |
| `/etc` | Read-only | Configuration files, TLS certs |
| `/proc` | Read-only | Process information |
| `/sandbox` | Read + Write | Agent's working directory (HOME) |
| `/tmp` | Read + Write | Temporary files, npm cache |

Everything outside these lists — including `/home`, `/root`, `/var/run`, and the root directory `/` itself — is **inaccessible**.

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

✅ **Allowed** — `/sandbox` is the agent's working directory and is in the `read_write` list.

## Step 3 — Test ALLOWED: Write to /tmp

```bash
echo "Temp file" > /tmp/test.txt
cat /tmp/test.txt
```

Expected:

```
Temp file
```

✅ **Allowed** — `/tmp` is in the `read_write` list. npm and build tools use `/tmp` for caches.

## Step 4 — Test ALLOWED: Read System Binaries

```bash
ls /usr/bin/node
```

Expected:

```
/usr/bin/node
```

✅ **Allowed** — `/usr` is in the `read_only` list. The agent can execute programs but cannot modify them.

## Step 5 — Test ALLOWED: Read /etc (read-only)

```bash
cat /etc/hostname
```

Expected: the container hostname is printed.

✅ **Allowed** — `/etc` is in the `read_only` list. The agent can read configuration files and TLS certificates.

## Step 6 — Test DENIED: Write to /etc

```bash
echo "hacked" > /etc/test 2>&1
```

Expected:

```
bash: /etc/test: Permission denied
```

❌ **Denied** — `/etc` is read-only. The agent can read but not modify system configuration.

## Step 7 — Test DENIED: Overwrite a Binary

```bash
cp /dev/null /usr/bin/node 2>&1
```

Expected:

```
cp: cannot create regular file '/usr/bin/node': Permission denied
```

❌ **Denied** — `/usr` is read-only. The agent can run Node.js but cannot replace it with a malicious binary.

## Step 8 — Test DENIED: List Root Directory

```bash
ls / 2>&1
```

Expected:

```
ls: cannot open directory '/': Permission denied
```

❌ **Denied** — the root directory `/` is not in any allowlist. The agent cannot even enumerate what exists on the filesystem.

## Step 9 — Test DENIED: Create /home/sandbox

```bash
mkdir -p /home/sandbox 2>&1
```

Expected:

```
mkdir: cannot create directory '/home/sandbox': Permission denied
```

❌ **Denied** — `/home` is not in any allowlist. The working directory is `/sandbox`, not `/home/sandbox`.

## Step 10 — Test: Kubernetes Service Account Token

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token 2>&1
```

Expected:

```
cat: /var/run/secrets/kubernetes.io/serviceaccount/token: No such file or directory
```

❌ **Not accessible** — the token path does not exist inside the sandbox. OpenShell does not mount Kubernetes service account tokens into sandbox containers, preventing the agent from authenticating to the Kubernetes API.

## Step 11 — Exit the Sandbox

```bash
exit
```

## Results Summary

| Test | Path | Operation | Expected | Policy |
|------|------|-----------|----------|--------|
| Write /sandbox | `/sandbox/test.txt` | Write | Success | ✅ read_write |
| Write /tmp | `/tmp/test.txt` | Write | Success | ✅ read_write |
| Read binaries | `/usr/bin/node` | Read | Success | ✅ read_only |
| Read /etc | `/etc/hostname` | Read | Success | ✅ read_only |
| Write /etc | `/etc/test` | Write | Permission denied | ❌ read_only |
| Overwrite binary | `/usr/bin/node` | Write | Permission denied | ❌ read_only |
| List root | `/` | Read | Permission denied | ❌ Not listed |
| Create /home | `/home/sandbox` | Write | Permission denied | ❌ Not listed |
| K8s SA token | `/var/run/secrets/...` | Read | No such file | ❌ Not mounted |

## Why This Matters

Without filesystem restrictions, an AI agent could:

- **Read system configuration** and modify it
- **Modify system binaries** to install backdoors
- **Access Kubernetes secrets** via service account tokens
- **Write to cron directories** to establish persistence
- **Enumerate the filesystem** to discover attack surfaces

With Landlock LSM:

- The agent can only work in `/sandbox` and `/tmp`
- System files are read-only or completely inaccessible
- Kubernetes secrets are not mounted
- The kernel enforces this — no userspace bypass possible

## Next Step

Network is locked. Filesystem is locked. But what about credentials? Next, you'll see how API keys are protected.

Proceed to: **Credential Security** →
