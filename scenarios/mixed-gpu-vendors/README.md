# Mixed GPU vendors in one cluster

Running HAMi where the cluster contains more than one kind of accelerator — NVIDIA plus AMD, or NVIDIA plus something HAMi doesn't handle at all, or a single machine with both a discrete card and an integrated GPU.

The short version: **coexistence is easy, selection is where it breaks.** Different vendors advertise different resource names, so their device plugins never fight. What goes wrong is landing a plugin on a node whose GPU it doesn't understand — and the labels that would tell you which node is which are frequently wrong.

Read [`../../docs/prerequisites.md`](../../docs/prerequisites.md) first for which vendors HAMi covers.

## Why coexistence is not the problem

The conflict the [migration scenarios](../migrate-nvidia-to-hami/) exist to manage — two plugins on one node, capacity flapping — happens because `nvidia-device-plugin` and HAMi both register the resource name `nvidia.com/gpu` with kubelet, and kubelet's registry is last-writer-wins per resource name.

Two *different* vendors never collide, because the names differ:

| Plugin | Advertises |
| --- | --- |
| HAMi (NVIDIA backend) | `nvidia.com/gpu`, plus `nvidia.com/gpumem` / `gpucores` accounting |
| HAMi (AMD backend) | `amd.com/gpu`, `amd.com/gpumem`, `amd.com/gpucores` |
| Intel device plugin | `gpu.intel.com/i915` |
| AMD GPU Operator's own plugin | `amd.com/gpu` |

So an Intel plugin and HAMi can run on the same node with no arrangement between them. The one pairing that *does* collide is HAMi's AMD backend and the AMD GPU Operator's native device plugin — both claim `amd.com/gpu`, which is exactly why HAMi's AMD guide tells you to disable the operator's plugin.

## Case 1: NVIDIA and AMD, both under HAMi

HAMi handles both, with a per-vendor backend and a per-vendor device plugin. Declare the AMD resource names and let each plugin select its own nodes:

```yaml
devices:
  amd:
    customresources:
      - amd.com/gpu
      - amd.com/gpumem
      - amd.com/gpucores

devicePlugin:
  runtimeClassName: nvidia
  nvidiaNodeSelector:
    gpu: null                # remove the chart default; Helm deep-merges maps
    gpu-vendor: nvidia
```

Label nodes by what they actually contain, then deploy HAMi's `amd-device-plugin` selecting `gpu-vendor: amd`:

```sh
kubectl label node <nvidia-node> gpu-vendor=nvidia --overwrite
kubectl label node <amd-node>    gpu-vendor=amd    --overwrite
```

**`gpu-vendor` is a second axis, not a replacement for `gpu-plugin`.** The other scenarios select on `gpu-plugin` (`hami` vs `nvdp`), which answers a different question — *which of two competing NVIDIA plugins serves this node* — and only matters while both are installed. This one answers *what hardware is here*. A single-vendor cluster mid-migration needs only `gpu-plugin`; a multi-vendor cluster that has finished migrating needs only `gpu-vendor`; a multi-vendor cluster mid-migration needs both, and HAMi's selector should match both:

```yaml
  nvidiaNodeSelector:
    gpu: null
    gpu-vendor: nvidia   # NVIDIA hardware...
    gpu-plugin: hami     # ...served by HAMi, not nvdp
```

