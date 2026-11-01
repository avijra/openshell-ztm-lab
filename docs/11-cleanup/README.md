# Cleanup

This module walks through removing the OpenShell demo from the cluster. The ZTVP components remain untouched — only the OpenShell resources are removed.

## Step 1 — Uninstall the Helm Release

From the bastion:

```bash
helm uninstall openshell -n openshell-system
```

**Why:** `helm uninstall` removes all resources created by the chart: Deployments, StatefulSets, Services, Routes, ConfigMaps, and Secrets. Helm tracks these resources via the release metadata.

Expected:

```
release "openshell" uninstalled
```

## Step 2 — Verify Resources Are Gone

```bash
oc get all -n openshell-system
```

Expected: only the namespace remains (or a few lingering pods in `Terminating` state that will clear in seconds).

```
No resources found in openshell-system namespace.
```

## Step 3 — Clean Up the Vault Secret (Optional)

The API key stored in Vault persists after Helm uninstall. To remove it:

```bash
export VAULT_ADDR=https://vault.<CLUSTER_DOMAIN>
export VAULT_SKIP_VERIFY=true

VAULT_TOKEN=$(oc get secret vault-init -n vault -o jsonpath='{.data.root_token}' | base64 -d)
export VAULT_TOKEN

vault kv delete secret/openshell/deepseek-api-key
```

**Why:** Vault secrets are external to Kubernetes. `helm uninstall` does not touch them. If you plan to reinstall, you can leave the key in Vault — the chart's pre-install hook will overwrite it with the new value.

## Step 4 — Remove the Namespace (Optional)

```bash
oc delete namespace openshell-system
```

**Why:** The namespace may contain finalizers or lingering resources. Deleting it ensures a clean slate for the next install.

## Step 5 — Remove the HelmChartRepository (Optional)

If you no longer want the chart in the Developer Catalog:

```bash
oc delete helmchartrepository openshell-ztm-demo
```

## What Remains

After cleanup:

| Component | Status |
|-----------|--------|
| ZTVP (Vault, ESO, Keycloak, SPIRE, ACS) | ✅ Untouched |
| OpenShell Gateway | ❌ Removed |
| OpenShell Supervisor | ❌ Removed |
| Sandbox (Pi agent) | ❌ Removed |
| Vault secret (openshell/) | Optional — removed if Step 3 was done |
| HelmChartRepository | Optional — removed if Step 5 was done |

## Reinstalling

To reinstall from scratch, go back to **Deploy from Catalog** and repeat the steps. The chart is idempotent — it will recreate everything.

---

## Congratulations! 🎉

You have completed the OpenShell on Zero Trust Demo Lab. You have:

- ✅ Deployed an AI agent sandbox from the OpenShift Developer Catalog
- ✅ Used Pi + DeepSeek to write and run real code inside a sandbox
- ✅ Proved that network access is restricted to approved endpoints only
- ✅ Proved that filesystem access is locked by Landlock LSM
- ✅ Proved that API credentials are never visible inside the sandbox
- ✅ Explored how Vault, ESO, SPIFFE, ACS, and Keycloak integrate
- ✅ Dynamically updated the zero-trust policy

**The takeaway:** AI agents can be powerful and secure at the same time. Zero Trust is not about blocking functionality — it's about ensuring every action is explicitly authorized.
