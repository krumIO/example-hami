# Prerequisites

## First: does HAMi cover your accelerators?

HAMi virtualizes **NVIDIA** GPUs, and has per-vendor backends for a set of others. The authoritative list is the vendor directories under [`pkg/device/`](https://github.com/Project-HAMi/HAMi/tree/master/pkg/device) in the source — as of v2.9.0: `nvidia`, `amd`, `ascend`, `awsneuron`, `cambricon`, `enflame`, `hygon`, `iluvatar`, `kunlun`, `metax`, `mthreads`, `vastai`, with `biren` added since.

**AMD is supported** — see [Enable AMD GPU sharing](https://project-hami.io/docs/next/userguide/amd-device/enable-amd-gpu-sharing). Two caveats worth knowing before you plan around it:

- That page is in the **`next` (unreleased) docs**; there is no equivalent page under the stable docs for v2.9.0, even though `pkg/device/amd` ships in the v2.9.0 tag. Treat the documented workflow as pre-release.
- It needs **AMD Instinct/ROCm** hardware (validated against ROCm 7.0.2, MI300X VF in the worked example), the **AMD GPU Operator with its native device-plugin disabled**, and HAMi's own `amd-device-plugin` (chart/image 0.0.1+). Workload images must have **glibc ≥ 2.34** — Ubuntu 20.04, RHEL 8, and musl/Alpine images are explicitly unsupported.

Resource names are `amd.com/gpu`, `amd.com/gpumem`, `amd.com/gpucores`, declared in values under `devices.amd.customresources`. Note the chart's 2.9.0 defaults list `amd.com/gpu-memory` rather than the `amd.com/gpumem` the `next` docs use — the names are configurable, so set them explicitly to match whichever plugin you actually deploy instead of relying on the default.

**Intel is not supported yet, but it is planned.** There is no `pkg/device/intel`, no `gpu.intel.com` resource anywhere in the source, and no docs page — but "Support Intel GPU device" appears under planned features on [HAMi's roadmap](https://project-hami.io/docs/next/contributor/roadmap#planned-features), with no version target or date attached. Until it lands, Intel Arc/Flex/Gaudi need [Intel's own device plugins](https://github.com/intel/intel-device-plugins-for-kubernetes), whose i915 plugin has a separate `sharedDevNum` fractional-sharing mechanism.

If you're choosing tooling for a fleet that includes Intel accelerators, this is the entry to watch — the coexistence pattern in [`../scenarios/mixed-gpu-vendors/`](../scenarios/mixed-gpu-vendors) is designed so that adopting a future HAMi Intel backend is a plugin swap on those nodes rather than a re-architecture.

Two other roadmap entries worth knowing about when planning: **DRA support** (the chart already ships a `dra.enabled: false` toggle and a `hami-dra` subchart, so this is partly in flight) and **MIG support**. Read the roadmap as direction, not commitment — "Support AMD GPU device" is still listed as planned even though `pkg/device/amd` ships in v2.9.0 and has a `next` guide, so delivered features can lag the list.

One name worth not misreading: **Hygon DCU is not AMD.** It's separate silicon with AMD-derived heritage, and `hygon.com/dcunum` says nothing about whether an Instinct card will work.

If your cluster has more than one vendor in it, read [`../scenarios/mixed-gpu-vendors/`](../scenarios/mixed-gpu-vendors) — coexistence is straightforward, but node selection is where it goes wrong.

## The four requirements

Four things must be true before HAMi will work. Three are about the nodes; the fourth is a decision you make at install time.

| | Requirement | Verify with |
| --- | --- | --- |
| 1 | NVIDIA driver + container toolkit on every GPU node | `nvidia-smi` on the host |
| 2 | A `RuntimeClass` the cluster can select the NVIDIA runtime with | `kubectl get runtimeclass nvidia` |
| 3 | Each GPU node can actually honour that RuntimeClass | the probe pod in §3 |
| 4 | A decision about the admission webhook | §4 |

Everything HAMi's device plugin does depends on reaching NVML, and 1–3 are what make that possible.

## 1. Driver and container toolkit

Each GPU node needs the **NVIDIA driver** and the **container toolkit** (`nvidia-container-runtime`). How they get there is distribution-specific — baked into the OS image, installed by the NVIDIA GPU Operator, or a package install.

On RKE2, if the toolkit is present when the node starts, RKE2 detects it, writes the runtime into its own containerd config, and the `nvidia` handler becomes usable. No further configuration needed.

## 2. The RuntimeClass

Most clusters leave `runc` as the default runtime and select the NVIDIA one per pod. That requires a `RuntimeClass` object:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

```sh
kubectl get runtimeclass nvidia
```

Some distributions create this for you; RKE2 configures the containerd runtime but does not create the Kubernetes object, so on RKE2 you apply it yourself.

Every GPU-touching pod then has to name it, **including HAMi's own device plugin**. The chart ships `devicePlugin.runtimeClassName: ""`, which is the most common reason a first install fails, so set it to the name of the RuntimeClass from above:

```yaml
devicePlugin:
  runtimeClassName: nvidia   # must match metadata.name of the RuntimeClass
  createRuntimeClass: false  # the cluster owns that object, not the chart
```

Leave `runtimeClassName` empty **only** if your cluster's default runtime is already `nvidia` — that is, if a pod with no `runtimeClassName` at all can already see a GPU. If you're unsure, set it to `nvidia`: naming a RuntimeClass that a node can honour is never wrong.

Your own GPU workloads need the same field:

```yaml
spec:
  runtimeClassName: nvidia
```

## 3. Verify each node, not just the cluster

`RuntimeClass` is cluster-scoped, so its existence says nothing about whether a *given* node can honour it. A node rebuilt in the last hour can carry every GPU label your tooling assigns and still have no `nvidia` runtime in containerd — labels come from the machine pool, the runtime comes from the OS image, and they arrive at different times.

Run something and see:

```sh
kubectl run rt-probe --rm -it --restart=Never --image=busybox \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' \
  -- true
```

Clean exit means the node can host a GPU plugin. If it hangs in `ContainerCreating` with `no runtime for "nvidia" is configured`, that node's NVIDIA enablement hasn't finished — fix it before installing, because neither HAMi nor `nvidia-device-plugin` can run there.

Repeat per GPU node. This is the check most worth doing: it costs seconds and rules out the failure that looks most like a HAMi bug.

## 4. Decide the webhook setting before installing

HAMi ships a `MutatingWebhookConfiguration` (`vgpu.hami.io`) that rewrites `schedulerName` to `hami-scheduler` on **any** pod requesting GPU resources, cluster-wide, from the moment the chart is installed. Its selectors exclude only objects labelled `hami.io/webhook: ignore`, so by default it catches everything.

Two properties make this worth deciding deliberately rather than accepting the default:

It takes effect **whether or not the device plugin works**. Install with a broken plugin and every GPU pod in the cluster is redirected to a scheduler with no registered devices — they sit `Pending` with no obvious cause, while the chart reports a healthy install and the crash-looping DaemonSet goes unnoticed in `kube-system`.

Its `failurePolicy: Ignore` does **not** protect you. That governs what happens when the webhook is *unreachable*, not when it succeeds. A successful mutation into a broken scheduler is still a successful mutation.

**Pick one:**

*Every GPU node will run HAMi* — leave it on:

```yaml
scheduler:
  admissionWebhook:
    enabled: true
```

*Some nodes will keep `nvidia-device-plugin`, or you're migrating in stages* — turn it off, and name the scheduler explicitly on the workloads you move:

```yaml
scheduler:
  admissionWebhook:
    enabled: false
```

```yaml
# in each migrated workload's pod spec
spec:
  schedulerName: hami-scheduler
```

The middle option, if you'd rather keep the webhook on during a staged rollout, is to exempt the namespaces that aren't ready yet and remove the label as each one migrates:

```sh
kubectl label namespace <ns> hami.io/webhook=ignore --overwrite
```

## Reading a healthy install

Once installed, these are the signals that mean it worked — knowing them prevents chasing a problem that isn't there.

```sh
kubectl -n kube-system get pods -o wide | grep hami-device-plugin   # Running, not restarting
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

The annotation is where per-card detail lives:

```text
[{"id":"GPU-...","count":10,"devmem":6144,"devcore":100,"type":"NVIDIA GeForce RTX 2060...","health":true}]
```

`count` is your `deviceSplitCount`, `devmem` the card's real memory in MiB. Allocatable `nvidia.com/gpu` should read physical cards × `deviceSplitCount`.

**`nvidia.com/gpumem` and `nvidia.com/gpucores` never appear in `allocatable`.** The device plugin advertises only the multiplied `nvidia.com/gpu` count; memory and core accounting live in that annotation, which HAMi's scheduler-extender reads at scheduling time. Looking for `gpumem` in `kubectl describe node` and not finding it will convince you a working install is broken.

## If verification fails

| Symptom | Cause | Fix |
| --- | --- | --- |
| Plugin `CrashLoopBackOff`, log says `invalid device discovery strategy` / `Incompatible strategy detected auto` | pod is running under `runc` and can't reach NVML — the message blames the toolkit, but the host is usually fine | set `devicePlugin.runtimeClassName: nvidia` (§2) |
| Plugin running but registers no devices, on a node that does have a GPU | that GPU isn't NVIDIA — an AMD/Intel iGPU is invisible to NVML | tighten the node selector; see the mixed-vendor section above |
| Pod stuck `ContainerCreating`, `no runtime for "nvidia" is configured` | that node has no NVIDIA runtime in containerd | finish node enablement (§1); affects both plugins identically |
| DaemonSet `DESIRED 0` | node selector matches nothing | check the labels, and that `helm template` rendered the selector you expected |
| GPU pods `Pending` on `hami-scheduler`, no events | webhook is live but no node has a registered device | fix the plugin, or disable the webhook (§4) |
| Allocatable GPU count flapping | two device plugins on one node | give them disjoint node selectors |
