# Cleanup

This module walks through removing the OpenShell demo from the cluster. The ZTVP components remain untouched — only the OpenShell resources are removed.

## Step 1 — Uninstall the Helm Release

From the bastion:

```bash
helm uninstall openshell-ztm-demo -n openshell-system
```

**Why:** `helm uninstall` removes all resources created by the chart: Deployments, StatefulSets, Services, Routes, ConfigMaps, and Secrets. Helm tracks these resources via the release metadata.

Expected:

```
release "openshell-ztm-demo" uninstalled
```

## Step 2 — Verify Resources Are Gone

```bash
oc get all -n openshell-system
```

Expected: only the namespace remains (or a few lingering pods in `Terminating` state that will clear in seconds).

```
No resources found in openshell-system namespace.
```

## Step 3 — Vault Secret (No Action Needed)

The API key stored in Vault at path `hub/infra/openshell/deepseek` persists after Helm uninstall. You do **not** need to clean it up:

- If you reinstall the chart, the pre-install hook will **overwrite** the key with the new value
- The key is only accessible via Kubernetes auth (path `hub/`), so it cannot be read without a valid ServiceAccount token in the `openshell-system` namespace (which was deleted)
- The ZTVP Vault uses Kubernetes auth — there is no root token available from the bastion

**Why leave it:** Vault secrets are external to Kubernetes. Cleaning them requires Vault admin access, which is managed by the ZTVP pattern. The key is effectively orphaned and inaccessible after the namespace is deleted.

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
