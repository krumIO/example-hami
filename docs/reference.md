# Reference

Dry lookup for HAMi as deployed by this repo. For *why* any of it works the way it does, see [`how-hami-works.md`](how-hami-works.md); for procedures, see the [scenarios](../scenarios).

Scenarios and manifests on this page target HAMi chart/app **v2.9.0**. Vendor support was re-checked against **v2.10.0** on **2026-10-28**.

Release positions move. Re-check them against the [HAMi releases page](https://github.com/Project-HAMi/HAMi/releases) rather than trusting this page for a newer version.

## Vendor support

The authoritative list is the vendor directories under [`pkg/device/`](https://github.com/Project-HAMi/HAMi/tree/master/pkg/device) in the source, not the chart values and not the roadmap.

| Vendor | Status | Resource names |
| --- | --- | --- |
| NVIDIA | Released, v2.9.0 | `nvidia.com/gpu`, `nvidia.com/gpumem`, `nvidia.com/gpucores` |
| AMD | Released in **v2.10.0** (2026-08-21) | `amd.com/gpu`, `amd.com/gpumem`, `amd.com/gpumem-percentage`, `amd.com/gpucores` |
| Intel | Not supported; roadmap only | — (use [Intel's own plugins](https://github.com/intel/intel-device-plugins-for-kubernetes), `gpu.intel.com/i915`) |
| Cambricon, Enflame, Hygon, Iluvatar, Kunlun, MetaX, Mthreads, Vastai, Ascend, AWS Neuron | Backends present in v2.9.0 | see `pkg/device/<vendor>/` |
| Biren | Released in v2.10.0 | see `pkg/device/biren/` |

### Release positions for the two that matter here

| | NVIDIA | AMD | Intel |
| --- | --- | --- | --- |
| In v2.9.0 (2026-05-19) | yes | partial code only, no sharing | no |
| Merged to `master` | yes | **2026-08-03**, [PR #2290](https://github.com/Project-HAMi/HAMi/pull/2290) | no PR, no issue, no milestone |
| Released | yes | **v2.10.0**, 2026-08-21 | no |

HAMi minors have landed roughly quarterly: v2.7.0 Sep 2025, v2.8.0 Jan 2026, v2.9.0 May 2026, v2.10.0 Aug 2026. That is an observed cadence, not a commitment.

AMD shipped 18 days after it merged, which is the useful lesson from the version-gate note this page used to carry: a merged PR with no milestone meant *soon*, not *indefinite*. The trap that did hold is the other one. `pkg/device/amd` exists in the v2.9.0 tag as a partial implementation predating the sharing feature (7.5 KB vs 13 KB), so directory presence in a release tag is still not feature availability.

### AMD-specific constraints

From the [AMD guide](https://project-hami.io/docs/userguide/amd-device/enable-amd-gpu-sharing), as of v2.10.0:

- **Radeon and Instinct both**, not Instinct alone. The requirement is a ROCm-supported AMD GPU with the `amdgpu` driver, `/dev/kfd` and `/dev/dri`. Earlier versions of this page said Instinct only; that was the scope of the worked example, not of the backend.
- Two components beyond the HAMi chart: [`amd-device-plugin`](https://github.com/Project-HAMi/amd-device-plugin), which publishes each GPU in the node annotation `hami.io/node-amd-register` and hands out CU slices, and [`amd-hami-core`](https://github.com/Project-HAMi/amd-hami-core) (`libamvgpu`), which enforces the memory limit in-container.
- **Nothing else may register `amd.com/gpu` on those nodes.** The kubelet serves one resource from one plugin and refuses the second registration. That rules out the AMD GPU Operator's own plugin and the upstream [ROCm device plugin](https://github.com/ROCm/k8s-device-plugin) alike. Disable whichever one is already there.
- Workload images need **glibc ≥ 2.34**. Ubuntu 20.04, RHEL 8, musl/Alpine are unsupported. This constrains *application* images, not just platform ones.
- The kernel-level memory cap needs cgroup v2 with the `dmem` controller, which is **kernel 6.14 or newer**, plus the systemd cgroup driver. Below that only the in-container hook enforces the limit, and a process that drops `LD_AUDIT` escapes it. A whole-GPU request gets neither.
- Non-root pods need `runAsUser` and `supplementalGroups` for the node's `video` and `render` groups, whose ids differ per node.
- Chart 2.9.0 defaults declare `amd.com/gpu-memory`; v2.10.0 uses `amd.com/gpumem`. Set names explicitly.

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
