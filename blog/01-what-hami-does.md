# What HAMi actually does, and what it costs

*Part 1 of [a five-part field report](../blog.md) on running HAMi in anger.*

Kubernetes allocates GPUs as whole devices. `nvidia-device-plugin` tells kubelet "this node has 1 `nvidia.com/gpu`", a pod asks for 1, and that pod owns the card until it exits. There is no notion of half a GPU.

For a training job that is correct behaviour: it wants the whole device. For inference, notebooks, CI, and small models it is badly wasteful. We had a 0.5B parameter model holding an entire RTX 2060 to do nothing in particular, and a second pod that wanted a GPU sitting `Pending` next to an idle card:

```text
NAME    READY   STATUS    RESTARTS   AGE
hog-a   1/1     Running   0          10s
hog-b   0/1     Pending   0          10s

... 0/1 nodes are available: 1 Insufficient nvidia.com/gpu.
```

HAMi exists to fix exactly that. It works. But it is three components rather than one, the sharing is enforced in a place that will surprise you, and it costs more than "turn it on".

## Three components, not one

Almost every confusing HAMi failure becomes obvious once you know which of these three is involved.

The **device plugin** is a DaemonSet on each GPU node. It talks to the driver through NVML and registers with kubelet, advertising `deviceSplitCount` virtual devices per card instead of one. Ten, by default. It also writes what it found to a node annotation: card UUID, real memory, core count, health.

The **scheduler** is a stock kube-scheduler plus a HAMi extender. Kubernetes itself only understands the integer count of `nvidia.com/gpu`; it has no idea what `nvidia.com/gpumem` means. The extender reads those requests, compares them against the per-card detail in the node annotation, and picks a physical card on a node with room.

That split explains something that trips up everyone at least once: **memory and core capacity never appear in `allocatable`.**

```sh
kubectl get node <node> -o jsonpath='{.status.allocatable}' | jq
# nvidia.com/gpu: "10"     ← present
# nvidia.com/gpumem        ← not here, and never will be
```

They are not Kubernetes resources. They are bookkeeping between the plugin and the extender, and they live here:

```sh
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
[{"id":"GPU-e1b4fb02-...","count":10,"devmem":6144,"devcore":100,"type":"NVIDIA GeForce RTX 2060...","health":true}]
```

We spent real time looking for `gpumem` in `describe node` before working this out. If you go looking and don't find it, nothing is broken.

The **admission webhook** rewrites `schedulerName` to `hami-scheduler` on any pod requesting GPU resources. Without it, pods go to `default-scheduler`, which cannot do fractional accounting and will happily overcommit a card.

The webhook is where the danger lives, and [part 2](02-the-install-that-broke-everything.md) is entirely about that.

## Sharing means the limit is enforced inside your container

This is the part that made us trust it. HAMi injects a library — `HAMI-core` — that intercepts CUDA calls. A pod granted `nvidia.com/gpumem: 1024` does not merely have a scheduler-side accounting entry. It genuinely cannot see more than 1024 MiB:

```text
$ kubectl logs vgpu-smoke-test
name, memory.total [MiB]
NVIDIA GeForce RTX 2060 with Max-Q Design, 1024 MiB
[HAMI-core Msg(45:...:multiprocess_memory_limit.c:703)]: Cleanup on exit for PID 45
```

The physical card has 6144 MiB. The container sees 1024. Two pods with different ceilings ran concurrently on that same card and each saw only its own.

Two consequences follow from *where* that enforcement happens, and both matter more than they first appear.

**Oversubscription turns into an application bug.** `deviceMemoryScaling: 2` advertises twice the real VRAM. The scheduler places pods believing the space exists; whichever container actually reaches for it gets an allocation failure that looks like its own fault. Your scheduling stays green while somebody's workload dies. Start at `1`, and only raise it when you know the real footprint of everything sharing the card.

**Interception constrains your application images.** The library runs in the container's address space. That is why HAMi's AMD support carries a glibc ≥ 2.34 floor that rules out Alpine and older base images. A platform capability imposing a constraint on application images is unusual, and it is worth telling application teams before you promise them GPU sharing.

Compute limits are softer than memory. `gpucores` bounds SM time, but neighbours on one card still contend for bandwidth and cache. Two pods that fit on paper can still be slower than either would be alone.

## Why two device plugins fight

Kubelet's device-plugin registry is keyed by resource name and is last-writer-wins. Both `nvidia-device-plugin` and HAMi register `nvidia.com/gpu`. Run both on one node and each re-registration displaces the other, so advertised capacity flaps between 1 and 10 while the scheduler makes decisions against whichever it last saw.

Two things follow, and they shape every procedure in the repo.

The conflict is **per node, not per cluster**. Two DaemonSets with disjoint node selectors never interact. That is why migration is a node-at-a-time relabel rather than a big-bang cutover, and why there is never a window with no GPUs anywhere. [Part 4](04-migrating-both-directions.md) is built entirely on this.

Different vendors **never conflict at all**. `amd.com/gpu` and `nvidia.com/gpu` are different keys. The only cross-vendor collision is HAMi's AMD backend against the AMD GPU Operator's own plugin, which is why HAMi's AMD guide tells you to disable the operator's. More in [part 5](05-labels-are-lying.md).

## What it costs

Worth stating plainly, because "share the GPU" sounds free:

- A scheduler and a webhook sit in the critical path of every GPU pod in the cluster.
- Your application images pick up a constraint they did not have.
- Contention shows up as latency variance, not as a scheduling error, so it is harder to attribute.
- Changing your mind in either direction is a migration, because `schedulerName` is immutable and every GPU workload has to be recreated.

On a cluster where GPUs are already saturated by jobs that legitimately want whole cards, none of that buys you anything. The win is concentrated where cards sit idle inside exclusive claims — which, for us, was most of the time.

---

Next: [The install that quietly broke every GPU pod](02-the-install-that-broke-everything.md) — what happens when you install with chart defaults on a cluster that is already using its GPUs.
