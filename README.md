# gke-config

Config Sync source of truth for `esther-cluster-1` (project `cloud-sa-sandbox-1`, fleet).

Format: **unstructured** (any folder layout; Config Sync reads every YAML file under the sync path).

```
namespaces/config-sync-demo.yaml     Namespace
config-sync-demo/configmap.yaml      ConfigMap app-settings
config-sync-demo/deployment.yaml     Deployment hello (2 replicas)
config-sync-demo/resourcequota.yaml  ResourceQuota guardrail
```

## How it works

1. A **RootSync** object on the cluster says "watch repo X, branch main, path /".
2. The `reconciler-manager` in `config-management-system` runs a reconciler pod that pulls the repo (about every 15 s).
3. The reconciler runs `kubectl apply`-style updates so the cluster matches git. If someone changes a managed object by hand, it changes it back (**drift correction**).
4. Delete a file from git -> Config Sync deletes the object (**pruning**).

Ownership rule: Terraform (`../gke-terraform-dev`) owns namespace `dev`. Don't put `dev` objects here, or the two tools will fight.

## Labs

| Lab | Do | Expect |
|---|---|---|
| 1 Sync | Wire RootSync (Console) | Console shows *Synced* + commit SHA; `kubectl -n config-sync-demo get all` |
| 2 Change | Edit `configmap.yaml` message, push | New SHA in Console in under 1 min; `kubectl -n config-sync-demo get cm app-settings -o yaml` |
| 3 Drift | `kubectl -n config-sync-demo scale deploy hello --replicas=5` | Back to 2 within seconds |
| 4 Prune | `git rm config-sync-demo/resourcequota.yaml`, push | Quota disappears from the cluster |
| 5 Error | Push a broken YAML | Console shows the error, last good commit stays applied |
| 6 Rollback | `git revert HEAD`, push | Cluster returns to the previous state |

Handy CLI:

```bash
gcloud container fleet config-management describe --project cloud-sa-sandbox-1
kubectl get rootsync -n config-management-system
kubectl describe rootsync root-sync -n config-management-system
nomos status        # optional CLI, ships with gcloud components (`gcloud components install nomos`)
```
