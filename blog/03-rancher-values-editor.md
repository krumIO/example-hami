# Rancher's values editor is not `helm -f values.yaml`

*Part 3 of [a five-part field report](../blog.md). This is the finding we have not seen written down anywhere, and it will break any chart, not just HAMi.*

We wrote the Rancher UI walkthrough from the Helm one. Same chart, same values, different front end — how different could it be. The instruction we wrote was the obvious one:

> On the Values step, switch to Edit YAML and paste the contents of `values.yaml`.

Then we ran it against a real cluster, and it failed on the first attempt:

```text
Error: INSTALLATION FAILED: template: hami/templates/scheduler/deployment.yaml:151:59:
  executing "hami/templates/scheduler/deployment.yaml"
  at <.Values.scheduler.metricsBindAddress>: wrong type for value; expected string; got interface {}
```

Nothing in that message suggests the UI had anything to do with it. It reads like a chart bug. It is not.

## What Rancher actually sends

Rancher's install wizard pre-fills the YAML editor with the chart's full defaults — 303 lines for HAMi 2.9.0. We had replaced all of it with a tidy 14-line values file, the way you would with `helm -f`.

Rancher does not treat that content as an overlay. It **diffs your editor content against the chart defaults and writes an explicit `null` for every key you removed.** Here is the file it generated, pulled straight out of the operation secret:

```json
{"dcuResourceCores":null,"dcuResourceMem":null,"dcuResourceName":null,
 "devicePlugin":{"deviceCoreScaling":null,"deviceListStrategy":null,...},
 "scheduler":{...,"metricsBindAddress":null,...}}
```

Seventy-six explicit nulls. Including `scheduler.metricsBindAddress`, whose chart default is the string `":9395"` and whose template expects a string. Null is not a string. The template fails, and the error names the chart.

The proof that this is Rancher's serialization and not our values: the identical minimal file installs cleanly from the CLI.

```sh
helm template hami hami-charts/hami --version 2.9.0 -f minimal.yaml   # exit 0
```

The CLI merges. Rancher nulls. Same input, opposite outcome.

## So the rule inverts

Everything you know about `helm -f` is wrong here. The correct instruction is the opposite of what we wrote:

**Edit the pre-filled YAML in place. Never replace it.**

The check that catches a mistake is the line count. Rancher pre-fills 303 lines; after three in-place edits you should still have 303:

```text
before: 303 lines / after: 303 lines
```

If yours drops, you removed keys, and Rancher is about to null them.

One knock-on: the `gpu: null` idiom that the CLI path needs — where you add a null to remove the chart's default `nvidiaNodeSelector` key, because Helm deep-merges maps — is **not** used in the UI. There you replace the line outright, because nulling removed keys is already what Rancher does. Two paths, opposite techniques, same goal.

## Then the values fight back

Having established you must make surgical edits, the next problem is that HAMi's values file contains two pairs of near-identical keys, and in both cases only one of the pair is the one you want.

**`runtimeClassName` appears twice:**

```yaml
devicePlugin:
  ...
  runtimeClassName: ''        # line 71, 2-space indent  ← the one you want
devices:
  ascend:
    ...
    runtimeClassName: ''      # line 117, 4-space indent  ← not this
```

Edit the wrong one and the install still succeeds, while the device plugin keeps crash-looping — [exactly the failure](02-the-install-that-broke-everything.md) that setting exists to prevent. Nothing tells you. The only tell is the indentation and the parent key.

**`enabled` appears twice under `admissionWebhook`, four lines apart:**

```yaml
  admissionWebhook:
    customURL:
      enabled: false          # line 219, 6-space indent  ← already false
      host: 127.0.0.1
      path: /webhook
      port: 31998
    enabled: true             # line 223, 4-space indent  ← the one you want
```

This one has a nastier property. Before you edit, the *value* distinguishes them: only one is `true`. After you edit, both read `false`, and there is nothing on screen to tell you which line you changed. If you come back later to check your work, you have to go by indentation.

We only noticed that when reviewing screenshots — the "after" image was genuinely ambiguous, and needed a "before" companion to make sense. That is a documentation lesson as much as a technical one: a screenshot of a completed change can be less informative than the pair.

## What the three edits look like

Captured from the editor buffer, before and after:

```diff
   nvidiaNodeSelector:
-    gpu: 'on'
+    krum.io/gpu-plugin: hami
   passDeviceSpecsEnabled: false
   ...
-  runtimeClassName: ''
+  runtimeClassName: nvidia
```
```diff
     customURL:
       enabled: false
       host: 127.0.0.1
       path: /webhook
       port: 31998
-    enabled: true
+    enabled: false
```

Three lines out of 303. Everything else stays exactly as Rancher pre-filled it.

Afterwards we confirmed on the cluster rather than on screen, which is the check that actually matters:

```text
devicePlugin.runtimeClassName      = 'nvidia'    ← changed
devices.ascend.runtimeClassName    = ''          ← look-alike, untouched
scheduler.metricsBindAddress       = ':9395'     ← preserved
scheduler.admissionWebhook.enabled = False       ← changed
```

## Smaller things the UI does differently

Collected while walking it, because each one cost a minute:

- The wizard opens **already in Edit YAML**. There is no form-to-YAML toggle to switch; the tabs are *Edit YAML* and *Compare Changes*.
- Namespace defaults to `default`. You must type `kube-system` **and press Enter** to commit the combobox — typing alone leaves it unselected, and the app installs into the wrong namespace.
- There is a **Name** field with no default.
- The chart is listed lowercase as `hami`.
- `?q=hami` works as a search URL. `?repo=hami` renders zero charts even with every repository checked.
- Rancher shows the app **Deployed** as soon as Helm returns, which happens well before the device plugin is working. A crash-looping DaemonSet does not make the app red.
- `kubectl delete app hami` removes Rancher's record but **leaves the Helm release running**. We hit this during teardown and had to `helm uninstall` the orphan. Use *Apps → Installed Apps → Delete*, or expect to clean up.

## Why this matters beyond HAMi

The null-diffing behaviour has nothing to do with this chart. Any chart with a templated value that must be a string, or a value whose absence changes behaviour, will fail the same way if someone treats the values editor like a `-f` flag. Which is exactly what an experienced Helm user will do, because that is what the box looks like.

If you maintain Rancher-facing install docs for anything, this is worth a line.

---

Next: [Migrating a live cluster to HAMi, and back](04-migrating-both-directions.md) — doing this on a cluster with workloads already running.
