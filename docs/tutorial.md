# Tutorial: your first shared GPU

By the end of this you will have watched two pods run at the same time on one physical GPU, each held to its own memory limit — and you'll have seen why they couldn't before.

About 20 minutes. Every command is given in full, with what you should see. There are no choices to make; where a real deployment would need a decision, this tutorial picks one and moves on, and the [scenarios](../scenarios) cover the decisions later.

**Use a cluster you can experiment on.** You'll install and uninstall two device plugins on one node. Step 6 puts everything back, but don't learn this on a cluster other people are using.

## What you need

- A Kubernetes cluster with **one node that has an NVIDIA GPU**, its driver and container toolkit installed.
- **No GPU device plugin installed yet** — you'll install both of them here. If `nvidia-device-plugin` or HAMi is already running, follow [migrate-nvidia-to-hami](../scenarios/migrate-nvidia-to-hami/) instead; this tutorial would collide with it.
- `kubectl` and `helm`, and permission to create cluster-scoped objects.

Pick your node and keep it in a variable — every step uses it:

```sh
export NODE=<your-gpu-node>
```

## 1. Confirm the node can run GPU pods

Two things have to be true before either plugin will work: a `nvidia` RuntimeClass exists, and this node can honour it.

```sh
kubectl get runtimeclass nvidia
```

If that says `NotFound`, create it:

```sh
kubectl apply -f - <<'EOF'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
EOF
```

Now prove the node can actually use it, by asking the driver what it sees:

```sh
kubectl run gpu-probe --rm -i --restart=Never \
  --image=nvidia/cuda:12.2.0-base-ubuntu22.04 \
  --overrides="{\"spec\":{\"runtimeClassName\":\"nvidia\",\"nodeSelector\":{\"kubernetes.io/hostname\":\"$NODE\"}}}" \
  -- nvidia-smi --query-gpu=name,memory.total --format=csv
```

You should see your card and its real memory:

```text
name, memory.total [MiB]
NVIDIA GeForce RTX 2060 with Max-Q Design, 6144 MiB
```

Write down that memory figure — you'll watch it change later.

If instead the pod hangs and you see `no runtime for "nvidia" is configured`, this node's NVIDIA setup isn't finished. Stop here and fix that; nothing below will work. ([prerequisites](prerequisites.md) covers it.)

## 2. See the limitation

Install the stock NVIDIA device plugin, so the cluster can schedule GPUs at all:

```sh
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update nvdp

helm install nvdp nvdp/nvidia-device-plugin \
  -n nvidia-device-plugin --create-namespace --version 0.14.5 \
  --set runtimeClassName=nvidia
```

Give it a few seconds, then ask what the node advertises:

```sh
kubectl get node $NODE -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

```text
1
```

One GPU — one pod may have it. Prove that's a real limit by asking for two:

```sh
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: hog-a}
spec:
  runtimeClassName: nvidia
  restartPolicy: Never
  containers:
    - name: c
      image: nvidia/cuda:12.2.0-base-ubuntu22.04
      command: ["sleep", "300"]
      resources: {limits: {nvidia.com/gpu: "1"}}
---
apiVersion: v1
kind: Pod
metadata: {name: hog-b}
spec:
  runtimeClassName: nvidia
  restartPolicy: Never
  containers:
    - name: c
      image: nvidia/cuda:12.2.0-base-ubuntu22.04
      command: ["sleep", "300"]
      resources: {limits: {nvidia.com/gpu: "1"}}
EOF

kubectl get pods hog-a hog-b
```

```text
NAME    READY   STATUS    RESTARTS   AGE
hog-a   1/1     Running   0          10s
hog-b   0/1     Pending   0          10s
```

One runs, one waits. Ask why:

```sh
kubectl get event --field-selector involvedObject.name=hog-b | grep -i insufficient
```

```text
... 0/1 nodes are available: 1 Insufficient nvidia.com/gpu.
```

That's the problem HAMi solves. `hog-a` is holding a whole card to run `sleep`, and nothing else can have any of it.

Clean up:

```sh
kubectl delete pod hog-a hog-b --wait=false
```

## 3. Hand the node to HAMi

The two plugins both claim the resource name `nvidia.com/gpu`, so they can't share a node. Remove the stock one:

```sh
helm uninstall nvdp -n nvidia-device-plugin
```

Install HAMi. These values are explained in [deploy-with-helm](../scenarios/deploy-with-helm/); for now, take them as given:

```sh
helm repo add hami-charts https://project-hami.github.io/HAMi/
helm repo update hami-charts

helm install hami hami-charts/hami -n kube-system --version 2.9.0 \
  --set devicePlugin.runtimeClassName=nvidia \
  --set devicePlugin.createRuntimeClass=false \
  --set devicePlugin.deviceSplitCount=10 \
  --set devicePlugin.nvidiaNodeSelector.gpu=null \
  --set devicePlugin.nvidiaNodeSelector.gpu-plugin=hami
