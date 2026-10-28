# Your GPU labels are lying to you

*Part 5 of [a seven-part field report](../blog.md). The shortest post, and the one most likely to apply to you even if you never touch HAMi.*

While writing the mixed-vendor guidance we did the obvious sanity check: list the GPU nodes and their vendors. Here is what `dev-machines` said about itself.

```text
5cd0270j29-hp                          gpu0-driver=amdgpu   Renoir-Radeon-Vega
5cd03149ty-hp                          gpu0-driver=amdgpu   Renoir-Radeon-Vega
m1nrkd013655035-asustek                gpu0-driver=amdgpu   Renoir-Radeon-Vega
bn1003ae13410-azw                      gpu0-driver=i915     Alder-Lake-N UHD
bn1003bg41831-azw                      gpu0-driver=i915     Alder-Lake-N UHD
ub70854-onlogic                        gpu0-driver=i915     Alder-Lake-P
```

Every node is AMD or Intel. There is not one NVIDIA GPU in that list.

The three `amdgpu` nodes are our GPU workers. They contain a GTX 1660 Ti, a GTX 1660 Ti, and an RTX 2060. They have been running NVIDIA workloads for months.

## What the labeller is actually describing

A labeller that walks PCI devices and reports the first one finds the **integrated** GPU in the CPU package. These are Ryzen APUs, so `gpu0` is the Radeon Vega iGPU. The discrete NVIDIA card is `gpu1` and never gets mentioned. On the Intel-based control-plane nodes it is the Alder Lake iGPU, and there is no discrete card at all.

Nothing is malfunctioning. The labeller reported the first GPU it found, accurately. It is just not the GPU anyone cares about.

The consequences are concrete:

- A selector of `gpu0-vendor=NVIDIA...` matches **zero** nodes despite three NVIDIA cards being present.
- `gpu-count: 1` counts the iGPU and undercounts wherever both exist.
- An iGPU of this class is not a target for anything — no ROCm, not an Instinct part — so labelling the node "AMD" and pointing an AMD plugin at it fails too.

There is a second-order version of this we hit while migrating: node labels arriving *before* node capability. A reflashed node carried every GPU label its machine pool assigns while its NVIDIA OS transition was still incomplete. It could run neither device plugin. The labels were current; the node was not ready.

## The rule

Select on a label you set deliberately. Not one emitted by autodiscovery.

```sh
kubectl label node <node> gpu-vendor=nvidia --overwrite
```

HAMi's chart default has the same weakness in a different form. `devicePlugin.nvidiaNodeSelector` defaults to `{gpu: "on"}`, which is vendor-blind — it will match an Intel-only control-plane node that picked up a generic GPU label from the same labeller, and then the plugin lands somewhere it cannot work.

For evidence rather than hints, ask the driver:

```sh
kubectl run nvidia-probe --rm -it --restart=Never \
  --image=nvidia/cuda:12.2.0-base-ubuntu22.04 \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' \
  -- nvidia-smi --query-gpu=name,memory.total --format=csv
```

An iGPU is invisible to NVML, so this and any generic `gpu-count` label will legitimately disagree. That disagreement is the point.

## Where AMD and Intel actually stand

