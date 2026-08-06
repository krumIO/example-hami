# Reference

Dry lookup for HAMi as deployed by this repo. For *why* any of it works the way it does, see [`how-hami-works.md`](how-hami-works.md); for procedures, see the [scenarios](../scenarios).

Accurate as of **2026-08-06**, HAMi chart/app **v2.9.0**.

## Vendor support

The authoritative list is the vendor directories under [`pkg/device/`](https://github.com/Project-HAMi/HAMi/tree/master/pkg/device) in the source, not the chart values and not the roadmap.

| Vendor | Status | Resource names |
| --- | --- | --- |
| NVIDIA | Released, v2.9.0 | `nvidia.com/gpu`, `nvidia.com/gpumem`, `nvidia.com/gpucores` |
| AMD | Merged to `master` only — **no release** | `amd.com/gpu`, `amd.com/gpumem`, `amd.com/gpucores` |
| Intel | Not supported; roadmap only | — (use [Intel's own plugins](https://github.com/intel/intel-device-plugins-for-kubernetes), `gpu.intel.com/i915`) |
| Cambricon, Enflame, Hygon, Iluvatar, Kunlun, MetaX, Mthreads, Vastai, Ascend, AWS Neuron | Backends present in v2.9.0 | see `pkg/device/<vendor>/` |
| Biren | On `master`, not in v2.9.0 | see `pkg/device/biren/` |

### Release positions for the two that matter here

| | NVIDIA | AMD | Intel |
| --- | --- | --- | --- |
| In v2.9.0 (2026-05-19) | yes | partial code only, no sharing | no |
| Merged to `master` | yes | **2026-08-03**, [PR #2290](https://github.com/Project-HAMi/HAMi/pull/2290) | no PR, no issue, no milestone |
| Docs channel | stable | `next` only (stable path 404s) | none |
| Target release | shipped | none published | none |

HAMi minors have landed roughly quarterly — v2.7.0 Sep 2025, v2.8.0 Jan 2026, v2.9.0 May 2026. That is an observed cadence, not a commitment.

Two traps in reading the above:

- `pkg/device/amd` **does** exist in the v2.9.0 tag, as a partial implementation predating the sharing feature (7.5 KB vs 13 KB on `master`). Directory presence in a release tag is not feature availability.
- The roadmap still lists "Support AMD GPU device" as planned. That is correct — it tracks shipped releases, and AMD has not shipped. A roadmap entry tells you a feature is unavailable; it doesn't tell you how close it is. Check for a merged PR for that.

### AMD-specific constraints

From the [`next` AMD guide](https://project-hami.io/docs/next/userguide/amd-device/enable-amd-gpu-sharing), for when it ships:

- AMD Instinct/ROCm hardware; validated against ROCm 7.0.2, MI300X VF in the worked example.
- AMD GPU Operator recommended, with **its native device-plugin disabled** — it claims `amd.com/gpu` too.
- HAMi's `amd-device-plugin`, chart/image 0.0.1+.
- Workload images need **glibc ≥ 2.34**. Ubuntu 20.04, RHEL 8, musl/Alpine are unsupported. This constrains *application* images, not just platform ones.
- Chart 2.9.0 defaults declare `amd.com/gpu-memory`; the `next` docs use `amd.com/gpumem`. Set names explicitly.

**Hygon DCU is not AMD** — separate silicon with AMD-derived heritage. `hygon.com/dcunum` says nothing about Instinct support.

## Node labels this repo uses

Neither label is created by HAMi. Both are conventions you apply, and keeping them out of any chart or machine-pool template is deliberate — they're the control surface you operate by hand during a migration, and something else asserting them means fighting your own automation mid-cutover.

| Label | Means | Values | When you need it |
| --- | --- | --- | --- |
| `gpu-vendor` | what hardware the node has | `nvidia`, `amd`, `intel` | only in a **multi-vendor** cluster, so each vendor's plugin finds its own hardware |
| `gpu-plugin` | which NVIDIA plugin serves this node | `hami`, `nvdp` | any cluster running **both** HAMi and `nvidia-device-plugin`, i.e. during and after a migration |

They are different axes and both can apply to one node. A single-vendor NVIDIA cluster mid-migration needs only `gpu-plugin`. A multi-vendor cluster that has finished migrating needs only `gpu-vendor`. A multi-vendor cluster mid-migration needs both, and HAMi's selector should match both:

```yaml
devicePlugin:
  nvidiaNodeSelector:
    gpu: null            # remove the chart default — Helm deep-merges maps
    gpu-vendor: nvidia   # this node has NVIDIA hardware
    gpu-plugin: hami     # ...and HAMi, not nvdp, serves it
```

Do not select on vendor labels emitted by autodiscovery (`gpu0-vendor`, `gpu0-driver`, `feature.node.kubernetes.io/*`). On a machine with both an integrated and a discrete GPU those describe the wrong device — see [`../scenarios/mixed-gpu-vendors/`](../scenarios/mixed-gpu-vendors).

## Node annotations HAMi writes

Written by the device plugin, read by the scheduler-extender. Not cleaned up by `helm uninstall`.

| Annotation | Contains |
| --- | --- |
| `hami.io/node-nvidia-register` | per-card registry: `id`, `count` (= `deviceSplitCount`), `devmem` MiB, `devcore` %, `type`, `health` |
| `hami.io/node-amd-register` | same, for the AMD backend |
| `hami.io/node-handshake` | liveness timestamp between plugin and scheduler |

```sh
kubectl get node <node> -o json | jq -r '.metadata.annotations | to_entries
  | map(select(.key | startswith("hami.io/node-"))) | .[] | "\(.key) = \(.value)"'
```

**Memory and core capacity never appear in `allocatable`.** Only the multiplied `nvidia.com/gpu` count does; the rest lives in these annotations. Looking for `gpumem` in `kubectl describe node` and not finding it does not mean the install is broken.

## Pod-level controls

| Field / label | Effect |
| --- | --- |
| `resources.limits."nvidia.com/gpu"` | number of virtual devices |
| `resources.limits."nvidia.com/gpumem"` | MiB ceiling, enforced in-container by HAMI-core |
| `resources.limits."nvidia.com/gpumem-percentage"` | same as a percentage of the card |
| `resources.limits."nvidia.com/gpucores"` | percent of SM time |
| `schedulerName: hami-scheduler` | required; injected by the webhook when enabled |
| `runtimeClassName: nvidia` | required wherever the default runtime is `runc` |
| label `hami.io/webhook: ignore` | exempt this pod from webhook mutation |
| annotation `nvidia.com/use-gputype` | restrict to matching card models |

Namespaces are exempted the same way: `kubectl label namespace <ns> hami.io/webhook=ignore`.

## Chart values that matter

Full defaults: `helm show values hami-charts/hami --version 2.9.0`.

| Value | Default | Notes |
| --- | --- | --- |
| `devicePlugin.runtimeClassName` | `""` | **set to `nvidia`** unless the cluster default runtime is already nvidia; empty is the most common first-install failure |
| `devicePlugin.createRuntimeClass` | `false` | leave false when the cluster owns the RuntimeClass |
| `devicePlugin.nvidiaNodeSelector` | `{gpu: "on"}` | **deep-merges** — add `gpu: null` to remove the default, then verify with `helm template` |
| `devicePlugin.deviceSplitCount` | `10` | virtual devices per card |
| `devicePlugin.deviceMemoryScaling` | `1` | >1 oversubscribes VRAM; failures surface as in-container OOM |
| `devicePlugin.deviceCoreScaling` | `1` | same for compute |
| `scheduler.admissionWebhook.enabled` | `true` | cluster-wide from install; **disable during a phased migration** |
| `schedulerName` | `hami-scheduler` | |
| `resourceName` / `resourceMem` / `resourceCores` | `nvidia.com/gpu`, `/gpumem`, `/gpucores` | rename only if something else already owns the names |
| `devices.amd.customresources` | `[amd.com/gpu, amd.com/gpu-memory]` | note the naming mismatch with the `next` docs |
| `dra.enabled` | `false` | DRA is a roadmap item; subchart `hami-dra` ships but is off |

## Objects the chart owns

18, no CRDs — so `helm uninstall` is complete except for the node annotations above.

`ServiceAccount` ×2 · `ConfigMap` ×3 · `ClusterRole` ×2 · `ClusterRoleBinding` ×4 · `Role` · `RoleBinding` · `Service` ×2 · `DaemonSet` (`hami-device-plugin`) · `Deployment` (`hami-scheduler`) · `MutatingWebhookConfiguration` (`hami-webhook`)

## Verification one-liners

```sh
# plugin healthy on every selected node
kubectl -n kube-system get ds hami-device-plugin

# what each node advertises, any vendor
kubectl get nodes -o json | jq -r '.items[] | . as $n |
  ($n.status.allocatable | to_entries | map(select(.key | test("nvidia.com|amd.com|gpu.intel.com")))) as $g |
  select($g | length > 0) | "\($n.metadata.name)  \($g | map("\(.key)=\(.value)") | join(" "))"'

# registration detail
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'

# anything routed to hami-scheduler
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.schedulerName=="hami-scheduler")
  | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'
```

## Failure signatures

| Symptom | Cause |
| --- | --- |
| `invalid device discovery strategy` / `Incompatible strategy detected auto` | plugin pod on `runc`, can't reach NVML — set `runtimeClassName` |
| `FailedCreatePodSandBox: no runtime for "nvidia" is configured` | node has no nvidia runtime in containerd; affects every GPU plugin equally |
| DaemonSet `DESIRED 0` | selector matches nothing, or deep-merged into requiring two labels |
| Plugin Running but registers no devices | node's GPU isn't the vendor the plugin handles (often an iGPU) |
| GPU pods `Pending` on `hami-scheduler`, no events | webhook live, no node registered |
| Allocatable GPU count flapping | two plugins claiming the same resource name on one node |
| Pods `Pending` requesting `gpumem` after leaving HAMi | stock plugin never advertises it; rewrite the requests |
| Pods keep the old scheduler after a change | `schedulerName` is immutable — recreate them |
| AMD workload fails on a library symbol | image glibc < 2.34 |
