# Module 5 — Observability (Prometheus & Grafana)

Goal: add cluster and application-level monitoring on top of the k3s cluster built in Modules 3-4, using the `kube-prometheus-stack` Helm chart, then build a custom dashboard focused specifically on Robot Shop rather than relying only on generic defaults.

## Setup

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```

`kube-prometheus-stack` deploys Prometheus, Grafana, Alertmanager, `kube-state-metrics`, and `node-exporter` together as a single coordinated stack — the standard, production-representative way to get Kubernetes monitoring running, rather than wiring each component up individually.

Access:
```bash
kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

## Finding: firewalld silently blocking pod-to-pod (CNI) traffic

After install, most pods reported `Running`, but Prometheus's own target health page (`/targets`) showed a very specific pattern: targets reachable via the **node's** network stack (`apiserver`, `kubelet`, `prometheus` itself, `node-exporter`) were healthy, while everything requiring **pod-to-pod** networking (`grafana`, `alertmanager`, `coredns`, `kube-state-metrics`, `kube-prometheus-operator`) failed with:

```
Error scraping target: Get "http://10.42.0.25:8080/metrics": dial tcp 10.42.0.25:8080: connect: no route to host
```

Root cause: `firewalld` (active by default on Fedora Workstation) was only aware of the wifi interface (`wlp0s20f3`) — it had no zone assignment at all for k3s's internal CNI interfaces (`cni0`, `flannel.1`). Traffic between pods on the overlay network was being silently dropped by firewalld's default forwarding policy, even though the pods themselves were healthy and k3s's own iptables rules were correctly configured.

This is a well-documented interaction specific to running k3s on a firewalld-managed desktop OS rather than a purpose-built server — not a k3s or Kubernetes bug, but a gap between two systems (Kubernetes' networking model and a desktop firewall) that don't know about each other by default.

**Fix:** explicitly trust the CNI interfaces in firewalld:

```bash
sudo firewall-cmd --zone=trusted --add-interface=cni0 --permanent
sudo firewall-cmd --zone=trusted --add-interface=flannel.1 --permanent
sudo firewall-cmd --reload
sudo systemctl restart k3s
```

After restarting k3s and refreshing the affected pods, all Prometheus targets reported healthy (`1/1 up`) across the board, and Grafana's namespace-scoped dashboards immediately began populating with `robot-shop` data that had previously been invisible (not because Prometheus lacked permission to see it, but because it genuinely couldn't reach it over the network).

**Lesson:** on a desktop Linux distribution, an active host firewall is a real, easy-to-miss point of failure for Kubernetes networking — worth checking firewalld/interface zone assignments early if pod-to-pod communication looks broken while node-level services work fine, rather than assuming the problem is in Kubernetes itself.

## Verifying data with default dashboards

Once targets were healthy, the built-in `Kubernetes / Compute Resources / Namespace (Pods)` dashboard, filtered to the `robot-shop` namespace, immediately showed live CPU/memory utilization and per-pod resource quotas for all 11+ Robot Shop pods (including the 3 `cart` replicas from the Module 3 scaling experiment) — confirming the whole observability pipeline (metrics → Prometheus → Grafana) was working end-to-end before building anything custom.

## Custom dashboard: Robot Shop Health Overview

Built a dashboard scoped specifically to Robot Shop, rather than relying only on generic cluster-wide defaults:

**Pod restart count (last 1h)**
```promql
sum by (container) (
  kube_pod_container_status_restarts_total{
    namespace="robot-shop",
    container!="POD",
    container!=""
  }
)
```
Bar gauge. Filters out the `POD` pause container and empty container labels so only real application containers show up. Notably, this panel captured real events from this session: most services showed 2 restarts and `web` showed 4, directly reflecting the two `k3s` restarts performed while diagnosing the firewalld issue above — a genuine example of the dashboard correctly surfacing real infrastructure activity, not synthetic data.

**Memory usage by service**
```promql
sum by (container) (
  container_memory_working_set_bytes{
    namespace="robot-shop",
    container!="POD",
    container!=""
  }
) / 1024 / 1024
```
Time series, one line per container, converted to MiB for readability.

**Replica availability by service**
```promql
(
  sum by (deployment) (
    kube_deployment_status_replicas_available{
      namespace="robot-shop"
    }
  )
)
/
(
  sum by (deployment) (
    kube_deployment_spec_replicas{
      namespace="robot-shop"
    }
  )
)
* 100
```
Stat panel showing percentage of desired replicas currently available per service — a quick, unambiguous "is everything actually up" glance, preferable to a raw replica count since it self-normalizes regardless of how many replicas a service is scaled to.

## Why this matters going forward

- The firewalld/CNI finding is arguably the most operationally realistic issue hit across the entire project — it's the exact category of problem that appears when running Kubernetes outside a managed cloud environment (EKS/GKE handle this networking setup invisibly), and recognizing "node-level traffic works, pod-level traffic doesn't" as the diagnostic signature is a transferable skill, not a one-off fix.
- The restart-count panel showing real restarts from today's own troubleshooting is a good example of monitoring doing its actual job: surfacing real infrastructure events rather than just looking healthy by default.
- This closes Project 1 (`DevOps_Homelab`). The full arc — Linux/networking baseline, Podman, k3s + Helm, Ansible automation, and now observability — covers containerization, orchestration, infrastructure-as-code, and monitoring end-to-end on a single local machine.
