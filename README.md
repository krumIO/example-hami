# example-hami

Share one physical GPU across several pods on Kubernetes, using [HAMi](https://github.com/Project-HAMi/HAMi). Each pod gets its own memory and compute slice, enforced inside the container — `nvidia-smi` in a pod limited to 1 GiB reports 1 GiB, not the card's real 6 GiB — so a single card can host several small model servers or notebooks instead of sitting idle behind one exclusive claim.

This repo is Kubernetes-focused: plain manifests, Helm, Rancher, and Fleet. Nothing here assumes a PaaS layer.

> Status: validated on RKE2 + SL Micro with 6 GB Turing cards, under Rancher. The deployment shapes and both migration directions were executed end to end; what was *not* exercised is a production inference workload on a shared card (see each scenario's Validated section for exactly what was proven).

## What HAMi changes

Without it, `nvidia-device-plugin` advertises one `nvidia.com/gpu` per card and a pod either gets the whole thing or waits. HAMi replaces that plugin and advertises `deviceSplitCount` virtual devices per card, plus two resources the stock plugin has no concept of:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1        # one slice
    nvidia.com/gpumem: 2048  # MiB, enforced in-container
    nvidia.com/gpucores: 30  # percent of SM time
```

It also runs its own scheduler and a mutating webhook that routes GPU pods to it. That webhook is the part most likely to surprise you — read [`docs/prerequisites.md`](docs/prerequisites.md) before installing anything.

## New to HAMi? Start with the tutorial

[`docs/tutorial.md`](docs/tutorial.md) — about 20 minutes on a scratch cluster, ending with two pods running on one physical GPU with different enforced memory limits. It installs the stock plugin first so you can see the limitation before you fix it, and puts everything back at the end. No decisions to make.

## Docs

| | |
| --- | --- |
| [`docs/tutorial.md`](docs/tutorial.md) | learn by doing: one GPU, two pods, start to finish |
| [`docs/prerequisites.md`](docs/prerequisites.md) | what must be true before installing, and how to verify it |
| [`docs/how-hami-works.md`](docs/how-hami-works.md) | the mental model — three components, what enforcement means, why plugins collide, what it costs |
| [`docs/reference.md`](docs/reference.md) | lookup: vendor/version matrix, resource names, annotations, label conventions, chart values, failure signatures |

## Set up the node prerequisites first

Every scenario needs the NVIDIA driver and container toolkit on the GPU nodes, and — on most clusters — pods that name a `runtimeClassName`. See [`docs/prerequisites.md`](docs/prerequisites.md). Skipping this produces a crash loop whose error message blames the host toolkit even when the host is fine.

**Check HAMi covers your accelerators before anything else.** It virtualizes NVIDIA GPUs in the current release (**v2.9.0**), added AMD on `master` after that release with no target version published yet, and has no Intel support beyond a [roadmap entry](https://project-hami.io/docs/next/contributor/roadmap#planned-features). Mixed-vendor clusters are fine; the trap is node selection. See [`scenarios/mixed-gpu-vendors/`](scenarios/mixed-gpu-vendors).

## Pick a scenario

**Installing:**

- [`scenarios/deploy-with-helm/`](scenarios/deploy-with-helm) — the baseline. Start here even if you intend to use one of the others; the values are the same everywhere and this is where they're explained.
- [`scenarios/deploy-from-rancher-apps/`](scenarios/deploy-from-rancher-apps) — the same install through the Rancher UI, plus the `ClusterRepo` CR if you'd rather not click.
- [`scenarios/deploy-with-fleet/`](scenarios/deploy-with-fleet) — GitOps delivery to one or many downstream clusters.

**Migrating a cluster that already has GPUs in use:**

- [`scenarios/migrate-nvidia-to-hami/`](scenarios/migrate-nvidia-to-hami) — cutting over from `nvidia-device-plugin` without a window where GPUs vanish cluster-wide.
- [`scenarios/migrate-hami-to-nvidia/`](scenarios/migrate-hami-to-nvidia) — going back, and the emergency procedure if HAMi has stopped GPU scheduling.

**More than one kind of accelerator:**

- [`scenarios/mixed-gpu-vendors/`](scenarios/mixed-gpu-vendors) — NVIDIA alongside AMD or Intel, and the hybrid discrete-plus-integrated node that mislabels itself.

The two migration scenarios are a pair: don't run the first on a cluster that matters until you've read the second, because the rollback determines how much risk the cutover actually carries.

## Layout

```text
docs/tutorial.md                       # 20-minute guided first run, self-contained
docs/prerequisites.md                  # driver, toolkit, RuntimeClass, per-node verification
docs/how-hami-works.md                 # explanation: components, enforcement, trade-offs
docs/reference.md                      # lookup tables: vendors, values, labels, annotations
samples/vgpu-smoke-test.yaml           # one pod: proves the memory limit is enforced
samples/two-pods-one-gpu.yaml          # two pods co-resident on a single card
scenarios/deploy-with-helm/            # helm install + the values that matter
scenarios/deploy-from-rancher-apps/    # Rancher UI / ClusterRepo
scenarios/deploy-with-fleet/           # Fleet bundle
scenarios/migrate-nvidia-to-hami/      # cutover, node by node
scenarios/migrate-hami-to-nvidia/      # rollback and emergency recovery
scenarios/mixed-gpu-vendors/           # NVIDIA + AMD/Intel, and hybrid nodes
```

## Sizing note

`deviceSplitCount` is how many pods may share a card; `deviceMemoryScaling` lets you advertise more memory than physically exists. Oversubscribing memory turns a scheduling problem into an OOM inside somebody's container, where it looks like an application bug. Start at `deviceMemoryScaling: 1` and only raise it once you know the real footprint of the workloads sharing the card — especially on small cards, where two 3 GB models on a 6 GB card leaves nothing for CUDA context.
