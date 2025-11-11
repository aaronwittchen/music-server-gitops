# Homelab Server Setup

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-blue?logo=kubernetes)
![Flux](https://img.shields.io/badge/Flux-2.7-purple?logo=flux)
![Docker](https://img.shields.io/badge/Docker%20Desktop-Latest-2496ED?logo=docker)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![License](https://img.shields.io/badge/License-MIT-green)
![Last Updated](https://img.shields.io/badge/Last%20Updated-Nov%202025-blue)

A self-hosted infrastructure for streaming music, syncing notes, and monitoring health metrics—running on a home server with Kubernetes, Flux, and GitOps principles. The system is designed to be lightweight enough to run on modest hardware, including repurposed laptops running Ubuntu.

## Features

* **Music Streaming** - Self-hosted Navidrome with 3-replica redundancy
* **Note Syncing** - Obsidian vault sync via Syncthing
* **Remote Access** - Secure VPN tunneling with Tailscale
* **High Availability** - Auto-healing pods with health checks
* **GitOps** - All configs version-controlled in Git
* **Load Balancing** - Automatic traffic distribution

---

## Tech Stack

### Kubernetes

Kubernetes is a powerful container orchestration platform that manages and runs all your containers with self-healing and redundancy. For local development, you can use Docker Desktop (which has built-in Kubernetes support for Windows), or lightweight alternatives like Kind or Minikube. Simply install Docker Desktop and enable Kubernetes in the settings to get started.

### Flux

Flux is a GitOps tool for Kubernetes that keeps your cluster in sync with your Git repository. It automatically applies YAML manifests, handles updates, and ensures your applications always match the desired state. Flux enables declarative, version-controlled, and reproducible deployments that are easy to maintain and self-healing.

### Git

Git is the backbone of version control for this setup. All deployment configurations—including Navidrome and Syncthing configs, Helm charts, and Kubernetes manifests—are stored in Git repositories. This allows you to track changes, roll back when necessary, and maintain a reproducible environment across devices.

### Helm

Helm is a package manager for Kubernetes that simplifies deployment by providing reusable, pre-built templates called charts. Instead of writing raw YAML files, you can define values and let Helm generate manifests. This makes updates consistent and reduces manual effort, with the added benefit of a rich ecosystem of community charts.

### Navidrome

Navidrome is a lightweight music server with a low resource footprint. It offers a web UI, mobile client support, and compatibility with the Subsonic API. In this setup, Navidrome runs with three replicas and health checks to ensure high availability. Music is read from a local Windows folder mounted into the containers, accessible at `http://localhost:30533` locally or `http://<tailscale-ip>:30533` remotely.

### Syncthing

Syncthing provides decentralized file synchronization, keeping your Obsidian vault (or other data) in sync across devices. It’s deployed with three replicas for redundancy, accessible at `http://localhost:30384` locally or `http://<tailscale-ip>:30384` remotely.

### Tailscale

Tailscale creates a secure, encrypted VPN mesh network, allowing remote access without exposing ports to the internet. It automatically connects your devices—including laptop, main PC, and phone—via private encrypted tunnels for seamless and safe connectivity.

---

## Architecture

Your Ubuntu-based access point runs Docker and Kubernetes, hosting Flux as the GitOps controller. Flux watches your GitHub repository and automatically deploys all applications. Navidrome runs as 3 replicas for load balancing and redundancy, reading music from your local folder. Syncthing runs as a single stateful pod with persistent storage, syncing your Obsidian vault across all devices. Tailscale provides encrypted VPN access to both services from your main PC, phone, or any remote location. All configurations are version-controlled in Git—the single source of truth for your entire infrastructure.

---

## Redundancy & High Availability

### Deployment Strategy

#### Navidrome (Music Server):
- **Read-only data** - Music files don't change
- **Multiple pods** - All pods serve the same data
- **3 replicas** - Provides load balancing + redundancy
- **Stateless** - No conflicts between instances

#### Syncthing (Note Sync):
- **Stateful** - Constantly syncing changes
- **Single instance** - Multiple replicas would conflict
- **1 replica** - Only one instance manages the vault
- **Regular backups** - Critical for data safety

### High Availability Setup

```
Load Balancer (K8s Service)
         │
    ┌────┼────┐
    ▼    ▼    ▼
  Pod1 Pod2 Pod3
  
If Pod1 crashes:
  - K8s detects failure (liveness probe)
  - Automatically restarts Pod1
  - Pod2 & Pod3 handle traffic (zero downtime)
  - Pod1 comes back online
```

### Health Checks

Each pod has:
- **Liveness Probe**: Checks if pod is alive (restarts if dead)
- **Readiness Probe**: Checks if pod is ready to serve traffic
- **Resource Limits**: Prevents pods from consuming all resources

---

## Usage

### Access Services

**Local:**
- Music: `http://localhost:30533`
- Notes: `http://localhost:30384`

**Remote (via Tailscale):**
- Music: `http://100.67.166.18:30533`
- Notes: `http://100.67.166.18:30384`

### Check Status

```bash
# View all pods
kubectl get pods

# Check Flux status
flux get all

# Watch pods in real-time
kubectl get pods -w

# View pod logs
kubectl logs deployment/navidrome
```

---

## Future Plans

### Health Monitoring

**Goal:** Track personal health metrics and visualize in Grafana

**Components:**
- **Calorie Tracker Integration** - Import data from fitness apps (Apple Health, Google Fit, MyFitnessPal)
- **Prometheus** - Metrics collection
- **Grafana** - Dashboard visualization
- **Custom Exporter** - Convert app data to Prometheus format

**Grafana Dashboards:**
- Daily calorie intake vs goal
- Weekly nutritional breakdown
- Monthly trends and alerts
- Integration with fitness goals

---

## System Requirements

- **Hardware:** Any x86-64 system with 4GB+ RAM (e.g., repurposed laptop, mini PC, or Single-Board Computer like a Raspberry Pi)
- **OS:** Ubuntu 20.04+ (recommended)
- **Network:** Stable internet connection
- **Storage:** 100GB+

---

## Security

**Encryption:**
- Tailscale provides end-to-end encryption
- All remote traffic encrypted
- Private VPN (no internet exposure)

**Access Control:**
- Change default passwords after setup
- Tailscale handles device authentication
- Private GitHub repo with token authentication

**Backups:**
- Git provides configuration backup
- Version history for rollbacks
- Data stored locally (under your control)

---

## Monitoring

### Check Cluster Health

```bash
# Overall status
kubectl get all

# Pod health
kubectl get pods --watch

# Resource usage
kubectl top nodes
kubectl top pods

# Flux status
flux get all

# View events
kubectl get events --sort-by='.lastTimestamp'
```

### Common Commands

```bash
# Restart a service
kubectl rollout restart deployment/navidrome

# View logs
kubectl logs -f deployment/navidrome

# Describe pod
kubectl describe pod <pod-name>

# Force reconciliation
flux reconcile kustomization flux-system --with-source
```

---

## Troubleshooting

### Pod Won't Start
```bash
kubectl describe pod <pod-name>
# Check Events section for error details
```

### Can't Access Service
```bash
kubectl get svc
# Verify service and ports are correct
```

### Changes Not Applied
```bash
flux reconcile kustomization flux-system --with-source
flux logs --all-namespaces --follow
```

---

## Documentation

- **[docs/architecture.md](docs/architecture.md)** - System design and data flow

---

## License

MIT License - See LICENSE file for details