# Cleanup

This module walks through removing the OpenShell demo from the cluster. The ZTVP components remain **untouched** — only the OpenShell resources are removed.

> **Why keep ZTVP?** The ZTVP is a shared platform. Other applications may be using Vault, Keycloak, SPIRE, and ACS. Uninstalling OpenShell only removes what the Helm chart created — the Gateway, Supervisor, sandbox, and associated resources.

## Step 1 — Uninstall the Helm Release

From the bastion:

```bash
helm uninstall openshell-ztm-demo -n openshell-system
```

> **What does `helm uninstall` do?** It removes all Kubernetes resources that Helm created for this release: the Gateway StatefulSet, the Supervisor, the sandbox pod, the Route, ConfigMaps, Secrets, and Services. Helm tracks every resource it creates (stored as release metadata in a Secret), so it knows exactly what to delete.

Expected:

```
release "openshell-ztm-demo" uninstalled
```

## Step 2 — Verify Resources Are Gone

```bash
oc get all -n openshell-system
```

Expected: only the namespace remains, possibly with the completed setup job and any pods in `Terminating` state:

```
NAME                           READY   STATUS      RESTARTS   AGE
pod/pi-sandbox-setup-1-xxxxx   0/1     Completed   0          7m
```

Or if everything has finished terminating:

```
No resources found in openshell-system namespace.
```

> **Why is the setup job still there?** Helm hook jobs with `hook-delete-policy: before-hook-creation` are cleaned up on the *next* install, not on uninstall. This completed job is harmless. Deleting the namespace in Step 4 removes it entirely.

## Step 3 — Vault Secret (No Action Needed)

The API key stored in Vault at path `hub/infra/openshell/maas` persists after Helm uninstall. You do **not** need to clean it up:

- If you **reinstall** the chart, the pre-install hook will **overwrite** the key with the new value
- The key is only accessible via Kubernetes auth (path `hub/`), so it cannot be read without a valid ServiceAccount token in the `openshell-system` namespace — which was just deleted
- The ZTVP Vault uses Kubernetes auth — there is no root token available from the bastion

> **Why leave it?** Vault secrets are external to Kubernetes. Cleaning them requires Vault admin access, which is managed by the ZTVP pattern (not by individual applications). The key is effectively **orphaned and inaccessible** after the namespace is deleted — it will be overwritten on the next install.

## Step 4 — Remove the Namespace (Optional)

```bash
oc delete namespace openshell-system
```

> **What are namespace finalizers?** When you delete a namespace, Kubernetes runs "finalizers" — cleanup hooks that ensure all resources inside are properly removed. If the namespace gets stuck in `Terminating` state, it usually means a finalizer is waiting for a resource to be cleaned up. In normal operation, this completes within 30 seconds.

## Step 5 — Remove the Keycloak Demo User (Optional)

If you completed the **User Authentication** module, you created a `demo-engineer` user in Keycloak. This user is **not** managed by the Helm chart, so it persists after uninstall. To remove it:

```bash
KEYCLOAK_URL=https://keycloak.<CLUSTER_DOMAIN>
ADMIN_PASSWORD=$(oc get secret keycloak-admin-user -n keycloak-system -o jsonpath='{.data.password}' | base64 -d)

ADMIN_TOKEN=$(curl -sk "$KEYCLOAK_URL/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=$ADMIN_PASSWORD" \
  -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

USER_ID=$(curl -sk "$KEYCLOAK_URL/admin/realms/ztvp/users?username=demo-engineer" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -c "import sys,json; u=json.load(sys.stdin); print(u[0]['id'] if u else '')")

if [ -n "$USER_ID" ]; then
  curl -sk -o /dev/null -w "Delete user: HTTP %{http_code}\n" -X DELETE \
    "$KEYCLOAK_URL/admin/realms/ztvp/users/$USER_ID" \
    -H "Authorization: Bearer $ADMIN_TOKEN"
else
  echo "No demo-engineer user found (already cleaned up or never created)"
fi
```

Expected: `Delete user: HTTP 204`

> **Why isn't this automatic?** The `demo-engineer` user was created manually via the Keycloak Admin API in Module 9a — not by the Helm chart. Helm only manages resources it creates. The `openshell-gateway` OIDC client *is* cleaned up automatically (it was created by a `KeycloakRealmImport` CR in the chart), but manually-created users need manual deletion.

## Step 6 — Remove the HelmChartRepository (Optional)

If you no longer want the chart in the Developer Catalog:

```bash
oc delete helmchartrepository openshell-ztm-demo
```

> This removes the chart from the OpenShift Console's Developer Catalog. The chart itself still exists on GitHub Pages — you can re-register it anytime by applying the HelmChartRepository CR again.

## What Remains After Cleanup

| Component | Status | Notes |
|-----------|--------|-------|
| ZTVP (Vault, ESO, Keycloak, SPIRE, ACS) | ✅ **Untouched** | These are shared platform services — they serve other applications too |
| OpenShell Gateway | ❌ Removed | Helm uninstall deleted it |
| OpenShell Supervisor | ❌ Removed | Deleted with the release |
| Sandbox (Pi agent) | ❌ Removed | Deleted with the release |
| Vault secret | 🔒 Orphaned | Inaccessible without the namespace; will be overwritten on reinstall |
| Keycloak `openshell-gateway` client | ❌ Removed | Helm uninstall deleted the `KeycloakRealmImport` CR |
| Keycloak `demo-engineer` user | ⚠️ Persists | Removed if Step 5 was done; harmless if left |
| HelmChartRepository | Optional | Removed if Step 6 was done |

## Reinstalling

To reinstall from scratch, go back to **Deploy from Catalog** and repeat the steps. The Helm chart is **idempotent** — it will recreate everything cleanly, including overwriting the Vault secret.

> **What does "idempotent" mean?** It means running the install multiple times produces the same result. The chart uses "if not exists" logic for Vault keys and sandbox creation, so you don't get duplicates or errors on reinstall.

---

## Congratulations! 🎉

You have completed the **OpenShell on Zero Trust Demo Lab**. Here's everything you proved:

| Module | What you demonstrated |
|--------|---------------------|
| **Deploy from Catalog** | One-click deployment from the OpenShift Developer Catalog via Helm |
| **AI Agent Coding** | Pi + MAAS (gpt-oss-120b) wrote real code (Hello World + Express.js REST API) autonomously |
| **Network Zero Trust** | Only 3 hosts reachable; Google, example.com, and exfil servers instantly denied |
| **Filesystem Zero Trust** | Only /sandbox and /tmp writable; /etc, /usr, / all locked by Landlock LSM |
| **Credential Security** | Real API key never visible in sandbox — only resolver tokens that are useless outside the boundary |
| **ZTVP Integration** | Vault stores secrets, ESO syncs them, SPIFFE provides identity, ACS scans images, Keycloak authenticates users |
| **User Authentication** | OIDC flow from Keycloak, JWT token acquisition and decoding, Gateway validation |
| **Dynamic Policy** | Added pypi.org live → HTTP 200 → Removed → blocked again, all without restart |

## The Takeaway

**AI agents can be powerful and secure at the same time.**

Zero Trust is not about blocking functionality — it's about ensuring every action is **explicitly authorized**. The agent can write code, install packages, and call APIs. It just can't do anything that isn't in the policy.

This is the future of AI agent deployment in production: **maximum capability, minimum risk**.
