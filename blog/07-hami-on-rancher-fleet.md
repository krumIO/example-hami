# HAMi on Rancher with Fleet

*Part 7 of [a field report](../blog.md), and the companion to [part 6](06-hami-on-rancher-manual.md), which does the same install by clicking.*

Fleet is the obvious answer once you have more than one GPU cluster. The bundle itself is unremarkable — chart coordinates and a values file, same as anything else. What makes it worth a post is that HAMi has an ordering dependency Fleet cannot express, and the workaround for that turns out to be the same design that makes migrations safe.

## The bundle

```yaml
name: hami
defaultNamespace: kube-system

labels:
  service: hami
  component: helm

helm:
  repo: https://project-hami.github.io/HAMi/
  chart: hami
  version: "2.9.0"
  releaseName: hami
  valuesFiles:
    - values.yaml

dependsOn:
  - selector:
      matchLabels:
        service: nvidia-runtimeclass
```

The values are identical to the CLI path — `runtimeClassName: nvidia`, your own node selector, `deviceSplitCount`, and a deliberate decision about the webhook. Nothing Fleet-specific.

The `dependsOn` is there because the device plugin needs `runtimeClassName: nvidia` to reach NVML, so whatever creates that RuntimeClass has to land first. Drop it if your RuntimeClass comes from the OS image or the GPU Operator rather than a sibling bundle.

Everything Fleet applies here is reproducible with `helm install`. That is deliberate, and it matters most during a migration, when you want to be certain you can act without waiting on a reconcile loop.

## The thing Fleet cannot do

Fleet has no ordered uninstall-then-install. `dependsOn` gates on a bundle becoming *Ready*, not on another bundle being *gone*.

So the intuitive way to model a migration — delete the `nvidia-device-plugin` bundle and add the `hami` bundle in one commit — gives you two bundles racing. For some window, neither plugin is registered, every GPU pod in the cluster is unschedulable, and you have no control over how long that lasts.

The fix is to stop modelling it as a bundle swap at all.

Both plugins stay deployed, permanently, with disjoint node selectors on a label you control:

```text
gpu-plugin=nvdp   ->  nvidia-device-plugin
gpu-plugin=hami   ->  HAMi
```

Now migrating a node is `kubectl label`, no bundle ordering is involved, and Fleet is never asked to sequence anything it cannot sequence. The stock plugin gets a matching anti-affinity so it steps aside on its own:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: gpu-plugin
              operator: NotIn
              values: ["hami"]
```

This is the same label partition that [part 4](04-migrating-both-directions.md) is built on. It was designed to make the per-node plugin conflict manageable; it happens to also make the GitOps problem disappear. That is usually a sign the design is right.

## Keep the node labels out of git

The bundle installs the chart. It deliberately does **not** manage the `gpu-plugin` node labels.

Those labels are the control surface for moving a node between plugins. During a migration you want to move one node, look at it, and decide. If a chart or a machine-pool template also asserts them, you end up fighting your own automation mid-cutover: you relabel a node, the reconcile loop puts it back, and the node flaps between two device plugins.

We verified this concern was real before designing around it. On `dev-machines`, `krum.io/gpu-plugin` is not in the machine-pool spec, so hand-labelling stays put. Had it been there, the whole approach would have needed rethinking.

Once the topology is stable and you are no longer migrating, moving the labels into the machine pool is reasonable. Not before.

## Fleet Ready does not mean the plugin works

The same trap as the UI's *Deployed*, for the same reason. Fleet reports a bundle `Ready` when Helm succeeded and the resources exist. A crash-looping DaemonSet does not make a bundle un-Ready.

So the check is the same as everywhere else — go to the plugin and the node annotation, not the bundle status:

```sh
kubectl -n kube-system get ds hami-device-plugin
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
```

If the bundle instead sits `Modified` forever, that is a different thing: some controller is stamping a field post-apply. Fix it with `diff.comparePatches` on the specific path rather than reaching for `takeOwnership`, which addresses a different failure entirely.

## Treat the webhook flag as its own commit

Enabling `scheduler.admissionWebhook.enabled` in a commit takes effect cluster-wide on the next reconcile, including on nodes that have not migrated yet. It is not a config detail; it is a migration step that happens to be expressed as a value.

Give it its own commit, at the end, once every GPU node runs HAMi.

## What we learned by running it

We delivered this bundle to `dev-machines` through Fleet on 2026-08-05, then later did a manual UI install on the same cluster while Fleet was still running normally. Two findings came out of that.

**Fleet applied it exactly as expected.** The scheduler came up, the device-plugin DaemonSet sat at `0` desired because no node carried the label, and the sibling plugin picked up its `NotIn` affinity. Labelling one node swapped plugins in seconds.

**Fleet did not need pausing during the manual install**, which had been our main worry going in. Three things had to be true, and we checked all three rather than assuming: no `hami` bundle existed in git at that point, `krum.io/gpu-plugin` was absent from the machine-pool spec, and the stock plugin already carried the affinity. With those, GitOps never contended with the hand-driven work.

That is worth stating because the instinct — pause Fleet before touching anything by hand — is a reasonable instinct that would have cost time and added risk. The specific reason it was unnecessary is that nothing in git claimed any of the objects being touched.

## When to prefer this over the UI

More than one cluster, straightforwardly. A `ClusterRepo` plus a per-cluster app install is a lot of clicking compared to one bundle targeting a cluster label.

Also when you want the values reviewed. The three edits that matter here — `runtimeClassName`, the node selector, the webhook flag — are exactly the ones where a mistake is quiet rather than loud, and a diff in a pull request catches things that a YAML editor at 5pm does not.

---

That is the series. The [repo](../README.md) has the bundle, the runbooks, and a [twenty-minute tutorial](../docs/tutorial.md) that ends with two pods sharing one card.
