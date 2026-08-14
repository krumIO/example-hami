# The install that quietly broke every GPU pod

*Part 2 of [a five-part field report](../blog.md). [Part 1](01-what-hami-does.md) covers the three components; this post is about what happens when one of them starts working before the others.*

Somebody installed HAMi on `dev-machines` with `helm install` and chart defaults. Helm reported success. The app showed as deployed. A week later we came back to look at something unrelated and found that GPU scheduling had been broken the entire time.

Nothing had alerted. Nothing was red.

## What we found

Two GPU pods sitting `Pending`, and a DaemonSet nobody had looked at:

```text
kserve-models   qwen2-5-0-5b-instruct-predictor-...   Pending   sched=hami-scheduler
kserve-models   qwen2-5-3b-instruct-predictor-...     Pending   sched=hami-scheduler

kube-system     hami-device-plugin   3 desired   0 ready   CrashLoopBackOff (80 restarts)
```

Those pods never asked for `hami-scheduler`. HAMi's admission webhook had rewritten `schedulerName` on them at admission, as designed. The problem was that it was routing them to a scheduler whose device registry was empty, because the device plugin had never once started successfully.

This is the failure mode worth internalising: **the webhook takes effect whether or not anything else works.** It is installed with the chart. It starts mutating immediately. It does not check that the device plugin is healthy, because that is not its job. So a broken plugin plus a working webhook equals cluster-wide GPU scheduling failure, with a healthy-looking Helm release sitting on top of it.

And `failurePolicy: Ignore` does not save you. That governs what happens when the webhook is *unreachable*. A successful mutation into a broken scheduler is still a successful mutation.

## The error message points at the wrong thing

Here is what the device plugin was saying, on loop, for a week:

```text
E factory.go:135] Incompatible strategy detected auto
E factory.go:136] If this is a GPU node, did you configure the NVIDIA Container Toolkit?
E factory.go:137] You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites
E main.go:201] error starting plugins: error getting plugins: unable to create plugins: failed to construct resource managers: invalid device discovery strategy
```

It asks whether you configured the NVIDIA container toolkit. The toolkit was fine. It was baked into the node image and the stock `nvidia-device-plugin` had been running happily on the same nodes for weeks.

The actual cause was one empty string. Compare the two DaemonSets:

| | working `nvdp` | crash-looping `hami` |
| --- | --- | --- |
| `runtimeClassName` | `nvidia` | *unset* |
| nodeSelector | `krum.io/gpu-class=turing-open` | `gpu=on` |

HAMi's device plugin reads the GPU through NVML, which is only reachable under the NVIDIA container runtime. On this cluster — following SUSE's guidance — the default runtime is `runc` and the NVIDIA one is selected per pod via `runtimeClassName`. The chart ships `devicePlugin.runtimeClassName: ""`, so the plugin ran under `runc`, could not reach NVML, and died. Every 20 seconds, for seven days.

The install had been done with no values at all. Pure chart defaults. Which on this cluster means: a device plugin that cannot work, and a webhook that will happily redirect everything anyway.

## Two failures that look similar and are not

There is a second, genuinely node-level version of this failure, and telling them apart saves an afternoon.

The one above is a crash loop: the container starts, tries to reach NVML, and exits. The pod is on the node, the node is fine, the pod is misconfigured.

This is the other one:

```text
Failed to create pod sandbox: rpc error: code = Unknown desc = unable to get OCI runtime
for sandbox "...": no runtime for "nvidia" is configured
```

That appears at sandbox creation, before the container starts, and it means the node genuinely has no `nvidia` runtime in containerd. We hit it on a node that had been reflashed hours earlier: it carried every GPU label the machine pool assigns, but its NVIDIA OS transition had not finished. Neither HAMi nor the stock plugin can run there. It is not a HAMi problem at all.

The tell is the phase. `CrashLoopBackOff` means your pod spec. `ContainerCreating` with that sandbox error means the node.

## What we changed

Three things, in the order that matters.

**Set `runtimeClassName` explicitly.** Unless a pod with no `runtimeClassName` at all can already see a GPU on your cluster, set it:

```yaml
devicePlugin:
  runtimeClassName: nvidia
  createRuntimeClass: false   # the cluster owns that object
```

**Decide the webhook before installing, not after.** During any phased rollout — meaning any cluster where some GPU nodes will still be served by the stock plugin — turn it off and name the scheduler explicitly on the workloads you move:

```yaml
scheduler:
  admissionWebhook:
    enabled: false
```

Turn it on at the end, once every GPU node runs HAMi. The alternative, if you are doing the whole cluster in one sitting, is to leave it on and label the namespaces that are not ready yet with `hami.io/webhook=ignore`.

**Verify per node, not per cluster.** `RuntimeClass` is cluster-scoped, so its existence tells you nothing about whether a given node can honour it. Actually run something:

```sh
kubectl run rt-probe --rm -it --restart=Never --image=busybox \
  --overrides='{"spec":{"runtimeClassName":"nvidia","nodeSelector":{"kubernetes.io/hostname":"<node>"}}}' \
  -- true
```

A clean exit means that node can host a GPU plugin. It costs seconds and rules out the failure that looks most like a HAMi bug.

## Recovering, if you are in this state now

The webhook is the outage. Remove it first:

```sh
helm uninstall hami -n kube-system
kubectl get mutatingwebhookconfigurations | grep -i hami   # confirm it is gone

# schedulerName is immutable, so pinned pods must be recreated, not patched
kubectl -n <ns> delete pod <pod>
```

Their controllers recreate them with `default-scheduler` and they schedule against whole GPUs again. That took us about two minutes once we understood what we were looking at, having spent considerably longer getting there.

## The part that stayed broken

One detail we did not notice at the time, and only found two days later: a pod admitted during the window when the webhook was live kept running afterwards, carrying `schedulerName: hami-scheduler` and an injected `nvidia.com/gpucores: 100` that no controller spec had ever declared.

It kept serving traffic the whole time. Kubernetes does not re-verify scheduling or resources for an already-bound pod. That story, and why our own runbook told us to look for the wrong thing, is in [part 4](04-migrating-both-directions.md).

---

Next: [Rancher's values editor is not `helm -f values.yaml`](03-rancher-values-editor.md) — where the same install goes wrong for a completely different reason.
