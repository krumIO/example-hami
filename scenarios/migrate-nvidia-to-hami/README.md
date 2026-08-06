# Migrate from nvidia-device-plugin to HAMi

Cutting a cluster with GPUs already in use over to HAMi, one node at a time, without a window where `nvidia.com/gpu` vanishes cluster-wide.

Read [`../../docs/prerequisites.md`](../../docs/prerequisites.md) first, and skim [`../migrate-hami-to-nvidia/`](../migrate-hami-to-nvidia) before you start — knowing how the rollback works is what makes this low-risk.

## Why one node at a time

Both plugins register the resource name `nvidia.com/gpu` with kubelet. Kubelet's device-plugin registry is keyed by resource name and is last-writer-wins, so two plugins on one node fight and the advertised capacity flaps.

They conflict **per node, not per cluster**. Two DaemonSets with disjoint node selectors coexist happily — which is the whole basis of this procedure. Put both plugins behind the same label with different values:

```text
gpu-plugin=nvdp   ->  nvidia-device-plugin
gpu-plugin=hami   ->  HAMi
```

Then a migration is `kubectl label`, and a rollback is the same command with the other value. A node with neither label runs no plugin and advertises no GPUs, which is the correct safe intermediate state.

## 0. Preflight

```sh
# RuntimeClass exists cluster-wide...
kubectl get runtimeclass nvidia

# ...but verify each node can actually honour it. A recently rebuilt node can
# carry every GPU label your tooling assigns and still have no nvidia runtime.
kubectl run rt-probe --rm -it --restart=Never --image=busybox \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' -- true

# Baseline for rollback.
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name) \(.status.allocatable["nvidia.com/gpu"] // "-")"'

# Every GPU workload. All of these get recreated before you're done.
kubectl get pods -A -o json | jq -r '
  .items[] | select([.spec.containers[].resources.limits // {} | keys[]] | any(startswith("nvidia.com/")))
  | "\(.metadata.namespace)/\(.metadata.name) sched=\(.spec.schedulerName)"'
```

## 1. Partition the existing plugin

Pin every GPU node to what it already runs, so nothing moves yet:

```sh
for n in $(kubectl get nodes -l <your-gpu-label> -o name); do
  kubectl label "$n" gpu-plugin=nvdp --overwrite
done
```

Then make the stock plugin step aside for HAMi nodes. Either point its `nodeSelector` at `gpu-plugin: nvdp`, or — less disruptive, since it doesn't require every node to be labelled first — give it an anti-affinity:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: gpu-plugin
              operator: NotIn
              values: ["hami"]
```

Capacity should be unchanged afterwards. If a node lost its GPU here, stop and fix that before continuing.

## 2. Install HAMi selecting nothing

Install with `nvidiaNodeSelector` matching **zero** nodes — the chart lands, the scheduler starts, the device plugin has nowhere to run. See [`../deploy-with-helm/`](../deploy-with-helm) for the values.

**Set `scheduler.admissionWebhook.enabled: false` for the duration of the migration.** Enabled, it rewrites `schedulerName` to `hami-scheduler` on every GPU pod cluster-wide — including pods on nodes still running the stock plugin, which then never schedule. During the migration, set `schedulerName: hami-scheduler` explicitly on the workloads you move. Turn the webhook on at the end.

(The alternative — leaving it enabled and labelling every not-yet-migrated namespace `hami.io/webhook=ignore` — works if you're doing the whole cluster in one sitting.)

```sh
kubectl -n kube-system get deploy hami-scheduler          # Running
kubectl -n kube-system get ds hami-device-plugin          # DESIRED 0
```

## 3. Canary one node

Cordon it and move **only the GPU consumers** off. A full `kubectl drain` is the wrong tool: it evicts storage daemons too, and on a Longhorn cluster with `node-drain-policy: block-for-eviction` that blocks until every replica rebuilds elsewhere — roughly an hour per node, for storage movement a device-plugin swap doesn't need.

```sh
NODE=<node>
kubectl cordon "$NODE"

