# How HAMi works

Background for operating it. Nothing here is a procedure — the [scenarios](../scenarios) are the procedures, and [`reference.md`](reference.md) is the lookup. This is the mental model that makes the procedures make sense, and that explains why the failures look the way they do.

## The problem it solves

Kubernetes allocates GPUs as whole devices. `nvidia-device-plugin` tells kubelet "this node has 1 `nvidia.com/gpu`", a pod asks for 1, and that pod owns the card until it exits. There is no notion of half a GPU, so a model server that needs 2 GB of a 24 GB card holds all 24, and the second pod that wants a GPU waits — even though the hardware is idle.

For large training jobs that's correct: they want the whole device. For inference, notebooks, CI, and small models it's badly wasteful, and it's the case HAMi addresses.

## Three moving parts

HAMi is not one component, and knowing which is which turns most of its failure modes from mysterious into obvious.

**The device plugin** (`hami-device-plugin`, a DaemonSet) runs on each GPU node, talks to the driver through NVML, and registers with kubelet. Instead of advertising one `nvidia.com/gpu` per card it advertises `deviceSplitCount` of them — 10 by default. It also writes what it found to a node annotation: card UUID, real memory, core count, health.

**The scheduler** (`hami-scheduler`, a Deployment) is a stock kube-scheduler plus a HAMi extender. Kubernetes itself only understands the integer count of `nvidia.com/gpu`; it has no idea what `nvidia.com/gpumem` means. The extender is what reads those requests, compares them against the per-card detail in the node annotation, and decides which physical card on which node has room. **This is why memory and core capacity never appear in `allocatable`** — they aren't Kubernetes resources at all, they're bookkeeping between the plugin and the extender.

**The admission webhook** (`vgpu.hami.io`) rewrites `schedulerName` to `hami-scheduler` on any pod requesting GPU resources. Without it, pods go to `default-scheduler`, which cannot do the fractional accounting and will happily overcommit a card.

The split explains the estate's worst HAMi outage. The webhook is installed with the chart and starts redirecting immediately; the device plugin has to actually start and register before the scheduler has anything to place pods on. Install with a broken plugin — a missing `runtimeClassName` will do it — and you get a live webhook sending every GPU pod in the cluster to a scheduler with an empty device registry. Helm reports success, the chart looks deployed, and GPU scheduling has stopped cluster-wide.

## What "sharing" actually means

The limit isn't advisory. HAMi injects a library (`HAMI-core`) into the container that intercepts CUDA calls, so `nvidia-smi` inside a pod granted `nvidia.com/gpumem: 1024` reports a 1024 MiB card even though the hardware has 6144 MiB. Allocations past the ceiling fail inside the container rather than starving a neighbour.

Two consequences:

**Oversubscription becomes an application bug.** `deviceMemoryScaling: 2` advertises twice the real VRAM. The scheduler will place pods believing the space exists; whichever container actually reaches for it gets an allocation failure that looks like its own bug. Scheduling stays green while a workload dies.

**Interception has requirements.** It's a library in the container's address space, so it depends on that image — which is why AMD support carries a glibc ≥ 2.34 floor that rules out Alpine and older base images. A platform capability constrains application images, which is unusual and worth telling application teams before promising them GPU sharing.

Compute limits are softer than memory: `gpucores` bounds SM time, but neighbours on one card still contend for bandwidth and cache. Two pods that fit on paper can still be slower than they'd be alone.

## Why two device plugins fight

Kubelet's device-plugin registry is keyed by **resource name** and is last-writer-wins. Both `nvidia-device-plugin` and HAMi register `nvidia.com/gpu`. Run both on one node and each re-registration displaces the other, so advertised capacity flaps between 1 and 10 and the scheduler makes decisions against whichever it last saw.

Two things follow, and they're the basis of every procedure in this repo:

**The conflict is per node, not per cluster.** Two DaemonSets with disjoint node selectors never interact. That's why migration is a node-at-a-time relabel rather than a cluster-wide cutover, and why there's never a window with no GPUs anywhere.

**Different vendors never conflict at all.** `amd.com/gpu` and `nvidia.com/gpu` are different keys. The collisions are all within AMD: HAMi's AMD backend against the AMD GPU Operator's plugin, and against the upstream ROCm device plugin. All three want `amd.com/gpu`, the kubelet serves one resource from one plugin, and the second registration is refused. HAMi's AMD guide says to disable whichever is already there.

## Why the runtime matters so much

The plugin reads the GPU through NVML, which is only reachable under the NVIDIA container runtime. Most clusters leave `runc` as the default and select the NVIDIA runtime per pod via `runtimeClassName`, so the plugin — an ordinary pod — has to name it like anything else.

The failure is loud but misdirected:

```text
E factory.go:135] Incompatible strategy detected auto
E factory.go:136] If this is a GPU node, did you configure the NVIDIA Container Toolkit?
```

It asks about the host toolkit, so the instinct is to go check the node. The node is usually fine; the pod is just running under the wrong runtime.

The genuinely node-level version of this failure looks different — `no runtime for "nvidia" is configured`, at sandbox creation, before the container starts. That one means the node really hasn't been GPU-enabled, and it breaks every GPU plugin equally, HAMi and stock alike.

## Why labels lie

Node labels seem like the obvious way to find GPU nodes, and on hybrid hardware they are actively misleading. A labeller that walks PCI devices and reports the first one finds the **integrated** GPU in the CPU package. A machine with a Ryzen APU and a discrete NVIDIA card reports itself as AMD; a discrete card in an Intel-based box gets an `i915` label. The accelerator you care about isn't described, and a selector keyed on vendor labels matches the wrong nodes — or none.

Hence the convention in [`reference.md`](reference.md): select on labels you set deliberately, and treat autodiscovery output as a hint to verify rather than a fact to act on. Verify with `nvidia-smi` in a pod, which reports what the driver actually sees.

## What this costs you

Worth stating plainly, because "share the GPU" sounds free:

- **A scheduler and a webhook** in the cluster's critical path for every GPU pod.
- **A constraint on application images** (the interception library).
- **Contention** that shows up as latency variance rather than as a scheduling error.
- **A migration** whenever you change your mind in either direction, because `schedulerName` is immutable and every GPU workload has to be recreated.

On a cluster where GPUs are already saturated by jobs that legitimately want whole cards, none of that buys anything. The win is concentrated where cards sit idle inside exclusive claims.
