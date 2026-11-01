# Dynamic Policy

In this module you will update the zero-trust policy live — adding a new allowed endpoint and verifying the change takes effect after a sandbox restart.

## Why Dynamic Policy?

In the real world, AI agents need evolving capabilities. A new API, a new package registry, a new data source. Rather than redeploying the entire chart, OpenShell supports updating the policy ConfigMap and reloading it.

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

## Step 2 — Update the Policy ConfigMap

From the bastion, edit the policy ConfigMap to add `pypi.org`:

```bash
oc get configmap pi-sandbox-policy -n openshell-system -o yaml > /tmp/policy-backup.yaml
```

**Why:** Always back up the current policy before modifying it. This lets you revert if something goes wrong.

Now patch the ConfigMap to add pypi.org:

```bash
oc get configmap pi-sandbox-policy -n openshell-system -o json | \
  python3 -c "
import sys, json, yaml

cm = json.load(sys.stdin)
policy = yaml.safe_load(cm['data']['policy.yaml'])

# Add pypi.org to network_policies (dict of named policies)
np = policy.get('network_policies', {})
if 'pypi_registry' not in np:
    np['pypi_registry'] = {
        'name': 'PyPI Registry',
        'endpoints': [{'host': 'pypi.org', 'port': 443}],
        'binaries': [{'path': '/usr/bin/pip'}, {'path': '/usr/bin/python3'}, {'path': '/usr/bin/curl'}]
    }
    policy['network_policies'] = np
    cm['data']['policy.yaml'] = yaml.dump(policy, default_flow_style=False)
    print(json.dumps(cm))
else:
    print(json.dumps(cm))
" | oc apply -f -
```

**Why:** This adds `pypi.org:443` to the `network_policies` dict in the policy ConfigMap. The script targets the `policy.yaml` key specifically, adds a new `pypi_registry` entry with allowed binaries, and applies the update.

## Step 3 — Trigger Policy Reload

The policy ConfigMap change does not automatically reload. Stop and restart the sandbox to pick up the new policy:

```bash
openshell sandbox stop pi-agent-demo
openshell sandbox start pi-agent-demo
```

**Why:** OpenShell reads the policy at sandbox startup. To apply a ConfigMap change, the sandbox must be stopped and started. This is a deliberate design choice — policy changes are auditable events, not silent background updates.

Wait for the sandbox to be ready again:

```bash
openshell sandbox list
```

Wait until `PHASE` shows `Ready`.

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

✅ `pypi.org` is now allowed! The agent can install Python packages.

## Step 5 — Test Python Package Installation

```bash
pip install requests 2>&1 | tail -3
```

Expected:

```
Successfully installed requests-x.x.x ...
```

```bash
exit
```

## Policy Change Flow

The update is a three-step process: **edit → stop/start → verify**.

| Step | Command | What Happens |
|------|---------|-------------|
| **Edit** | `oc edit configmap pi-sandbox-policy` | Add `pypi.org` to the network allowlist |
| **Stop/Start** | `openshell sandbox stop` then `start` | Gateway stops old sandbox, reads new ConfigMap, starts fresh sandbox with updated policy |
| **Verify** | `curl https://pypi.org` inside sandbox | Connection succeeds — pypi.org is now reachable |

## Step 6 — Revert the Policy (Optional)

To remove the pypi.org entry and restore the original policy:

```bash
oc apply -f /tmp/policy-backup.yaml
openshell sandbox stop pi-agent-demo
openshell sandbox start pi-agent-demo
```

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| Policy storage | Kubernetes ConfigMap — GitOps-friendly |
| Policy format | YAML — human-readable, auditable |
| Update mechanism | Edit ConfigMap → stop/start sandbox |
| Audit trail | ConfigMap changes are logged by the K8s API audit log |
| No downtime | Sandbox restarts in seconds |

## Next Step

You've completed all the demo scenarios. Next, clean up the environment.

Proceed to: **Cleanup** →