kubectl get pods -A --field-selector "spec.nodeName=$NODE" -o json | jq -r '
  .items[] | select([.spec.containers[].resources.limits // {} | keys[]] | any(startswith("nvidia.com/")))
  | "\(.metadata.namespace) \(.metadata.name)"' | while read ns p; do kubectl -n "$ns" delete pod "$p"; done

kubectl label node "$NODE" gpu-plugin=hami --overwrite
```

The swap itself takes seconds — the old plugin's pod goes, the new one arrives, kubelet re-registers.

```sh
kubectl -n kube-system get pods -o wide | grep hami-device-plugin
```

`CrashLoopBackOff` here is almost always the missing `runtimeClassName`. `ContainerCreating` with `no runtime for "nvidia" is configured` means the node was never GPU-enabled — the stock plugin couldn't run there either.

Now confirm registration. **Don't look for `nvidia.com/gpumem` in `allocatable`; it is never there.** Memory and core accounting live in an annotation:

```sh
kubectl get node "$NODE" -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
# [{"id":"GPU-...","count":10,"devmem":6144,"devcore":100,...,"health":true}]

kubectl get node "$NODE" -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
# 10   (physical x deviceSplitCount)

kubectl uncordon "$NODE"
```

Prove enforcement before trusting it — capacity numbers alone don't tell you the limit is real:

```sh
kubectl apply -f ../../samples/vgpu-smoke-test.yaml   # reports its limit, not the card
kubectl apply -f ../../samples/two-pods-one-gpu.yaml  # both Running, same node
```

## 4. Roll the rest

Repeat step 3 per node. A cluster with some nodes on each plugin is a valid steady state, so there's no pressure to finish in one sitting.

```sh
kubectl get nodes -L gpu-plugin -o json | jq -r '.items[] |
  select(.status.allocatable["nvidia.com/gpu"]) |
  "\(.metadata.name) \(.metadata.labels["gpu-plugin"] // "-") gpu=\(.status.allocatable["nvidia.com/gpu"])"'
```

## 5. Retire the old plugin and enable the webhook

Once no node carries `gpu-plugin=nvdp`:

```sh
helm uninstall nvdp -n nvidia-device-plugin

helm upgrade hami hami-charts/hami -n kube-system --reuse-values \
  --set scheduler.admissionWebhook.enabled=true
```

## 6. Recreate GPU workloads

The webhook only mutates at admission and `schedulerName` is immutable, so existing pods keep whatever scheduler they were created with. Until they're recreated they're still whole-GPU consumers on `default-scheduler`:

```sh
kubectl rollout restart deployment -n <ns> <name>
```

Then move them onto fractional requests — the reason for all this. `nvidia.com/gpu: 1` alone still consumes a whole virtual device without bounding memory:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
    nvidia.com/gpumem: 2048
    nvidia.com/gpucores: 30
```

## Rollback

Per node, any time: cordon, move GPU pods, `kubectl label node <n> gpu-plugin=nvdp --overwrite`. Nothing is one-way until you uninstall the old plugin in step 5, and [the reverse scenario](../migrate-hami-to-nvidia/) reinstalls it in minutes.

## Failure signatures

| Symptom | Cause |
|---|---|
| `invalid device discovery strategy` / `Incompatible strategy detected auto` | `devicePlugin.runtimeClassName` unset; pod is on `runc` and can't reach NVML |
| `FailedCreatePodSandBox: no runtime for "nvidia" is configured` | node never completed NVIDIA enablement; affects both plugins identically |
| GPU pods Pending on `hami-scheduler`, no events | webhook enabled while HAMi isn't registered on any node |
| Capacity flapping between physical and split counts | both plugins on one node — the selectors overlap |
| `hami-device-plugin` DESIRED=0 | selector matches nothing; check `helm template` for a deep-merged `nodeSelector` |
| `gpumem` missing from `describe node` | expected, always — check the annotation instead |

## Validated

Executed end to end on RKE2 + SL Micro (6 GB Turing cards) under Rancher, three GPU nodes, HAMi 2.9.0, then rolled back. Plugin swaps took seconds each. A pod limited to 1024 MiB reported `0MiB / 1024MiB` from `nvidia-smi`; a second at 2048 MiB ran concurrently on the same card with its own limit. With the webhook enabled, a pod naming no scheduler was mutated to `hami-scheduler` and reported 1536 MiB for a 1536 request.

Not covered: a production inference workload on a shared card. Sharing was demonstrated with CUDA base images.
