# DevOps_Homelab

A hands-on DevOps homelab built on a single Fedora Workstation laptop, covering Linux/networking, containers, Kubernetes orchestration, infrastructure automation, and observability, end to end.

Each module documents real work: the same Robot Shop microservices app deployed and progressively hardened across Podman → k3s/Helm → Ansible automation → Prometheus/Grafana monitoring, including the actual issues hit along the way i.e registry resolution failures, Docker Hub rate limiting, Kubernetes node identity mismatches, and a firewalld/CNI networking conflict, each diagnosed and resolved. Full troubleshooting logs are in each module's README File.

This is Project 1 of two. Project 2 (deploying and automating the same Robot Shop app on AWS) begins once this one is complete.

## What this is ?

I'm building homelab on one machine (no extra hardware) to understand how the pieces actually work before automating them. This repo is where I'm tracking that process.

## Architecture
<img width="1400" height="750" alt="image" src="https://github.com/user-attachments/assets/325b806a-e2a8-4197-a5e9-1604b71853b1" />

## What's covered

- **Linux & networking** — host baseline: services, storage, IP/routing, DNS, firewall, SSH
- **Podman** — rootless containers; deployed Robot Shop via Compose, debugged a registry resolution failure
- **k3s & Helm** — redeployed the same app on Kubernetes; resolved 5 distinct real issues (DNS, rate limiting, stuck namespace, missing StorageClass, node IP/identity mismatch)
- **Ansible** — 3 idempotent playbooks automating the entire setup above, each proven with zero-change second runs
- **Prometheus & Grafana** — full observability stack; diagnosed and fixed a firewalld/CNI networking conflict blocking pod-to-pod traffic; built a custom Robot Shop health dashboard

## Environment

- OS: Fedora 44 Workstation
- Hardware: single laptop, 16GB RAM
- Container runtime: Podman
- Demo app: [Robot Shop](https://github.com/instana/robot-shop)

## Structure

```
DevOps_Homelab/
├── 01-linux-networking/
├── 02-podman/
├── 03-k3s-helm/
├── 04-ansible/
├── 05-observability/
```

## Progress

- [x] Linux & networking
- [x] Podman
- [x] k3s & Helm
- [x] Ansible
- [x] Prometheus & Grafana
