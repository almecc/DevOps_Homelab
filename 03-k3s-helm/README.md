# Module 3 — k3s & Helm

Goal: redeploy Robot Shop — the same app deployed via Podman Compose in Module 2 — on a local Kubernetes cluster (k3s), using its official Helm chart instead of `docker-compose.yaml`. Same application, different orchestration layer, so the two approaches can be compared directly.

## Setup

- k3s installed as a single-node cluster (control-plane + worker on the same laptop)
- `kubectl` pointed at the k3s cluster via `~/.kube/config` (not Podman Desktop's separate Kubernetes context, which was checked and ruled out first)
- Podman Compose stack torn down (`podman-compose down`) before starting k3s workloads, per the RAM-discipline rule — don't run two full deployments of the same app simultaneously on a 16GB machine
- Helm installed via the official install script
- Deployed with `helm install robot-shop . --namespace robot-shop --set image.version=2.1.0`, pinning the same image tag that was verified working in Module 2 (chart's default `image.version: latest` was not trusted)

## What Helm actually replaced

`podman-compose up -d` in Module 2 ran 11 containers directly from a single YAML file. Here, `helm install` reads the chart's `templates/` directory and renders a full set of Kubernetes objects — a **Deployment** + **Service** pair for each of the 11 microservices, plus a **StatefulSet** for `redis` (since it needs stable identity and persistent storage). The Deployments describe desired replica counts; the Kubernetes control plane continuously reconciles actual state to match, restarting failed pods automatically — something Podman Compose doesn't do on its own.

## Troubleshooting log

Five distinct issues were hit and resolved during this deployment — documented in the order encountered, since each one is a genuinely different class of Kubernetes/infrastructure problem.

### 1. Transient DNS resolution failures during image pulls

Several pods (`cart`, `catalogue`, `payment`, `ratings`, `user`, `web`) failed with:

```
failed to resolve reference "docker.io/robotshop/rs-cart:2.1.0": failed to authorize: failed to fetch anonymous token:
Get "https://auth.docker.io/token...": dial tcp: lookup auth.docker.io: Try again
```

`nslookup` against the host resolver succeeded moments later, confirming this was a transient wifi/DNS blip rather than a structural containerd/DNS misconfiguration. Manually pulling the image with `sudo k3s ctr images pull` succeeded once DNS was briefly stable, confirming the image and registry path were both fine.

### 2. Docker Hub anonymous pull rate limiting

After deleting and recreating pods repeatedly while chasing the DNS issue, several images stopped pulling entirely. This pointed to Docker Hub's anonymous-pull rate limit (roughly 100 pulls per 6 hours per IP) — the retry storm from the DNS troubleshooting burned through it. Root cause confirmed by the mixed pattern: some services (`dispatch`, `mongodb`, `mysql`, `rabbitmq`, `shipping`) pulled fine while others consistently failed, which is inconsistent with a pure DNS or registry issue and consistent with per-image throttling hitting at different points in the retry cycle.

**Lesson:** repeatedly deleting/recreating pods while debugging a pull issue makes the problem worse, not better — it multiplies pull attempts against a rate-limited registry.

### 3. Namespace stuck in `Terminating`

After a full `helm uninstall` + `kubectl delete namespace` to get a clean slate, the namespace hung in `Terminating` status well past pod deletion. Checked for blocking resources across all namespaced API types:

```bash
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get -n robot-shop --ignore-not-found
```

Nothing was found blocking it — it cleared on its own shortly after, confirming this was normal (if slow) cleanup rather than a stuck finalizer requiring manual intervention.

### 4. Missing `standard` StorageClass blocking Redis's PVC

`redis-0` sat in `Pending` with:

```
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
```

k3s ships a default StorageClass named `local-path`, but the Redis chart's PVC template requests a class named `standard`, which didn't exist on this cluster. Confirmed via `kubectl get pvc` events: `storageclass.storage.k8s.io "standard" not found`.

**Fix:** created a `standard` StorageClass pointing at the same `rancher.io/local-path` provisioner k3s already uses:

```bash
kubectl get storageclass local-path -o yaml > standard-sc.yaml
# edit: rename to "standard", strip resourceVersion/uid/creationTimestamp
kubectl apply -f standard-sc.yaml
```

PVC bound immediately afterward. Known minor cleanup item: the cluster now has two StorageClasses (`local-path` and `standard`) both marked default and pointing at the same provisioner — redundant but not harmful, left as-is since it works.

### 5. Node IP change mid-session broke kubelet's self-identity

The most significant issue: partway through the session, the laptop's IP changed (wifi network switch/DHCP reassignment, `192.168.x.x` → `10.186.x.x`). k3s had registered the node under the old IP, and kubelet entered a persistent retry loop:

```
"Failed to set some node status fields" err="failed to validate nodeIP: node IP: \"10.186.x.x\" not found in the host's network interfaces"
```

alongside API authorization errors on pod status updates (`no relationship found between node 'fedora' and this object`) — a downstream symptom of the same node-identity mismatch. This explains the inconsistent, hard-to-pin-down behavior seen earlier in the session: pods stuck silently in `ContainerCreating` with no error, old `Terminating` pods that wouldn't clear, node status intermittently unavailable — all stemming from kubelet fighting an identity mismatch against the API server, not from the pull/storage issues being chased at the time.

**Fix:** `sudo systemctl restart k3s` — forced kubelet to re-register cleanly against the current IP. Once restarted, previously stuck pods cleared and the final `cart` replica came up normally within seconds.

**Lesson:** on a laptop (vs. a server with a static IP), a mid-session network change is a real, non-obvious failure mode for a single-node k3s cluster — worth checking node identity/IP consistency early if troubleshooting turns unusually inconsistent, rather than continuing to chase the original symptom.

## Result

```
kubectl get pods -n robot-shop
```

All 11 workloads `Running` (11/11 ready). `web` Service is type `LoadBalancer`, and k3s's built-in `klipper-lb` assigned it the node's own IP directly — no manual NodePort override needed, unlike some guidance aimed at minikube/minishift.

Verified end-to-end in browser at `http://<node-ip>:8080` — same Robot Shop frontend as Module 2, this time backed by Kubernetes-managed Pods/Services/Deployments instead of Compose-managed containers.

## Compose vs. Helm/k3s — what actually changed

| | Podman Compose (Module 2) | k3s + Helm (Module 3) |
|---|---|---|
| Unit of deployment | Container, run directly | Pod, managed by a Deployment |
| Self-healing | None — a crashed container stays down | Control plane restarts failed pods automatically |
| Networking | Compose's internal DNS/network | Kubernetes Services (stable ClusterIP per service) |
| Storage | Bind mounts / named volumes | PersistentVolumeClaims + StorageClass-backed provisioning |
| Scaling | Manual, per-container | Declarative replica counts on Deployments |
| Exposure | Explicit port mapping | Service types (`ClusterIP`, `NodePort`, `LoadBalancer`) |
| Failure surface hit this module | One registry-resolution issue | Five distinct issues: DNS, rate limiting, namespace cleanup, storage class mismatch, node identity — genuinely broader operational surface area |

## Why this matters going forward

- The node-identity issue is the single most valuable finding in this module: Kubernetes' node/kubelet identity model doesn't tolerate an unannounced IP change well, which is directly relevant to running any single-node cluster on non-static-IP hardware (laptops, home networks).
- The StorageClass mismatch is a good example of a Helm chart making assumptions (`standard` class name) that don't hold on every distribution — always worth checking `values.yaml` and template assumptions against the actual cluster before assuming defaults will work.
- Ansible (Module 4) is a natural next step from here — much of what was done manually in this module (installing k3s, fixing registries.conf-equivalent config, applying the StorageClass fix) is exactly the kind of repeatable provisioning Ansible is built to automate.
