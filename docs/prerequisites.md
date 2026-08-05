# Prerequisites

HAMi's device plugin talks to NVML. Everything below exists to make sure it can. When it can't, the plugin crash-loops with a message that sends you to the wrong place:

```
E factory.go:135] Incompatible strategy detected auto
E factory.go:136] If this is a GPU node, did you configure the NVIDIA Container Toolkit?
E main.go:201] failed to construct resource managers: invalid device discovery strategy
```

That names the toolkit, so the natural reaction is to go check the host. The host is usually fine — the pod just isn't running under the NVIDIA runtime.

## On the node

Each GPU node needs the **NVIDIA driver** and the **container toolkit** (`nvidia-container-runtime`). How they get there is distribution-specific: baked into the OS image, installed by the GPU Operator, or a package install. On RKE2, when the toolkit is present at startup RKE2 detects it, writes the runtime into its own containerd config, and the `nvidia` handler becomes usable.

## The RuntimeClass

Most clusters leave `runc` as the default runtime and select the NVIDIA one per pod. That means a `RuntimeClass` must exist:

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

and every GPU-touching pod — **including the device plugin itself** — must name it:

```yaml
spec:
  runtimeClassName: nvidia
```

HAMi's chart leaves `devicePlugin.runtimeClassName` empty by default, which is the single most common reason a first install fails. Set it unless your cluster's default runtime is already `nvidia`.

## Verify per node, not per cluster

The RuntimeClass is a cluster-scoped object; its existence says nothing about whether a given node can honour it. A node that was recently rebuilt can carry every GPU label your tooling assigns and still have no `nvidia` runtime in containerd — the labels come from the machine pool, the runtime comes from the OS image, and they arrive at different times. The symptom is not a crash loop but a pod that never starts:

```
Failed to create pod sandbox: no runtime for "nvidia" is configured
```

Both HAMi and `nvidia-device-plugin` fail identically here, so it's worth ruling out before blaming either. The cheapest check is to actually run something:

```sh
kubectl run rt-probe --rm -it --restart=Never --image=busybox \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' \
  -- true
```

If that returns cleanly, the node can host a GPU plugin. If it hangs in `ContainerCreating` with the sandbox error, the node's NVIDIA enablement hasn't finished — fix that first.

## Know what the webhook does before you install

HAMi ships a `MutatingWebhookConfiguration` (`vgpu.hami.io`) that rewrites `schedulerName` to `hami-scheduler` on **any** pod requesting GPU resources, cluster-wide, from the moment the chart is installed. Its selectors exclude only objects labelled `hami.io/webhook: ignore`, so by default it catches everything.

Two consequences worth internalising:

It takes effect **whether or not HAMi's device plugin works**. Install the chart with a broken plugin and every GPU pod in the cluster is redirected to a scheduler that has no registered devices, so they sit `Pending` with no obvious cause. The device plugin crash-looping in `kube-system` is easy to miss; the outage it causes is cluster-wide.

Its `failurePolicy: Ignore` does **not** protect you. That governs what happens when the webhook is unreachable, not when it succeeds. A successful mutation into a broken scheduler is still a successful mutation.

During a phased rollout, either disable it (`scheduler.admissionWebhook.enabled: false`) and set `schedulerName: hami-scheduler` explicitly on migrated workloads, or leave it on and label the namespaces that aren't ready yet:

```sh
kubectl label namespace <ns> hami.io/webhook=ignore --overwrite
```

## One more thing that will confuse you

`nvidia.com/gpumem` and `nvidia.com/gpucores` never appear in a node's `allocatable`. The device plugin advertises only the multiplied `nvidia.com/gpu` count; memory and core accounting live in a node annotation that HAMi's scheduler-extender reads:

```sh
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
# [{"id":"GPU-...","count":10,"devmem":6144,"devcore":100,"type":"NVIDIA GeForce RTX 2060...","health":true}]
```

Checking `allocatable` for `gpumem` and finding it missing will convince you a working install is broken. Check the annotation instead.
