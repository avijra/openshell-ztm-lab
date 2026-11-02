# Dynamic Policy

In this module you will update the zero-trust policy **live** — adding a new allowed endpoint and verifying the change takes effect immediately, without redeploying the Helm chart, recreating the sandbox, or even restarting a pod.

## Why Dynamic Policy?

In the real world, AI agents need **evolving capabilities**. Today the agent only calls MAAS and npm. Tomorrow it might need to:
- Access a **new package registry** (PyPI for Python packages)
- Call a **new API** (GitHub Actions, Jira, Slack)
- Reach a **new data source** (S3 bucket, database endpoint)

Rather than redeploying the entire chart or recreating the sandbox, OpenShell supports **live policy updates** via a single CLI command. The policy is versioned — every change creates a new version with a hash, providing a full audit trail.

> **Why not just edit the Kubernetes ConfigMap?** While the initial policy is stored as a ConfigMap, OpenShell manages the *effective* policy through its Gateway. Using `openshell policy update` ensures the change is validated, versioned, and propagated to the sandbox instantly. Editing the ConfigMap directly would require a sandbox restart and bypass the version tracking.

## Current Policy State

Recall the current network policy from Module 6:

| Host | Status | Why |
|------|--------|-----|
| `maas-rhdp.apps.maas.redhatworkshops.io` | ✅ Allowed | LLM inference (MAAS) |
| `registry.npmjs.org` | ✅ Allowed | npm packages |
| `github.com` | ✅ Allowed | Code repos |
| `pypi.org` | ❌ **Denied** | Not in the policy |

Let's change that last row.

## Step 1 — Confirm pypi.org Is Blocked

Enter the sandbox and verify that `pypi.org` is currently blocked:

```bash
openshell sandbox exec pi-agent-demo -- bash
```

```bash
curl -v --max-time 5 https://pypi.org/simple/requests/ 2>&1 | tail -3
```

Expected:

```
* Closing connection
curl: (7) Failed to connect to pypi.org port 443 after 2 ms: Couldn't connect to server
```

❌ `pypi.org` is not in the allowlist. The connection is immediately rejected — just like Google and example.com in Module 6.

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

> **What do these fields mean?**
> - **Version**: The policy version number. It increments with each update. `2` means this is the second version (the first was the initial policy, the second includes provider-composed entries).
> - **Hash**: A SHA-256 hash of the policy content. This provides tamper detection — you can verify the policy hasn't been modified unexpectedly.
> - **Status**: `Effective` means this policy is currently active on the sandbox.
> - **Source**: `sandbox` means the policy is managed at the sandbox level (vs. a global gateway policy).

## Step 3 — Add pypi.org to the Policy

Use the `openshell policy update` command to add the endpoint **live**:

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

> **Breaking down the command:**
> - `openshell policy update pi-agent-demo` — update the policy for the `pi-agent-demo` sandbox
> - `--add-endpoint "pypi.org:443:read-write"` — add `pypi.org` on port 443 with `read-write` access (meaning the agent can both send and receive data)
> - `--binary /usr/bin/curl` — only allow the `curl` binary to reach this endpoint (binary-level enforcement)
> - `--wait` — block until the sandbox has **loaded** the new policy version, ensuring the change is active before we test
>
> **No restart needed!** The policy is pushed to the Supervisor, which applies it to the sandbox in real time. The sandbox pod keeps running — no downtime.

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

✅ **pypi.org is now allowed!** The connection succeeds immediately. The sandbox can reach PyPI — a few seconds ago it couldn't.

> **What just happened?** The `openshell policy update` command sent the updated policy to the Gateway, which forwarded it to the Supervisor. The Supervisor updated the sandbox's network rules in real time. When curl tried to connect to `pypi.org`, the Supervisor checked the updated allowlist and allowed the connection.

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

❌ **Blocked again** — the revert took effect immediately. The policy is back to its original state.

> **Full circle:** Blocked → Added → Allowed (HTTP 200) → Removed → Blocked again. All without touching the Helm chart, restarting pods, or editing YAML.

## Policy Management Commands

| Command | What it does | Use case |
|---------|-------------|----------|
| `openshell policy get <sandbox>` | Shows the current effective policy version and hash | "What's the current policy?" |
| `openshell policy get <sandbox> --full -o json` | Shows the full policy payload as JSON | "What endpoints are allowed?" |
| `openshell policy update --add-endpoint ... --wait` | Adds an endpoint to the live sandbox policy | "The agent needs access to a new API" |
| `openshell policy update --remove-endpoint ... --wait` | Removes an endpoint from the live sandbox policy | "Revoke access to an API" |
| `openshell policy set <sandbox> --policy file.yaml` | Replaces the entire policy from a YAML file | "Deploy a completely new policy" |
| `openshell policy list <sandbox>` | Shows the version history | "What changes have been made?" |

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Policy storage** | Gateway-managed, versioned per sandbox |
| **Update mechanism** | `openshell policy update` — live, no restart, no redeployment |
| **Audit trail** | Every update creates a new version with a unique hash |
| **Rollback** | Use `--remove-endpoint` to revert individual changes, or `policy set` to restore a full policy |
| **Downtime** | Zero — the sandbox keeps running throughout |
| **Granularity** | Per-endpoint, per-binary, with access level (read-only, read-write) |

> **In a real-world workflow**, policy changes would go through a review process (e.g., a PR that modifies the policy YAML, reviewed by a security team, applied via CI/CD). OpenShell's versioning and hashing support this by providing tamper-evident audit trails.

## Next Step

You've completed all the demo scenarios! The only thing left is cleanup.

Proceed to: **Cleanup** →
