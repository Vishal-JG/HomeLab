# Homelab

A self-hosted Kubernetes homelab running on a single Ubuntu box (bytecave), exposed securely over Tailscale, with a CI/CD pipeline automating builds and deploys. Ansible-based infrastructure-as-code is in progress, aiming to reproduce the entire setup from a bare machine.

The cluster runs Briefly, a Telegram bot that sends scheduled news updates, as the primary workload. The project was built end to end: bare metal to K3s, K3s to Tailscale-secured networking, manual deploys to automated GitOps-style CI/CD. This project is an ongoing work in progress and will keep evolving as I add new pieces.

## Architecture

```
Sydney laptop
      |
      | (Tailscale mesh network, WireGuard-based)
      v
bytecave (Ubuntu host)
      |
      v
K3s cluster
      |
      +-- Tailscale Operator (exposes services on the tailnet)
      |
      +-- Briefly Deployment (Telegram bot, containerized)
      |        |
      |        +-- ConfigMap (app config)
      |        +-- Secret (Telegram credentials)
      |
      +-- Service + Ingress (tailscale ingress class, reachable at a .ts.net URL)
      |
      +-- CronJob (scheduled news fetch/send)

GitHub Actions (on push to main)
      |
      +-- Build Docker image --> push to registry
      |
      +-- Deploy job: tailscale ssh into bytecave
                |
                +-- git pull latest manifests
                +-- kubectl apply -f . (full reconciliation)
                +-- kubectl set image (bump running container)
```

My laptop never talks to bytecave over the public internet. Everything routes through the tailnet, including the GitHub Actions runner, which joins the tailnet as a tagged device (tag ci) to reach the cluster and deploy.

## Tech Stack

| Layer | Tool |
|---|---|
| Orchestration | K3s (lightweight Kubernetes) |
| Networking | Tailscale (mesh VPN, Tailscale Kubernetes Operator, Tailscale SSH) |
| CI/CD | GitHub Actions |
| Infrastructure as Code | Ansible (in progress) |
| Application | Briefly, a containerized Telegram bot |
| Container registry | Docker Hub / GHCR |
| Config and secrets | Kubernetes ConfigMap and Secret |

## Repo Structure

```
homelab/
  manifests/
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    secret.yaml
    cronjob.yaml
  ansible/
    playbook.yml
    inventory.ini
    roles/
      k3s-install/
      tailscale-setup/
      app-deploy/
  .github/
    workflows/
      ci.yml
  journey.md
  README.md
```

- manifests holds every Kubernetes resource that gets applied to the cluster.
- ansible holds the playbook and roles aiming to reproduce the entire host setup from a bare Ubuntu box. This is still a work in progress, see the note in Setup below.
- .github/workflows/ci.yml builds, pushes, and deploys on every push to main.
- journey.md is the full narrative of what I built, what broke, and how I fixed it. Worth reading if you want the real story behind the project rather than just the end state.

## Setup

### Prerequisites

- A bare Ubuntu machine (or VM) you control
- A Tailscale account, with an OAuth client that has both Devices Core and Auth Keys scopes
- A GitHub repository with the following secrets configured: registry credentials, Tailscale OAuth client ID and secret, and any app-specific secrets (for example Telegram bot token and chat ID)

### Manual setup

1. Install K3s on the target host:

```
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

2. Install the Tailscale Kubernetes Operator via Helm:

```
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  -n tailscale --create-namespace \
  --set-string oauth.clientId="<your-client-id>" \
  --set-string oauth.clientSecret="<your-client-secret>"
```

3. Apply the manifests:

```
kubectl apply -f manifests/
```

4. Confirm the app is reachable at its .ts.net URL from any device on your tailnet.

### Verifying it worked

```
kubectl get nodes
kubectl get pods
kubectl get cronjobs
kubectl get ingress
```

All pods should show Running, and the Ingress should list a .ts.net hostname you can hit from any device on your tailnet.

### Ansible automation (work in progress)

The ansible directory contains a playbook aiming to fully automate the manual steps above, covering K3s install, Tailscale setup, and app deploy through separate roles. It is not yet stable enough to recommend as the primary setup path, largely due to a chain of authentication issues (SSH keys, host keys, sudo) and path resolution quirks that are still being ironed out. See journey.md for details on what has come up so far. Once it is reliable, this will replace the manual steps as the default setup method.

## CI/CD

On every push to main, GitHub Actions:

1. Builds the Docker image for the app.
2. Pushes the image to the container registry.
3. Connects to bytecave over Tailscale SSH using a CI runner tagged tag ci.
4. Runs git pull to fetch the latest manifests, then kubectl apply -f . so any changes to Deployments, Services, ConfigMaps, Secrets, or CronJobs are actually applied, not just the image tag.
5. Patches the running Deployment's image if needed.

This two-step apply-then-patch approach exists because an image-only deploy silently misses any changes to the surrounding manifests. See journey.md for the exact bug that revealed this.

## Known Limitations

- Single-node cluster. No high availability, and a host reboot or crash takes everything down.
- Secrets are created manually rather than managed through a proper secrets manager or sealed secrets approach.
- No monitoring or alerting. A failing pod can go unnoticed if an older healthy pod is still serving traffic, which happened during development.
- Ansible automation is still a work in progress and not yet the recommended setup path.

## What I Would Add Next

- Finish hardening the Ansible playbook so it can fully replace the manual setup steps.
- Proper secrets management, replacing manually created Kubernetes Secrets with something like Sealed Secrets or an external secrets operator.
- Monitoring and alerting (Prometheus and Grafana, or a lighter alternative) so failing pods surface immediately instead of being masked by stale traffic on an old pod.
- Extending the Ansible playbook to handle multi-node setups instead of a single bare box.
- CKA-style hardening: resource limits, liveness and readiness probes, and network policies that are currently absent from the manifests.

## The Full Story

This project is a continuous work in progress. journey.md walks through every major step of the build so far and the real challenges that came up along the way, including a four-layer authentication failure chain in Ansible, a Tailscale SSH ACL authorization gap that briefly locked me out of my own machine, and a CI/CD bug where an entire CronJob manifest sat unused in the repo for days without me noticing. If you want the debugging stories behind this project rather than just the current state, that's the file to read.