> **Update, 2026-10-28.** AMD shipped in HAMi **v2.10.0** on 2026-08-21, eighteen days after the merge described below. The scope is wider than we recorded: Radeon as well as Instinct. The section is left as written because the reasoning is the point, and the postscript at the end says which parts held up. Current requirements are in [`docs/reference.md`](../docs/reference.md#amd-specific-constraints).

Since we were in mixed-vendor territory anyway, we checked properly. We had initially written that HAMi supports neither. That was half wrong, and the correction is worth having.

**AMD is supported** — but not in any released version. The sharing feature merged to `master` on 2026-08-03, after v2.9.0 was cut on 2026-05-19. There is no milestone on the PR and none open for the next minor, so there is no published target release. Using it today means running `master`.

Do not be misled by `pkg/device/amd` existing in the v2.9.0 tag. It does, as a partial implementation predating the sharing feature — 7.5 KB there versus 13 KB on `master`. Directory presence in a release tag is not feature availability, which is a trap we walked into ourselves.

When it does ship: AMD Instinct/ROCm hardware, the AMD GPU Operator with **its native device-plugin disabled** (it claims `amd.com/gpu` too, the one genuine cross-vendor collision), and workload images with **glibc ≥ 2.34** — no Ubuntu 20.04, no RHEL 8, no Alpine.

**Intel is not supported, and is further out.** "Support Intel GPU device" appears on the roadmap, but that is the only trace: no `pkg/device/intel`, no `gpu.intel.com` resource anywhere in the source, no docs page, and no issue, PR, or milestone. There is nothing in flight to watch merge. Intel Arc/Flex/Gaudi need [Intel's own device plugins](https://github.com/intel/intel-device-plugins-for-kubernetes), whose i915 plugin has a separate `sharedDevNum` sharing mechanism, and that should be planned as the long-term arrangement rather than a stopgap.

One naming trap: **Hygon DCU is not AMD.** Separate silicon with AMD-derived heritage. `hygon.com/dcunum` tells you nothing about whether an Instinct card will work.

## Reading a roadmap

A small methodological note, since we got this wrong twice in opposite directions.

We first claimed AMD was unsupported by reasoning from the *shape of values.yaml* — no top-level `amdResourceName` like the other vendors have. AMD's config lives under `devices.amd.customresources` instead. Config-file shape is not evidence of feature support.

Then we described the roadmap as lagging, because it still lists AMD as planned even though the code has merged. That was also wrong. The roadmap tracks what has **shipped**, and AMD has not shipped. It is correct.

The rule that came out of it: a roadmap entry reliably means *not available to you yet*. What it cannot tell you is distance. For that, look for a merged PR. AMD and Intel sit in the same roadmap section and are nowhere near the same distance out — one is a `master` build away, the other has no implementation at all.

## Mixed vendors are fine; mixed selectors are not

The reassuring part: different vendors advertise different resource names, so their device plugins never collide. An Intel plugin serving `gpu.intel.com/i915` and HAMi serving `nvidia.com/gpu` coexist on the same node with no arrangement between them. The [per-node conflict](04-migrating-both-directions.md) that shapes every migration procedure is specifically same-name contention.

What breaks is selection. And selection breaks because the labels lie.

---

## Postscript: what the version-gate reasoning got right

AMD shipped in v2.10.0 on 2026-08-21, eighteen days after the PR merged. Scoring the calls above:

**The merged-PR heuristic held.** "A roadmap entry reliably means not available to you yet; for distance, look for a merged PR" was the right instinct. A merged PR with no milestone turned out to mean weeks, not quarters.

**Reading "no milestone" as "no date" was too cautious.** We treated the absent milestone as the signal and concluded "running this today means running `master`". The merge itself was the stronger signal, and it was already sitting there.

**The directory-presence trap held exactly as written.** `pkg/device/amd` in the v2.9.0 tag is still a partial implementation. Anyone who checked for the directory rather than the release still gets the wrong answer.

**The hardware scope was wrong.** We wrote "AMD Instinct/ROCm hardware", taking the worked example for the supported set. The shipped backend covers Radeon too, and the real requirement is a ROCm-supported GPU with `amdgpu`, `/dev/kfd` and `/dev/dri`. Worth remembering that a vendor's example configuration is not their compatibility list.

**One collision we named too narrowly.** We called the AMD GPU Operator's plugin "the one genuine cross-vendor collision". It is not the only one: the upstream ROCm device plugin claims `amd.com/gpu` as well, and the kubelet refuses whichever registers second. The rule is one plugin per resource name, whoever ships it.

---

That is the end of the general field report. Two Rancher-specific posts follow if that is your platform: [the click-through path](06-hami-on-rancher-manual.md) and [delivery with Fleet](07-hami-on-rancher-fleet.md).

Otherwise, the [repo](../README.md) has the runbooks and samples, and the [tutorial](../docs/tutorial.md) is about twenty minutes on a scratch cluster, ending with two pods sharing one card.
