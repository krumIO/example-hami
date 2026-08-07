# Deploy HAMi from Rancher Apps

The same install as [`../deploy-with-helm/`](../deploy-with-helm), driven through Rancher instead of the CLI. Read that scenario first for what the values mean and why two of them are load-bearing.

**The one rule that matters here:** Rancher's values editor is **not** a Helm `-f values.yaml`. Edit the pre-populated YAML *in place*; never replace it. Section 2 explains why, and why getting this wrong produces an install failure whose error message points at the chart rather than at you.

Walked end to end on Rancher **v2.14.3** against an RKE2 cluster; screenshots in [`images/`](images) are from that run.

## 1. Add the chart repository

*Apps* → *Repositories* → *Create*.

![Repositories list](images/01-repositories.png)

Pick the **Helm Repository** target card ("HTTP(S) URL pointing to a Helm chart repository index"), then fill in:

| Field | Value |
| --- | --- |
| Name | `hami` |
| Index URL | `https://project-hami.github.io/HAMi/` |

![Create repository](images/02-create-repo.png)

Or declaratively, which is preferable across more than one cluster — a `ClusterRepo` is what the UI creates anyway:

```sh
kubectl apply -f clusterrepo.yaml
```

Confirm it indexed before moving on; the chart won't appear until it has:

```sh
kubectl get clusterrepo hami -o jsonpath='{.status.conditions}' | grep -o 'Downloaded'
```

## 2. Install the chart

*Apps* → *Charts*. **Search the catalogue for `hami`** rather than filtering by repository — the chart is listed lowercase `hami`, described as "Heterogeneous AI Computing Virtualization Middleware".

![Chart search](images/05-chart-search.png)

Don't try to deep-link with `?repo=hami` in the URL; that filters to zero results even with every repository checked. Loading `/apps/charts` plain and searching works.

Open the chart and press **Install this version** (defaults to the newest, 2.9.0).

![Chart detail](images/06-chart-detail.png)

### Step 1 — metadata

Two fields, neither of which defaults usefully:

- **Namespace** — defaults to `default`. Type `kube-system` and **press Enter to commit the combobox**; typing alone leaves it unselected and the app lands in the wrong namespace.
- **Name** — set `hami`.

![Install step 1](images/07-install-step1.png)

The namespace is committed when the text turns solid rather than grey:

![Namespace selected](images/07b-namespace-selected.png)

### Step 2 — values

The wizard opens **already in Edit YAML** — there is no form-to-YAML toggle to switch, only *Edit YAML* / *Compare Changes* tabs — and the editor is **pre-filled with every chart default**.

![Install step 2](images/08-install-step2-values.png)

**Edit the values you need in place. Do not select-all and paste your own values file.**

Rancher does not pass this content to Helm as an overlay. It diffs the editor against the chart defaults and writes an explicit **`null`** for every key you removed. Replace the ~200 pre-filled lines with a 14-line values file and Rancher generates 76 explicit nulls, wiping out chart defaults you never intended to touch. The install then fails like this:

```text
Error: INSTALLATION FAILED: template: hami/templates/scheduler/deployment.yaml:151:59:
  executing "hami/templates/scheduler/deployment.yaml"
  at <.Values.scheduler.metricsBindAddress>: wrong type for value; expected string; got interface {}
```

Nothing in that message suggests the UI caused it — `scheduler.metricsBindAddress` was simply nulled out from under the template. The identical minimal values file installs cleanly via `helm install` from the CLI, because the CLI merges where Rancher nulls.

So make three targeted edits:

| Find | Change to | Why |
| --- | --- | --- |
| `runtimeClassName: ''` | `runtimeClassName: nvidia` | without it the plugin can't reach NVML and crash-loops |
| `gpu: 'on'` (under `nvidiaNodeSelector`) | your own label, e.g. `gpu-plugin: hami` | so the plugin only lands on nodes you chose |
| `enabled: true` (under `admissionWebhook`, *not* `customURL`) | `enabled: false` **during a phased rollout** | see below |

![Values edited in place](images/09-values-edited.png)

Note the `gpu: null` trick from [the Helm scenario](../deploy-with-helm/) is **not** used here. There you add a null to remove the chart's default key; here you replace the line outright, because nulling removed keys is already what Rancher does.

**On the webhook:** leaving it enabled turns on cluster-wide `schedulerName` rewriting the moment the app reports *Deployed*. If any GPU node is still served by `nvidia-device-plugin`, its pods get redirected to `hami-scheduler` and stop scheduling. Set `enabled: false` until every GPU node has migrated — see [`../migrate-nvidia-to-hami/`](../migrate-nvidia-to-hami).

## 3. Label the GPU nodes

Nothing schedules until a node matches the selector. In the UI: *Nodes* → select node → *Edit Config* → *Labels*. Or:

```sh
kubectl label node <node> gpu-plugin=hami --overwrite
```

## 4. Verify

Rancher shows the app *Deployed* as soon as Helm returns, which happens well before the device plugin is working — a crash-looping DaemonSet does not make the app red. Check the real signals from *Kubectl Shell*:

```sh
kubectl -n kube-system get pods -o wide | grep hami-device-plugin   # Running, not restarting
kubectl get node <node> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

Then prove enforcement with [`../../samples/vgpu-smoke-test.yaml`](../../samples) — the container should report its limit, not the card's real memory.

## Uninstalling

*Apps* → *Installed Apps* → **hami** → *Delete*.

**Deleting the `App` object with `kubectl` is not equivalent.** Doing so removes Rancher's record but leaves the Helm release running — `helm list -n kube-system` still shows it, and the workloads keep going. If you end up in that state:

```sh
helm uninstall hami -n kube-system
```

Either way, the node registration annotations survive and need removing by hand:

```sh
kubectl annotate node <node> hami.io/node-nvidia-register- hami.io/node-handshake-
```

## When not to use this path

If GPUs are already in use on the cluster, the UI has the same hazard as the CLI with less visibility — the webhook starts redirecting every GPU pod the moment the app reports *Deployed*. Use [`../migrate-nvidia-to-hami/`](../migrate-nvidia-to-hami), which sequences it deliberately.

For fleets, [`../deploy-with-fleet/`](../deploy-with-fleet) is the better fit — a `ClusterRepo` plus a per-cluster app install is a lot of clicking compared to one bundle targeting a label.

## Validated

Walked end to end on Rancher v2.14.3 / RKE2 / SL Micro, 2026-08-07, installing HAMi 2.9.0 onto one GPU node and rolling it back. Confirmed:

- The wholesale-paste failure above is real and reproducible — it is how this scenario was first written, and it failed on the first attempt. The generated values file carried 76 explicit nulls.
- Editing in place installed cleanly: scheduler `1/1`, device-plugin DaemonSet at `0` desired until a node was labelled.
- Labelling one node swapped plugins in **seconds** — the sibling `nvidia-device-plugin` stepped aside on its own via a `NotIn` affinity on the same label.
- **Fleet did not need pausing.** With no HAMi bundle in git and the plugin label absent from the machine-pool spec, GitOps never contended with the hand-driven install.
- A pod limited to `nvidia.com/gpumem: 1024` reported `1024 MiB` from `nvidia-smi` against a physical 6144 MiB card, with `HAMI-core` interception messages in its log.
- Teardown restored `nvidia.com/gpu: 1` per node with no leftovers, once the orphaned release and the node annotations were cleaned up by hand.

Not covered: the UI label path in step 3 (labels were applied with `kubectl`), and the UI *Delete* path in the uninstall section.
