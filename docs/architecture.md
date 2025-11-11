# Homelab Architecture

## System Overview

This document explains how all components work together in the homelab setup.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Laptop using ubunut                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            Docker Desktop                           │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │      Kubernetes Cluster                       │ │  │
│  │  │                                                │ │  │
│  │  │  ┌──────────────────────────────────────────┐ │ │  │
│  │  │  │  Flux (GitOps Controller)               │ │ │  │
│  │  │  │  - Watches GitHub repo                  │ │ │  │
│  │  │  │  - Applies YAML manifests               │ │ │  │
│  │  │  │  - Keeps cluster in desired state       │ │ │  │
│  │  │  └──────────────────────────────────────────┘ │ │  │
│  │  │                                                │ │  │
│  │  │  ┌──────────────────────────────────────────┐ │ │  │
│  │  │  │  Navidrome Pod                          │ │ │  │
│  │  │  │  - Music server container               │ │ │  │
│  │  │  │  - Reads: C:\Users\username\Music          │ │ │  │
│  │  │  │  - Listens: Port 4533                   │ │ │  │
│  │  │  └──────────────────────────────────────────┘ │ │  │
│  │  │                                                │ │  │
│  │  │  ┌──────────────────────────────────────────┐ │ │  │
│  │  │  │  Syncthing Pod                          │ │ │  │
│  │  │  │  - Note syncing container               │ │ │  │
│  │  │  │  - Syncs: C:\Users\username\Obsidian       │ │ │  │
│  │  │  │  - Listens: Port 8384                   │ │ │  │
│  │  │  └──────────────────────────────────────────┘ │ │  │
│  │  │                                                │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │                                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Local Folders:                                            │
│  - C:\Users\username\Music\     (mounted to /music)           │
│  - C:\Users\username\Obsidian\  (mounted to /syncthing)       │
└─────────────────────────────────────────────────────────────┘
         ▲                                              │
         │                                              │
         │ (1) Watches GitHub                           │ (2) Exposes services
         │                                              │
         │                                              ▼
    ┌────────────────────┐              ┌──────────────────────────┐
    │  GitHub Repository │              │  Your Devices            │
    │                    │              │  - Main PC (via TS)      │
    │ - Flux manifests   │              │  - Phone (via TS)        │
    │ - Helm values      │              │  - Laptop (local)        │
    │ - K8s configs      │              │                          │
    └────────────────────┘              └──────────────────────────┘
                                                  ▲
                                                  │ (3) Tailscale VPN
                                                  │
                                        ┌─────────────────┐
                                        │  Tailscale      │
                                        │  (Encrypted     │
                                        │   tunnel)       │
                                        └─────────────────┘
```

---

## Component Breakdown

### 1. Docker Desktop & Kubernetes

**Purpose:** Container runtime and orchestration

**What it does:**
- Runs containers (Navidrome, Syncthing)
- Manages networking between containers
- Handles port mapping (internal to external)
- Restarts failed containers automatically

**How it works:**
```
Windows Laptop
    └── Docker Desktop
        └── Kubernetes Cluster
            ├── kube-system namespace (system components)
            ├── flux-system namespace (Flux controllers)
            └── default namespace (your apps)
```

**Why Kubernetes?**
- Declarative: You specify desired state, K8s makes it happen
- Self-healing: Restarts failed pods automatically
- Scalable: Can add/remove replicas easily
- Standard: Same setup works on any OS

---

### 2. Flux (GitOps Controller)

**Purpose:** Automatically deploy applications from Git

**How it works:**

```
┌──────────────┐
│  GitHub Repo │  (You push changes)
└──────┬───────┘
       │
       │ (Flux polls every 1 minute)
       ▼
┌────────────────────┐
│  Flux Controller   │
│  (running in K8s)  │
│                    │
│ 1. Fetch repo      │
│ 2. Parse YAML      │
│ 3. Apply to K8s    │
│ 4. Reconcile       │
└────────────────────┘
       │
       ▼
┌────────────────────┐
│  Kubernetes Cluster│
│  - Creates pods    │
│  - Runs containers │
└────────────────────┘
```

**Key concept: Reconciliation Loop**

Flux continuously asks: "Does my cluster match what's in Git?"

- **Yes?** Do nothing
- **No?** Apply Git configs to make it match

This means:
- Manual changes to pods are reverted (desired state wins)
- All changes go through Git (version history)
- Easy rollback (revert commit, Flux applies old version)

---

### 3. Helm (Package Manager)

**Purpose:** Reusable Kubernetes application templates

**Without Helm:**
```yaml
# You'd write 50+ lines of raw K8s YAML for each app
apiVersion: v1
kind: Pod
...
apiVersion: v1
kind: Service
...
apiVersion: v1
kind: ConfigMap
...
```

**With Helm:**
```yaml
# One HelmRelease, Helm expands it to full config
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: navidrome
spec:
  chart:
    spec:
      chart: navidrome
      version: "6.7.6"
  values:
    image:
      repository: deluan/navidrome
      tag: "0.58.5"
