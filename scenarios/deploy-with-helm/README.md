# Deploy HAMi with Helm

The baseline install, and the reference for the values every other scenario reuses. Read [`../../docs/prerequisites.md`](../../docs/prerequisites.md) first — most first-install failures are node enablement, not chart configuration.

If you've not used HAMi before, [`../../docs/tutorial.md`](../../docs/tutorial.md) walks the same ground on a scratch cluster with nothing to decide, and takes about 20 minutes. This page assumes you know what you want.

This assumes a cluster with **no** GPU device plugin currently running. If `nvidia-device-plugin` is already serving GPUs to real workloads, use [`../migrate-nvidia-to-hami/`](../migrate-nvidia-to-hami) instead: the two plugins cannot share a node, and installing on top of a running one gives you flapping capacity rather than a clean cutover.

## Install

```sh
helm repo add hami-charts https://project-hami.github.io/HAMi/
helm repo update hami-charts

helm install hami hami-charts/hami \
  -n kube-system --version 2.9.0 \
  -f values.yaml
```

[`values.yaml`](values.yaml) carries the settings that matter. Two of them are the difference between working and not:

**`devicePlugin.runtimeClassName: nvidia`** — without it the plugin runs under `runc`, can't reach NVML, and crash-loops. The chart default is empty.

**`devicePlugin.nvidiaNodeSelector`** — the chart defaults to `gpu: "on"`. Whatever you choose, verify what you actually rendered, because Helm **deep-merges maps**: setting this key does not replace the default, it adds to it, and you end up with a DaemonSet requiring *both* labels. That fails in the confusing direction — it schedules fine on nodes carrying both, so it looks correct until one doesn't. Remove the default explicitly with `gpu: null`, then confirm:

```sh
helm template hami hami-charts/hami --version 2.9.0 -f values.yaml \
  | grep -A4 'nodeSelector:'
```

This `gpu: null` idiom is specific to the CLI. In [the Rancher UI](../deploy-from-rancher-apps/) you replace the `gpu: 'on'` line outright instead — Rancher already nulls anything you remove, which is its own hazard.

## Label the GPU nodes

The DaemonSet won't schedule anywhere until nodes match the selector:

```sh
kubectl label node <node> gpu-plugin=hami --overwrite
```

Using a dedicated label rather than something like `gpu=on` costs nothing now and is what makes a later migration a one-label operation instead of a chart edit.

`gpu-plugin` is the right axis when HAMi is the only vendor in play. Multi-vendor clusters add a second, `gpu-vendor`; both conventions are defined in [`../../docs/reference.md`](../../docs/reference.md#node-labels-this-repo-uses).

## Verify

```sh
# The plugin should be Running, not CrashLoopBackOff.
kubectl -n kube-system get pods -o wide | grep hami-device-plugin

# Registration lives in an annotation — NOT in allocatable. See prerequisites.
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
# [{"id":"GPU-...","count":10,"devmem":6144,"devcore":100,"type":"NVIDIA GeForce RTX 2060...","health":true}]

# Capacity = physical GPUs x deviceSplitCount.
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
# 10
```

Then prove enforcement rather than assuming it:

```sh
kubectl apply -f ../../samples/vgpu-smoke-test.yaml
kubectl logs vgpu-smoke-test
```

You want the container to report its **limit**, not the card's real memory:

```text
name, memory.total [MiB]
NVIDIA GeForce RTX 2060 with Max-Q Design, 1024 MiB
```

And for the actual point of the exercise:

```sh
kubectl apply -f ../../samples/two-pods-one-gpu.yaml
kubectl get pods -o wide | grep vgpu-share    # both Running, same node
```

## Uninstall

```sh
helm uninstall hami -n kube-system
```

The chart owns 18 objects and no CRDs, so that's complete — with one exception. HAMi writes registration data to node annotations that `helm uninstall` leaves behind:

```sh
for n in $(kubectl get nodes -o name); do
  kubectl annotate "$n" hami.io/node-nvidia-register- hami.io/node-handshake- 2>/dev/null
done
```

They're harmless but stale, and they'll confuse the next person reading the node.

## Validated

Installed this way on RKE2 (SL Micro, 6 GB Turing cards) with `deviceSplitCount: 10`. Capacity went from `nvidia.com/gpu: 1` to `10` per node. A pod limited to `nvidia.com/gpumem: 1024` reported `0MiB / 1024MiB` from `nvidia-smi` against a physical 6144 MiB card, with `HAMI-core` interception messages in the pod log. Two pods with different limits ran concurrently on one card.
