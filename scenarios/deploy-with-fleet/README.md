# Deploy HAMi with Fleet

GitOps delivery of the same chart and values, to one cluster or to every cluster carrying a label. Read [`../deploy-with-helm/`](../deploy-with-helm) first for what the values do.

Fleet is only the delivery mechanism here. Everything it applies is reproducible with `helm install` — which matters most during a migration, when you want to be certain you can act without waiting on a reconcile loop.

## The bundle

```text
fleet.yaml     # bundle definition: chart coordinates, namespace, targets
values.yaml    # identical to the helm scenario's values
```

Drop both into a directory in your Fleet git repo, inside a path your `GitRepo` already watches.

```sh
cp -r . <fleet-repo>/<cluster>/platform/hami/
```

`fleet.yaml` targets `kube-system` and pins chart version 2.9.0. Adjust `targets` if you're delivering to a subset of clusters; with no `targets` block the bundle goes everywhere the parent `GitRepo` points.

## Node labels stay out of git

The bundle installs the chart. It does **not** manage the `gpu-plugin` node labels, and that's deliberate.

Those labels are the control surface for moving a node between plugins, and during a migration you want to move one node, look at it, and decide. If a chart or a machine-pool template also asserts them, you end up fighting your own GitOps mid-cutover — you relabel a node, the reconcile loop puts it back, and the node flaps between two device plugins. Set them by hand:

```sh
kubectl label node <node> gpu-plugin=hami --overwrite
```

Once the topology is stable and you're no longer migrating, moving them into the machine pool is reasonable. Not before.

## Verify

Fleet reporting the bundle `Ready` means Helm succeeded, which happens well before the device plugin works — a crash-looping DaemonSet does not make a bundle un-Ready. Check the plugin and the node annotation directly:

```sh
kubectl -n kube-system get ds hami-device-plugin
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
```

If the bundle sits `Modified` forever, HAMi's controllers are stamping something post-apply; `diff.comparePatches` on the specific path is the fix, not `takeOwnership`.

## What Fleet cannot do for you

**It has no ordered uninstall-then-install.** `dependsOn` gates on a bundle becoming Ready, not on another bundle being gone. If you model a migration as "delete the nvidia-device-plugin bundle and add the HAMi bundle in one commit", the two happen concurrently and you get an uncontrolled window with no device plugin at all — every GPU pod unschedulable until it resolves.

The migration scenarios avoid this entirely by keeping **both bundles deployed** with disjoint node selectors, so the cutover is a node label change and no bundle ordering is involved. That's the main reason the label partition is worth adopting even on a single cluster.

**It won't sequence the webhook for you either.** Enabling `scheduler.admissionWebhook.enabled` in a commit takes effect cluster-wide on the next reconcile, including on nodes that haven't migrated yet. Treat that flag as a migration step with its own commit, not a config detail.

## Validated

Delivered this way to an RKE2 cluster under Rancher, including a full cutover from `nvidia-device-plugin` and back. Fleet applied the chart and the sibling plugin's node affinity; every plugin swap was driven by `kubectl label` and took seconds. See [`../migrate-nvidia-to-hami/`](../migrate-nvidia-to-hami).