```

**How Helm integrates with Flux:**

```
Git Repository
├── helmrepository.yaml (where to find charts)
└── helmrelease.yaml (what chart + values)
        │
        ▼
Flux reads HelmRelease
        │
        ▼
Helm Controller (part of Flux)
        │
        ├─ Fetches chart from HelmRepository
        ├─ Merges values.yaml
        ├─ Renders templates to K8s YAML
        │
        ▼
Applies to Kubernetes
        │
        ▼
Pod running (Navidrome container)
```

---

### 4. Navidrome (Music Server)

**Architecture:**

```
┌─────────────────────────────────────┐
│  Navidrome Container (Linux)        │
│                                     │
│  ┌─────────────────────────────┐   │
│  │  Navidrome Application      │   │
│  │  - Web server (port 4533)   │   │
│  │  - Music indexer            │   │
│  │  - Streaming engine         │   │
│  └─────────────────────────────┘   │
│           │                         │
│           ▼ (reads)                │
│  ┌─────────────────────────────┐   │
│  │  /music folder (mounted)    │   │
│  │  - FLAC files               │   │
│  │  - MP3 files                │   │
│  │  - Metadata cache           │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
        ▲
        │ (mount)
        │
  Windows Laptop
  C:\Users\username\Music\
```

**Data Flow:**

```
User opens browser
        │
        ▼
http://localhost:30533
        │
        ▼
K8s routes to Navidrome pod:port 4533
        │
        ▼
Navidrome reads /music folder
        │
        ▼
Returns music list + streams audio
```

---

### 5. Syncthing (Note Sync)

**Architecture:**

```
Obsidian Vault on Laptop
C:\Users\username\Obsidian\
        │
        ▼
Syncthing Container (K8s)
        │
        ├─ Watches for file changes
        ├─ Syncs to other devices
        └─ Receives syncs from other devices
        │
        ▼
Syncthing on Phone
Syncthing on Main PC
```

**Sync Process:**

```
You edit note on Main PC
        │
        ▼
Syncthing detects change
        │
        ▼
Sends to Syncthing container (via Tailscale)
        │
        ▼
Container syncs to all connected devices
        │
        ▼
Phone/Laptop automatically updates
```

---

### 6. Tailscale (Remote Access)

**Purpose:** Secure VPN to access services from anywhere

**How it works:**

```
┌─────────────────────────────────────────┐
│  Tailscale Network (Encrypted Mesh)     │
│                                         │
│  Laptop (100.67.166.18)                 │
│  Main PC (100.66.186.111)               │
│  Phone (100.x.x.x)                      │
│                                         │
│  All connected via encrypted tunnel     │
└─────────────────────────────────────────┘
```

**Remote Access Flow:**

```
You on Main PC (anywhere)
        │
        ▼
Open browser: http://100.67.166.18:30533
        │
        ▼
Request travels through Tailscale tunnel
        │
        ▼
Reaches Laptop securely
        │
        ▼
K8s routes to Navidrome
        │
        ▼
Music streams back through tunnel
```

---

## Data Flow Examples

### Example 1: Playing Music Locally

```
1. You: Open http://localhost:30533 in browser
2. Browser: Sends request to localhost:30533
3. K8s: Routes to Navidrome service on port 4533
4. Navidrome: Reads /music folder
5. Navidrome: Returns list of songs
6. You: Click a song
7. Navidrome: Streams audio file to browser
8. Browser: Plays music
```

### Example 2: Adding Music

```
1. You: Copy new album to C:\Users\username\Music\
2. Windows: File appears in music folder
3. Navidrome: Detects new files (next scan)
4. Navidrome: Indexes metadata (artist, title, etc)
5. Navidrome: Adds to database
6. You: See new album in web UI
7. You: Play songs
```

### Example 3: Deploying a Change via Git

```
1. You: Edit clusters/my-laptop/apps/navidrome/helmrelease.yaml
2. You: git commit + git push to GitHub
3. GitHub: Updates repo
4. Flux: Polls GitHub (every 1 minute)
5. Flux: Detects change
6. Flux: Reads new helmrelease.yaml
7. Helm Controller: Fetches Navidrome chart
8. Helm: Renders templates with new values
9. K8s: Applies new configuration
10. K8s: Updates running pod if needed
11. You: See changes (new port, new settings, etc)
```

### Example 4: Accessing Music Remotely

```
1. You: On main PC, open Tailscale app
2. Tailscale: Connects to VPN
3. You: Navigate to http://100.67.166.18:30533
4. Request: Goes through Tailscale encrypted tunnel
5. Laptop: Receives request via tunnel
6. K8s: Routes to Navidrome
7. Navidrome: Returns music list
8. You: Stream music through encrypted tunnel
```

---

## Folder Structure Explained

```
clusters/my-laptop/
├── flux-system/                  # Flux system config (auto-generated)
│   ├── gotk-components.yaml     # Flux controllers
│   ├── gotk-sync.yaml           # Git sync source
│   └── kustomization.yaml       # Ties them together
│
└── apps/                         # Your applications
    ├── kustomization.yaml       # References navidrome + obsidian-sync
    │
    ├── navidrome/               # Music server
    │   ├── helmrepository.yaml  # Where to find chart
    │   ├── helmrelease.yaml     # How to deploy
    │   └── kustomization.yaml   # Package for Kustomize
    │
    └── obsidian-sync/           # Note sync
        ├── helmrepository.yaml
        ├── helmrelease.yaml
        └── kustomization.yaml