Both conventions are defined in [`../../docs/reference.md`](../../docs/reference.md#node-labels-this-repo-uses).

Workloads then request whichever vendor they need, and the scheduler places them:

```yaml
resources:
  limits:
    amd.com/gpu: 1
    amd.com/gpumem: 4096
    amd.com/gpucores: 30
```

**Version gate:** AMD vGPU is not in a released HAMi yet. It merged to `master` on 2026-08-03 ([PR #2290](https://github.com/Project-HAMi/HAMi/pull/2290)), after v2.9.0 (2026-05-19) was cut, with no milestone and so no published target release. Running this today means running `master`. The `pkg/device/amd` directory in the v2.9.0 tag is an earlier partial implementation and will not give you sharing.

Constraints specific to the AMD path, all from [HAMi's AMD guide](https://project-hami.io/docs/next/userguide/amd-device/enable-amd-gpu-sharing) — that page is in the **`next` (unreleased)** docs, so treat this as pre-release:

- AMD Instinct/ROCm hardware; validated against ROCm 7.0.2, MI300X VF in the worked example.
- The AMD GPU Operator is recommended, with **its native device-plugin disabled** so it doesn't fight HAMi for `amd.com/gpu`.
- HAMi's `amd-device-plugin`, chart/image 0.0.1 or newer.
- Workload images need **glibc ≥ 2.34**. Ubuntu 20.04, RHEL 8, and musl/Alpine images are unsupported — a constraint on *your application images*, not just the platform, so check before promising sharing to a team.
- The chart's 2.9.0 defaults list `amd.com/gpu-memory`, while the `next` docs use `amd.com/gpumem`. Set the names explicitly rather than relying on defaults, and make the values match what your plugin actually advertises.

## Case 2: NVIDIA under HAMi, another vendor under its own plugin

Intel has no HAMi backend, so its GPUs are served by [Intel's device plugins](https://github.com/intel/intel-device-plugins-for-kubernetes) instead. It is on the roadmap — "Support Intel GPU device" under [planned features](https://project-hami.io/docs/next/contributor/roadmap#planned-features) — but with **no target version and nothing in flight**: as of 2026-08-06 there is no `pkg/device/intel`, no issue, no PR, and no milestone. Plan for Intel's own plugins as the arrangement you'll be running for the foreseeable future, not a stopgap measured in months.

That said, the per-vendor node labelling below costs nothing now and makes a future swap cheap — if a HAMi Intel backend does land, nodes already labelled `gpu-vendor: intel` change plugin rather than architecture.

Nothing needs to mediate between the two — the resource names differ — but two things need attention:

**HAMi's selector must exclude the Intel nodes.** Its plugin only understands NVIDIA; on an Intel-only node it finds nothing through NVML.

**Fractional sharing works differently per vendor.** Intel's i915 plugin shares via `sharedDevNum`, which is a plugin-level setting, not a per-pod request — there's no `gpu.intel.com/gpumem` equivalent to ask for a slice size. Workloads that need a specific memory bound can't express it that way, so don't design a scheduling policy that assumes uniform semantics across vendors.

The same applies to any other accelerator with its own plugin: coexistence free, semantics not portable.

## Case 3: one machine, discrete card plus integrated GPU

This is the one that produces confusing failures, because **generic GPU labellers describe the wrong device.** A labeller that enumerates PCI devices and reports the first one finds the integrated GPU in the CPU package and never mentions the discrete card:

```text
gpu-count      1
gpu0-driver    amdgpu
gpu0-vendor    Advanced-Micro-Devices-Inc.-AMD-ATI
gpu0-product   Renoir-Radeon-Vega-Series-Radeon-Vega-Mobile-Series
```

That is a real node with a discrete NVIDIA card in it, running NVIDIA workloads. The labels describe the Ryzen APU's iGPU; the accelerator anyone cares about isn't represented. Consequences:

- A selector of `gpu0-vendor=NVIDIA...` matches **zero** nodes despite the NVIDIA cards being present.
- `gpu-count: 1` counts the iGPU and undercounts wherever both exist.
- An iGPU of this class is not a HAMi target — it has no ROCm support and isn't an Instinct part — so labelling the node "AMD" and pointing an AMD plugin at it fails too.

**Select on a label you control.** A pool label from your provisioner, or one you set by hand, that means "this node has an NVIDIA GPU I intend to share":

```sh
kubectl label node <node> gpu-vendor=nvidia --overwrite
```

Never wire `nvidiaNodeSelector` to a vendor label emitted by autodiscovery. The chart's own default, `gpu: "on"`, has the same weakness in a different way — it's vendor-blind, so it will happily match an Intel-only control-plane node that picked up a generic GPU label from the same labeller.

## Verify what each node really has

Autodiscovery labels are a hint. These are evidence:

```sh
# NVIDIA: what the driver itself sees. An iGPU is invisible to NVML, so this
# and any generic gpu-count label legitimately disagree.
kubectl run nvidia-probe --rm -it --restart=Never \
  --image=nvidia/cuda:12.2.0-base-ubuntu22.04 \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' \
  -- nvidia-smi --query-gpu=name,memory.total --format=csv

# What each node is actually advertising, by vendor.
kubectl get nodes -o json | jq -r '.items[] | . as $n |
  ($n.status.allocatable | to_entries | map(select(.key | test("nvidia.com|amd.com|gpu.intel.com")))) as $g |
  select($g | length > 0) | "\($n.metadata.name)  \($g | map("\(.key)=\(.value)") | join(" "))"'

# HAMi registration is per-vendor: node-nvidia-register / node-amd-register.
kubectl get node <node> -o json | jq -r '.metadata.annotations | to_entries
  | map(select(.key | startswith("hami.io/node-"))) | .[] | "\(.key) = \(.value)"'
```

## Failure signatures

| Symptom | Cause |
| --- | --- |
| HAMi plugin Running but registers no devices | node's GPU isn't NVIDIA — likely an iGPU matched by a vendor-blind selector |
| `amd.com/gpu` capacity flapping | HAMi's AMD plugin and the AMD GPU Operator's native plugin both on the node; disable the operator's |
| AMD workload starts then fails on a library symbol | image glibc < 2.34; rebuild on a newer base |
| Intel GPUs invisible to HAMi | expected — no HAMi backend ([roadmap only](https://project-hami.io/docs/next/contributor/roadmap#planned-features), no target version); deploy Intel's plugin alongside |
| Pod requests `nvidia.com/gpumem` and stays Pending on a node with GPUs | those GPUs are a different vendor; the request only means something on NVIDIA nodes |

## Validated

**Not tested.** This scenario is assembled from HAMi's source (`pkg/device/` vendor backends, present for `amd` in v2.9.0), the `next` AMD guide, and the vendor-label behaviour observed on a real hybrid cluster — six nodes whose autodiscovery labels reported AMD Renoir and Intel Alder Lake iGPUs while the discrete NVIDIA cards went unlabelled, which is where Case 3 comes from.

What has **not** been exercised here: HAMi's AMD backend on real Instinct hardware, and Intel plugin coexistence. Both cases are structurally sound but unverified — treat the AMD path in particular as pre-release, since its documentation is.
