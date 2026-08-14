# Sharing a GPU on Kubernetes: a five-part field report

We spent a couple of weeks putting [HAMi](https://github.com/Project-HAMi/HAMi) onto a real Rancher-managed RKE2 cluster, breaking it, fixing it, migrating to it, migrating back off it, and writing down everything that surprised us. This is that write-up.

It is not a getting-started guide. The [repo this lives in](.) has those, and they are the thing to follow if you want to actually do this. These posts are the reasoning behind them: why the design is what it is, which failures cost us a day, and which advice we had to retract after testing it.

The short version, if you read nothing else: HAMi works, the enforcement is real, and the two things most likely to hurt you are an admission webhook that takes effect before the thing it depends on, and a values editor that does not mean what you think it means.

## The series

**1. [What HAMi actually does, and what it costs](blog/01-what-hami-does.md)**
Three components, not one. What "sharing" means when the limit is enforced by a library injected into your container, why that puts a constraint on your application images, and an honest list of what you give up. Start here — the rest of the series assumes this model.

**2. [The install that quietly broke every GPU pod](blog/02-the-install-that-broke-everything.md)**
Installing with chart defaults took GPU scheduling down cluster-wide on a live cluster, while Helm reported success. The error message named the NVIDIA container toolkit, which was fine. The real causes were a missing `runtimeClassName` and a webhook that does its job whether or not anything else works.

**3. [Rancher's values editor is not `helm -f values.yaml`](blog/03-rancher-values-editor.md)**
The most surprising finding, and the one we have not seen documented anywhere: Rancher diffs your edited YAML against the chart defaults and sends an explicit `null` for every key you removed. Paste a tidy 14-line values file over the pre-filled 303 and you generate 76 nulls, one of which fails the install with an error pointing at the chart. Plus two duplicate-key traps in HAMi's own values.

**4. [Migrating a live cluster to HAMi, and back](blog/04-migrating-both-directions.md)**
Both directions on a cluster with running workloads, with no window where GPUs vanish. Why the conflict is per-node rather than cluster-wide, why a full `kubectl drain` is the wrong tool, an hour-long deadlock that had nothing to do with GPUs, and the fossil pods we found two days later because we skipped a step in our own runbook.

**5. [Your GPU labels are lying to you](blog/05-labels-are-lying.md)**
Every node in our GPU cluster reported itself as AMD or Intel. All the GPUs we care about are NVIDIA. What autodiscovery actually describes on a hybrid machine, how that interacts with vendor-specific device plugins, and where AMD and Intel really stand in HAMi today (one is further along than the docs suggest; the other is further behind).

## What was actually tested

Everything in these posts happened on `dev-machines`: RKE2 on SUSE Linux Micro, managed by Rancher v2.14.3, three GPU worker nodes with 6 GB Turing cards, HAMi 2.9.0. Dates are in the posts where they matter.

Two things we did **not** test, stated up front because the posts would otherwise imply more than we know: a production inference workload on a shared card (sharing was proven with CUDA base images), and HAMi's AMD backend on real Instinct hardware.

Where we got something wrong and later corrected it, the posts say so. There is a fair amount of that, and it is the most useful part.