```

**How Kustomize ties it together:**

```yaml
# clusters/my-laptop/kustomization.yaml
resources:
  - flux-system
  - apps

# This tells K8s: "Apply both flux-system and apps folders"
```

---

## Why This Architecture?

### 1. Declarative (not imperative)
- You declare desired state in YAML
- System makes it happen automatically
- No manual commands needed

### 2. Version Controlled
- All changes in Git
- Full history and rollback capability
- Easy to review what changed

### 3. Reproducible
- Same config on different laptops = identical setup
- Easy onboarding for new team members
- Disaster recovery is just `git clone + flux bootstrap`

### 4. Self-Healing
- Pod crashes? K8s restarts it
- Manual config change? Flux reverts it
- Infrastructure as Code guarantees consistency

### 5. Scalable
- Add more services by adding folders
- Each service independently managed
- Easy to add/remove without affecting others

---

## State Diagram

```
GitHub Repository (Source of Truth)
        ▲
        │ (5) You push changes
        │
        └─────────────────┐
                          │
                          │ (1) Flux polls
                          ▼
                    Flux Controller
                          │
                          │ (2) Compares desired vs actual
                          ▼
                    Is cluster state
                    matching Git?
                     /        \
                   YES         NO
                    │           │
                    ▼           ▼
            Do nothing    Apply changes
                    │           │
                    └─────┬─────┘
                          │
                          ▼
                  Kubernetes Cluster
                  (running services)
                          │
                          │ (4) State changes
                          │ (pod crashes, etc)
                          ▼
                   Back to comparison
```

---

## Performance Considerations

### Resource Usage
- **Kubernetes**: ~500MB RAM
- **Flux**: ~100MB RAM
- **Navidrome**: ~200MB RAM (varies with library size)
- **Syncthing**: ~150MB RAM

### Network
- **Flux polling**: ~1MB per check (every 1 minute)
- **Music streaming**: Depends on bitrate (128kbps = ~1MB/min)
- **Syncthing**: Only syncs changes (efficient)

### Storage
- **K8s overhead**: ~1GB
- **Music library**: Depends on your files
- **Syncthing metadata**: ~100MB per vault

---

## Security Considerations

### Isolation
- Containers run in isolation from Windows
- Services communicate via K8s networking
- Each pod has its own filesystem

### Access Control
- Local: Anyone on laptop can access (unencrypted)
- Remote: Tailscale provides encryption + authentication
- No internet exposure (private VPN only)

### Credentials
- Default passwords should be changed (admin/admin)
- Tailscale handles authentication automatically
- Git repo is private (authentication via token)

---

## Troubleshooting Guide

### "Pod won't start"
```
Probable cause: Container image download failed or config error
Solution: kubectl describe pod <name> → check Events section
```

### "Can't access service locally"
```
Probable cause: Service port mapping incorrect or pod crashed
Solution: kubectl get svc → check ports, kubectl logs → check errors
```

### "Changes from Git not applied"
```
Probable cause: Flux not running or Git sync failed
Solution: flux reconcile kustomization flux-system --with-source
```

### "Music files not showing up in Navidrome"
```
Probable cause: File mounting issue or scan not triggered
Solution: Verify files exist, restart pod, trigger manual scan in UI
```

---

## Future Extensibility

This architecture easily extends to:

- **Add more services**: Just create new app folder + helmrelease.yaml
- **Scale horizontally**: Add more pods/replicas
- **Multi-cluster**: Bootstrap Flux to multiple laptops
- **Advanced networking**: Ingress controllers, load balancing
- **Monitoring**: Prometheus + Grafana via same pattern
- **Backups**: Automated snapshots of volumes

All following the same Git → Flux → K8s pattern.