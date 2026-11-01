# Dynamic Policy

In this module you will update the zero-trust policy live — adding a new allowed endpoint and verifying the change takes effect immediately, without redeploying the chart or recreating the sandbox.

## Why Dynamic Policy?

In the real world, AI agents need evolving capabilities. A new API, a new package registry, a new data source. Rather than redeploying the entire chart, OpenShell supports updating the policy on a live sandbox with a single CLI command.

## Current Policy State

Recall the current network policy:

| Host | Status |
|------|--------|
| `api.deepseek.com` | ✅ Allowed |
| `registry.npmjs.org` | ✅ Allowed |
| `github.com` | ✅ Allowed |
| `pypi.org` | ❌ Denied |

## Step 1 — Confirm pypi.org Is Blocked

Enter the sandbox and verify:

```bash
openshell sandbox exec pi-agent-demo -- bash
```

```bash
curl -v --max-time 5 https://pypi.org/simple/requests/ 2>&1 | tail -3
```

Expected:

```
* Immediate connect fail for x.x.x.x: Permission denied
* Failed to connect to pypi.org port 443 after 2 ms: Couldn't connect to server
curl: (7) Failed to connect to pypi.org port 443 after 2 ms: Couldn't connect to server
```

❌ `pypi.org` is not in the allowlist. The connection is immediately rejected.

```bash
exit
```

## Step 2 — View the Current Policy

Check the current effective policy version:

```bash
openshell policy get pi-agent-demo
```

Expected:

```
Version:      2
Hash:         ...
Status:       Effective
Source:       sandbox
```

## Step 3 — Add pypi.org to the Policy

Use the `openshell policy update` command to add the endpoint live:

```bash
openshell policy update pi-agent-demo \
  --add-endpoint "pypi.org:443:read-write" \
  --binary /usr/bin/curl \
  --wait
```

Expected:

```
✓ Policy version 3 submitted (hash: ...)
✓ Policy version 3 loaded (active version: 3)
```

**Why:** `--add-endpoint` adds `pypi.org:443` to the network allowlist with `read-write` access. `--binary /usr/bin/curl` restricts which binaries can reach it. `--wait` blocks until the sandbox has loaded the new policy version. No restart is needed — the policy is applied live.

## Step 4 — Verify pypi.org Is Now Allowed

```bash
openshell sandbox exec pi-agent-demo -- bash
```

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 https://pypi.org/simple/requests/
```

Expected:

```
HTTP 200
```

✅ `pypi.org` is now allowed! The connection succeeds immediately.

```bash
exit
```

## Step 5 — Revert the Policy (Optional)

To remove the pypi.org entry and restore the original policy:

```bash
openshell policy update pi-agent-demo \
  --remove-endpoint pypi.org:443 \
  --wait
```

Expected:

```
✓ Policy version 4 submitted (hash: ...)
✓ Policy version 4 loaded (active version: 4)
```

Verify pypi.org is blocked again:

```bash
openshell sandbox exec pi-agent-demo -- bash -c "curl -v --max-time 5 https://pypi.org 2>&1 | tail -3"
```

Expected:

```
curl: (7) Failed to connect to pypi.org port 443 ...
```

❌ Blocked again — the revert took effect immediately.

## Policy Change Flow

| Step | Command | What Happens |
|------|---------|-------------|
| **View** | `openshell policy get pi-agent-demo` | Shows current policy version and hash |
| **Add** | `openshell policy update --add-endpoint ... --wait` | Adds endpoint to the live sandbox policy, waits for it to load |
| **Remove** | `openshell policy update --remove-endpoint ... --wait` | Removes endpoint from the live sandbox policy |
| **Full replace** | `openshell policy set pi-agent-demo --policy policy.yaml` | Replaces the entire policy from a file |

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| Policy storage | Gateway-managed, versioned per sandbox |
| Update mechanism | `openshell policy update` — live, no restart |
| Audit trail | Every update creates a new version with a hash |
| Rollback | Use `--remove-endpoint` to revert individual changes |
| No downtime | Policy loads in seconds without stopping the sandbox |

## Next Step

You've completed all the demo scenarios. Next, clean up the environment.

Proceed to: **Cleanup** →
