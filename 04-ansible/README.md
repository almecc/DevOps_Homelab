# Module 4 — Ansible

Goal: automate what Modules 2 and 3 did by hand — Podman configuration, k3s installation, and the Robot Shop Helm deployment — as repeatable, idempotent Ansible playbooks. Everything here was already working manually; this module is about turning that manual, one-off process into something that produces the same result reliably, every time, on a machine that starts from scratch or one that's already partially configured.

## Setup

- `ansible-core` installed (not the full `ansible` metapackage, which bundles hundreds of community collections not needed here)
- `kubernetes.core` collection installed for the Kubernetes/Helm modules
- Single-host inventory (`inventory.ini`) using `ansible_connection=local`, since Ansible manages the same machine it runs on — structured the same way a real multi-host inventory would be, for the pattern to transfer

```ini
[homelab]
localhost ansible_connection=local
```

## Playbook 1 — `podman.yml`

Ensures Podman and `podman-compose` are installed, and encodes the Module 2 registry fix directly: guarantees `docker.io` is the configured `unqualified-search-registries` entry in `/etc/containers/registries.conf`, rather than leaving `registry.access.redhat.com` ahead of it (the root cause of Module 2's pull failure).

Run twice to confirm idempotency: first run reported `ok=4, changed=0` (Podman/podman-compose already installed, registry config already correct from the earlier manual fix) — everything was already true, so nothing needed to change.

## Playbook 2 — `k3s.yml`

Installs k3s if not already present (checked via a `stat` on the k3s binary, so re-running never re-triggers the install script unnecessarily), ensures the service is running/enabled, waits for the node to report `Ready`, and — the more interesting task — recreates the `standard` StorageClass fix from Module 3, so a fresh machine doesn't hit the same Redis PVC failure that took real debugging to diagnose the first time.

First run: `ok=6, changed=1` (only the StorageClass task reported a change, since Ansible's `k8s` module reconciles/re-applies declared objects even when they already match — expected behavior, not a sign of an actual recreation).

Second run, immediately after: `ok=6, changed=0` — full idempotency confirmed, nothing left to do.

### Troubleshooting hit while building this playbook

- **kubeconfig permission denied**: `/etc/rancher/k3s/k3s.yaml` is `0600` by default (root-only), which blocked the `kubernetes.core.k8s` module from reading it even under `become`. Fixed by adding a task to set it to `0644` before the StorageClass task runs.
- **Missing Python `kubernetes` library under root**: the `k8s` Ansible module depends on the `kubernetes` Python package, but installing it under the regular user account (`pip install kubernetes --break-system-packages`) didn't make it visible to tasks running as root via `become` — root has its own separate site-packages location. Fixed by explicitly installing as root: `sudo /usr/bin/python3.14 -m pip install kubernetes --break-system-packages`.

## Playbook 3 — `robot-shop.yml`

Ensures the `robot-shop` namespace exists and deploys the Robot Shop Helm chart (pinned to `image.version: 2.1.0`, matching what was verified working in Modules 2–3) using the `kubernetes.core.helm` module.

Run against the already-deployed Module 3 release: `ok=3, changed=0` — confirmed the existing deployment already matches the declared state, no action taken.

**Known limitation:** Ansible's `helm` module surfaced a warning that its default idempotency check can be imprecise without `helm diff` installed (`>=3.4.1`). Not installed for this module since the check was accurate for this deployment, but worth knowing as a real limitation rather than treating the `changed=0` result as an absolute guarantee in more complex scenarios.

**Prerequisite:** this playbook references a local clone of `instana/robot-shop` for the chart path (`../../03-k3s-helm/robot-shop/K8s/helm`), which isn't committed to this repo (same decision as Modules 2–3 — vendoring someone else's full source tree adds no value here). Running this playbook against a fresh machine requires cloning it first:

```bash
cd 03-k3s-helm
git clone https://github.com/instana/robot-shop.git
```

## Why this matters going forward

- All three playbooks were proven idempotent with real second-run output, not just written and assumed correct — `changed=0` on a repeat run is the actual evidence that the automation is doing what it claims.
- The StorageClass and kubeconfig-permission fixes prove this isn't just "install some packages" automation — it encodes real operational knowledge earned through the Module 3 debugging session, which is exactly what makes Ansible playbooks valuable over a plain install script.
- Next: Prometheus and Grafana (Module 5) — observability for the cluster and app these playbooks now provision, closing out Project 1.