```

Nothing happens yet — the plugin only runs on nodes you've labelled. Label yours:

```sh
kubectl label node $NODE gpu-plugin=hami --overwrite
```

Watch the plugin arrive. Wait until it reads `2/2 Running`:

```sh
kubectl -n kube-system get pods -l app.kubernetes.io/component=hami-device-plugin -w
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
hami-device-plugin-6r5qf   2/2     Running   0          25s
```

(`Ctrl-C` to stop watching. If it goes to `CrashLoopBackOff` instead, the `runtimeClassName` setting didn't apply — check the install command.)

## 4. See what changed

```sh
kubectl get node $NODE -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

```text
10
```

One physical card is now ten schedulable slots — that's the `deviceSplitCount=10` you installed with.

The card's real details moved to a node annotation:

```sh
kubectl get node $NODE -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}{"\n"}'
```

```text
[{"id":"GPU-e1b4fb02-...","count":10,"devmem":6144,"devcore":100,"type":"NVIDIA GeForce RTX 2060...","health":true}]
```

`devmem: 6144` is the real memory from step 1. HAMi's scheduler reads this to decide who fits. **Don't go looking for a `nvidia.com/gpumem` figure in the node's `allocatable`** — it is never there, and its absence is not a problem.

## 5. Share the GPU

Ask for a slice rather than a card. `nvidia.com/gpumem` is in MiB:

```sh
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: share-a}
spec:
  runtimeClassName: nvidia
  restartPolicy: Never
  containers:
    - name: c
      image: nvidia/cuda:12.2.0-base-ubuntu22.04
      command: ["sh","-c","nvidia-smi --query-gpu=memory.total --format=csv; sleep 300"]
      resources: {limits: {nvidia.com/gpu: "1", nvidia.com/gpumem: "1024"}}
---
apiVersion: v1
kind: Pod
metadata: {name: share-b}
spec:
  runtimeClassName: nvidia
  restartPolicy: Never
  containers:
    - name: c
      image: nvidia/cuda:12.2.0-base-ubuntu22.04
      command: ["sh","-c","nvidia-smi --query-gpu=memory.total --format=csv; sleep 300"]
      resources: {limits: {nvidia.com/gpu: "1", nvidia.com/gpumem: "2048"}}
EOF

kubectl get pods share-a share-b -o wide
```

Both run, on the same node:

```text
NAME      READY   STATUS    RESTARTS   AGE   NODE
share-a   1/1     Running   0          15s   <your node>
share-b   1/1     Running   0          15s   <your node>
```

That is the thing you came for. Now look at what each one believes it has:

```sh
kubectl logs share-a
kubectl logs share-b
```

```text
memory.total [MiB]
1024 MiB
```

```text
memory.total [MiB]
2048 MiB
```

Neither reports 6144. The card is physically the same one you probed in step 1, but each container sees only its own limit — HAMi injects a library that intercepts CUDA memory calls, so the ceiling is enforced inside the container rather than merely promised by the scheduler. A process in `share-a` that tries to allocate 2 GB fails, instead of quietly eating its neighbour's memory.

One more thing worth noticing:

```sh
kubectl get pod share-a -o jsonpath='{.spec.schedulerName}{"\n"}'
```

```text
hami-scheduler
```

You never asked for that. HAMi's admission webhook rewrote it, because the default Kubernetes scheduler has no idea what `nvidia.com/gpumem` means and would have overcommitted the card.

## 6. Put everything back

```sh
kubectl delete pod share-a share-b --wait=false
helm uninstall hami -n kube-system
kubectl label node $NODE gpu-plugin-
kubectl annotate node $NODE hami.io/node-nvidia-register- hami.io/node-handshake-
kubectl delete namespace nvidia-device-plugin
```

The annotations need removing by hand — `helm uninstall` leaves them behind. Confirm you're clean:

```sh
kubectl get node $NODE -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

Empty output: no device plugin, no GPUs advertised, back where you started.

## What you did

- Watched the stock plugin advertise one GPU, and a second pod fail to get one.
- Replaced it with HAMi, and watched one card become ten slots.
- Ran two pods on that card with different memory ceilings, and confirmed from inside each container that the ceilings were real.
- Saw the webhook redirect a pod to HAMi's scheduler without being asked.

## Where to go next

- [How HAMi works](how-hami-works.md) — why there are three components, what the interception library implies for your images, and what sharing costs.
- [deploy-with-helm](../scenarios/deploy-with-helm/) — the same install for real, with the values and trade-offs explained.
- [migrate-nvidia-to-hami](../scenarios/migrate-nvidia-to-hami/) — doing this on a cluster with GPU workloads already running, one node at a time.
- [reference](reference.md) — the lookup tables.
