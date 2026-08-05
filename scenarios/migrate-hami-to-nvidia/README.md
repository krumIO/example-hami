# Migrate from HAMi back to nvidia-device-plugin

The rollback for [`../migrate-nvidia-to-hami/`](../migrate-nvidia-to-hami), and the emergency procedure when HAMi has stopped GPU scheduling.

This direction has a hazard the forward one doesn't: **workloads may be requesting resources that are about to stop existing.** A pod asking for `nvidia.com/gpumem` becomes unschedulable the moment HAMi leaves, and the cluster tells you nothing beyond a generic "0/N nodes available". Find those first.

## Emergency short version

HAMi is installed, GPU pods are Pending on `hami-scheduler`, and you need scheduling back now.

```sh
# 1. Remove the release. This deletes the webhook, which is what's redirecting
#    pods to a scheduler that can't place them — that's the actual outage.
helm uninstall hami -n kube-system

# 2. Confirm the webhook is gone.
kubectl get mutatingwebhookconfigurations | grep -i hami

# 3. Delete pods pinned to the now-absent scheduler. schedulerName is immutable,
#    so they must be recreated, not patched. Name them explicitly.
kubectl -n <ns> delete pod <pod>

# 4. If the stock plugin is still installed, it's already serving GPUs again.
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name) \(.status.allocatable["nvidia.com/gpu"] // "-")"'
```

Controllers recreate the deleted pods with `default-scheduler` and they schedule against whole GPUs. If the stock plugin was already uninstalled, do step 2 of the full procedure first.

This is worth rehearsing, because the failure it fixes is easy to cause by accident: installing HAMi with default values on a cluster whose default runtime is `runc` gives you a crash-looping device plugin **and** a live webhook, so GPU scheduling stops cluster-wide while the chart reports a healthy install.

## Full procedure

### 0. Find workloads that depend on HAMi-only resources

```sh
kubectl get pods -A -o json | jq -r '
  .items[] | select([.spec.containers[].resources.limits // {} | keys[]]
    | any(. == "nvidia.com/gpumem" or . == "nvidia.com/gpucores" or . == "nvidia.com/gpumem-percentage"))
  | "\(.metadata.namespace)/\(.metadata.name)"'
```

Check the owning controllers too — the spec you need to edit lives in a Deployment or StatefulSet, not the pod:

```sh
kubectl get deploy,statefulset -A -o json | jq -r '
  .items[] | select((.spec.template.spec.containers[].resources.limits // {} | keys[]?)
    | startswith("nvidia.com/gpumem") or startswith("nvidia.com/gpucores"))
  | "\(.kind) \(.metadata.namespace)/\(.metadata.name)"'
```

Rewrite each to whole GPUs. There is no fractional equivalent under the stock plugin, so pods that shared a card now need one each — **confirm you have the physical GPUs to land them all** before going further. Skipping this is what turns a rollback into an incident.

```yaml
# before (HAMi)                      # after (stock)
limits:                              limits:
  nvidia.com/gpu: 1                    nvidia.com/gpu: 1
  nvidia.com/gpumem: 2048            # gpumem/gpucores removed
  nvidia.com/gpucores: 30
```

### 1. Neutralize the webhook first

Before touching any plugin. This way, workloads that restart during the migration come back on `default-scheduler` instead of queueing against a scheduler that's about to disappear.

```sh
helm upgrade hami hami-charts/hami -n kube-system --reuse-values \
  --set scheduler.admissionWebhook.enabled=false
```

Or, without touching the release:

```sh
kubectl delete mutatingwebhookconfiguration hami-webhook
```

Or per namespace, if you're going gradually:

```sh
kubectl label namespace <ns> hami.io/webhook=ignore --overwrite
```

Confirm nothing new is being mutated:

```sh
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.schedulerName=="hami-scheduler") | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'
```

### 2. Bring back the stock plugin, selecting nothing

```sh
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm install nvdp nvdp/nvidia-device-plugin \
  -n nvidia-device-plugin --create-namespace --version 0.14.5 -f nvdp-values.yaml
```

```yaml
# nvdp-values.yaml
runtimeClassName: nvidia     # same NVML requirement as HAMi
nodeSelector:
  gpu-plugin: nvdp
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

### 3. Move nodes back, one at a time

```sh
NODE=<node>
kubectl cordon "$NODE"

# GPU consumers only — a full drain evicts storage daemons and can block for
# an hour on replica eviction, for movement this doesn't need.
kubectl get pods -A --field-selector "spec.nodeName=$NODE" -o json | jq -r '
  .items[] | select([.spec.containers[].resources.limits // {} | keys[]] | any(startswith("nvidia.com/")))
  | "\(.metadata.namespace) \(.metadata.name)"' | while read ns p; do kubectl -n "$ns" delete pod "$p"; done

kubectl label node "$NODE" gpu-plugin=nvdp --overwrite

kubectl -n nvidia-device-plugin get pods -o wide | grep "$NODE"
kubectl get node "$NODE" -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
kubectl uncordon "$NODE"
```

The count should return to the physical number — `1` per card, not the split. Don't test this by watching `nvidia.com/gpumem` disappear from `allocatable`: it was never there under HAMi either.

### 4. Uninstall HAMi

Once no node carries `gpu-plugin=hami`:

```sh
helm uninstall hami -n kube-system

kubectl get mutatingwebhookconfiguration hami-webhook
kubectl -n kube-system get ds hami-device-plugin
kubectl -n kube-system get deploy hami-scheduler
```

The chart owns 18 objects and no CRDs, so that's the whole release.

### 5. Clean up node state

`helm uninstall` does not remove the annotations HAMi wrote. They're harmless but stale, and they'll mislead both the next operator and a future reinstall:

```sh
for n in $(kubectl get nodes -o name); do
  kubectl annotate "$n" hami.io/node-nvidia-register- hami.io/node-handshake- 2>/dev/null
done
```

A node where HAMi never successfully registered has none of these.

### 6. Recreate the workloads

Anything still carrying `schedulerName: hami-scheduler` is unschedulable now. Apply the step 0 edits and restart:

```sh
kubectl rollout restart deployment -n <ns> <name>

kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.schedulerName=="hami-scheduler" or .status.phase=="Pending")
  | "\(.metadata.namespace)/\(.metadata.name) \(.spec.schedulerName) \(.status.phase)"'
```

## Failure signatures

| Symptom | Cause |
|---|---|
| Pending, `insufficient nvidia.com/gpumem` | workload still requests HAMi-only resources; step 0 skipped |
| Pending with `schedulerName: hami-scheduler` after uninstall | pods predate the change; `schedulerName` is immutable, delete them |
| Fewer pods fit than before | expected — sharing is gone, each pod needs a whole card |
| Count still shows the split after relabel | HAMi's plugin is still on the node; check the selector actually changed |
| Stock plugin CrashLoopBackOff after reinstall | missing `runtimeClassName: nvidia`, same root cause as the forward direction |

## Validated

Executed on RKE2 + SL Micro under Rancher, rolling three nodes back from HAMi 2.9.0. Capacity returned to `nvidia.com/gpu: 1` within seconds of each relabel. Every node that had registered with HAMi retained `hami.io/node-nvidia-register` and `hami.io/node-handshake` after `helm uninstall` — step 5 is not optional housekeeping.

The emergency short version is not hypothetical: it's what was run when a default-values HAMi install took GPU scheduling down on a live cluster.
