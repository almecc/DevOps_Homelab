# Module 2 — Podman

Goal: run the Robot Shop microservices demo locally via Podman Compose, on rootless Podman as the sole container runtimR.

## Setup

- Podman + `podman-compose` installed and verified
- Spent a focused week independently getting comfortable with Podman's structure (images, rootless model, compose) before this module, so this module documents the actual deployment rather than re-covering fundamentals

## Deploying Robot Shop

```bash
cd 02-podman
git clone https://github.com/instana/robot-shop.git
cd robot-shop
podman-compose up -d
```

Robot Shop is Instana's demo e-commerce app: 11 microservices across multiple languages (Node.js, Java, Python, PHP, Golang) backed by MongoDB, MySQL, Redis, and RabbitMQ. `Istio/` and the Instana agent components were intentionally skipped — out of scope for this homelab.

## Finding: rootless Podman resolved the wrong registry

First `podman-compose up -d` failed pulling `robotshop/rs-dispatch:2.1.0`:

```
error: unable to copy from source docker://registry.access.redhat.com/robotshop/rs-dispatch:2.1.0:
reading manifest 2.1.0 in registry.access.redhat.com/robotshop/rs-dispatch: name unknown: Repo not found
```

Root cause: the image name in Robot Shop's compose file is unqualified (`robotshop/rs-dispatch:2.1.0`, no registry prefix). Podman resolves unqualified image names against the search order in `/etc/containers/registries.conf` under `unqualified-search-registries`, and `registry.access.redhat.com` was ahead of `docker.io` in that list — so Podman looked for a Red Hat-hosted image that doesn't exist, instead of the actual image on Docker Hub.

**Fix:** removed `registry.access.redhat.com` from `unqualified-search-registries` in `/etc/containers/registries.conf`, leaving `docker.io` as the resolution target. Re-ran `podman-compose up -d` and all images pulled correctly.

This is a rootless-Podman-specific gotcha worth knowing: unlike Docker, which defaults to Docker Hub only, Podman's multi-registry search order means the same unqualified image name can silently resolve differently depending on how `registries.conf` is configured.

## Result

```bash
podman ps
```

All 11 containers up:

| Service | Image | Status |
|---|---|---|
| web | rs-web:2.1.0 | Up, healthy, `0.0.0.0:8080->8080/tcp` |
| catalogue | rs-catalogue:2.1.0 | Up, healthy |
| cart | rs-cart:2.1.0 | Up, healthy |
| user | rs-user:2.1.0 | Up, healthy |
| payment | rs-payment:2.1.0 | Up, healthy |
| shipping | rs-shipping:2.1.0 | Up, healthy |
| ratings | rs-ratings:2.1.0 | Up, healthy |
| dispatch | rs-dispatch:2.1.0 | Up (background AMQP worker, no HTTP healthcheck) |
| mongodb | rs-mongodb:2.1.0 | Up |
| mysql | rs-mysql-db:2.1.0 | Up |
| redis | redis:6.2-alpine | Up |
| rabbitmq | rabbitmq:3.8-management-alpine | Up |

Frontend accessible at `http://localhost:8080`. Verified end-to-end: browsed the catalogue, added an item to cart, confirming inter-service communication (frontend → catalogue → mongo, cart → redis) is actually working, not just static rendering.

## Why this matters going forward

- The `registries.conf` search-order issue is a good example of a real rootless-Podman operational difference from Docker — worth remembering before deploying anything else with unqualified image names.
- Robot Shop now runs as containers understood end-to-end (not just "docker-compose up and hope"), setting up Module 3, where the same application gets redeployed on k3s via its Helm chart — a direct before/after comparison of Compose vs. Kubernetes for the same app.

## Next: Module 3 — k3s & Helm

Redeploy Robot Shop on a local k3s cluster using its official Helm chart.
