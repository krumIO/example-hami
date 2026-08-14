# Migrating a live cluster to HAMi, and back

*Part 4 of [a seven-part field report](../blog.md). Both directions, on a cluster with workloads running, in a single afternoon.*

The clean-install case is the easy one. The real question is what you do on a cluster where GPUs are already in use and `nvidia-device-plugin` is already serving them — and, just as importantly, how you get back out if you change your mind.

We ran both directions on `dev-machines`. Three GPU nodes, HAMi 2.9.0, forward and back, then wrote the runbooks from what actually happened rather than what we expected.

## The whole design rests on one fact

Both plugins register the resource name `nvidia.com/gpu` with kubelet, and kubelet's device-plugin registry is last-writer-wins. So they cannot share a node.

But they conflict **per node, not per cluster**. Two DaemonSets with disjoint node selectors never interact at all. That single fact is what makes this safe: put both plugins behind the same label with different values, and migration becomes one `kubectl label` at a time.

```text
gpu-plugin=nvdp   ->  nvidia-device-plugin
gpu-plugin=hami   ->  HAMi
```

A node with neither label runs no plugin and advertises no GPUs, which is the correct safe intermediate state. Rollback is the same command with the other value. There is never a moment where the cluster has no GPUs.

It also sidesteps a GitOps problem. Fleet cannot express "uninstall A, then install B" as an ordered transaction — `dependsOn` gates on a bundle being *Ready*, not on another being *gone*. Model the migration as delete-one-bundle-add-another in a single commit and you get an uncontrolled window with no device plugin at all. With the label partition, both bundles stay deployed the whole time and no ordering is involved.

That worked in practice better than we expected. We gave the stock plugin a `NotIn [hami]` affinity on the label, and when we relabelled a node it stepped aside on its own:

```text
nvdp-on-node=0 | hami=2/2 Running | allocatable gpu=10
```

Seconds, not minutes. And Fleet never needed pausing, which had been our main worry going in.

## A full drain is the wrong tool

Our first runbook said to `kubectl drain` each node before swapping plugins. That is the reflex, and on this cluster it was wrong.

Draining evicts everything, including the Longhorn instance-manager. With `node-drain-policy: block-for-eviction`, that blocks until every storage replica has rebuilt elsewhere — roughly an hour per node, for storage movement that a device-plugin swap does not need.

You only need the GPU consumers off the node:

```sh
kubectl cordon "$NODE"

kubectl get pods -A --field-selector "spec.nodeName=$NODE" -o json | jq -r '
  .items[] | select([.spec.containers[].resources.limits // {} | keys[]] | any(startswith("nvidia.com/")))
  | "\(.metadata.namespace) \(.metadata.name)"' | while read ns p; do kubectl -n "$ns" delete pod "$p"; done

kubectl label node "$NODE" gpu-plugin=hami --overwrite
```

The swap itself takes seconds. The old plugin's pod goes, the new one arrives, kubelet re-registers the resource.

## The hour we lost to something else entirely

We did try the full drain first, and it deadlocked in a way worth recording because it had nothing to do with GPUs.

The drain blocked on Longhorn replica eviction, which is expected. What was not expected: eviction could not proceed, because of the three storage nodes, one had a Longhorn disk sitting at **0 GiB capacity** while reporting `Ready=True` and `Schedulable=True` at the node level.

```text
disk default-disk-...  Ready=False  DiskFilesystemChanged
                       Schedulable=False  DiskNotReady
```

That node had been reflashed a day earlier. Its Kubernetes Node object was hours old; Longhorn's node record was two months old and still held the *previous* filesystem's `diskUUID`. Longhorn correctly refused to adopt a disk whose UUID did not match its records — a safety feature working as designed, with no self-healing path.

So: one usable storage node for 18 two-replica volumes, no eviction target, drain blocked forever. Repairing the stale disk record (disable, remove, re-add) let eviction start immediately and the whole thing unblocked.

Then the node died mid-drain and the instance-manager PDB became permanently unsatisfiable, which needed a force-delete of one stranded pod.

None of this is HAMi's fault. It is what happens when you reach for a heavyweight operation on a cluster whose storage layer has an unrelated latent problem. The lesson we took: reach for the narrowest operation that achieves the goal. The narrow one would have skipped all of it.

## Coming back

The reverse direction has one hazard the forward direction does not: workloads may be requesting resources that are about to stop existing. A pod asking for `nvidia.com/gpumem` becomes unschedulable the moment HAMi leaves, and the cluster tells you nothing beyond a generic "0/N nodes available".

The order that matters:

1. Find workloads requesting HAMi-only resources and rewrite them to whole GPUs. Check you have the physical cards to land them — pods that shared one card now need one each. This is the step that turns a rollback into an incident if skipped.
2. Neutralize the webhook **first**, before touching any plugin, so anything that restarts during the migration comes back on `default-scheduler`.
3. Bring the stock plugin back, selecting nothing.
4. Move nodes back one at a time.
5. Uninstall HAMi.
6. Clean the node annotations — `helm uninstall` does not remove them.
7. Recreate the workloads.

Capacity came back within seconds of each relabel.

## The step we skipped, and what it taught us

We skipped steps 1 and 7. Two days later we found this:

```text
qwen2-5-0-5b-instruct-predictor-...   Running   sched=hami-scheduler
  requests: {"nvidia.com/gpu": "1", "nvidia.com/gpucores": "100"}
```

A pod running under a scheduler that no longer existed, requesting a resource nothing advertised. Our first reaction was that this was a time bomb — it would never reschedule.

That was wrong, and the correction is the useful part.

The Deployment and ReplicaSet templates were both clean: `schedulerName: default-scheduler`, only `nvidia.com/gpu`. Nothing in the spec chain had ever asked for `gpucores`. The webhook had injected **both** fields at admission, during the roughly forty minutes it was enabled. Sibling pods from the same ReplicaSet, created before and after that window, were clean.

So it is a fossil, not a bomb. Delete it and the controller recreates it from the clean template, and the replacement schedules fine.

But this exposed two real defects in our own runbook:

**Step 0 sent people hunting for a manifest that does not exist.** It said to find pods requesting HAMi-only resources and edit their controllers. Follow that literally and you find this pod, go looking for the Deployment that declares `gpucores`, and find nothing — with no explanation. The runbook now says the webhook injects these, so a pod with no matching controller spec needs *recreating*, not editing.

**Step 6 claimed running pods were broken.** It said anything carrying `schedulerName: hami-scheduler` "is unschedulable now that the scheduler is gone". False for already-bound pods — they keep running indefinitely, which is precisely why we did not notice we had skipped the step. It now leads with "nothing looks broken", and gives the real reason to do it: until pods turn over, your cluster contains specs that cannot be reproduced from any manifest, and any controller edits you made in step 0 are not live.

The failure tables now distinguish Pending (delete them) from Running (harmless, recreate at leisure).

## What we would tell someone starting

The label partition is the whole trick. Adopt it on day one even on a single cluster, because it converts an irreversible cutover into a reversible one and costs nothing.

Verify with `helm get values` after install, not with the UI's status. And do the narrow operation: cordon and move GPU consumers, not drain.

---

Next: [Your GPU labels are lying to you](05-labels-are-lying.md) — why we could not trust node labels to find our GPU nodes in the first place.
