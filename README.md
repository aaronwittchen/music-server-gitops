# Homelab Music Server Setup

Deployed a self-hosted music server on my old Windows laptop using Kubernetes, Flux, Git, and Navidrome.

## Tech Stack

### Kubernetes
- **What**: Container orchestration platform
- **Why**: Manages and runs your music server container
- **Implementation**: Docker Desktop (built-in K8s for Windows)
- **Setup**: Install Docker Desktop, enable K8s in settings

### Flux
- **What**: GitOps tool for Kubernetes
- **Why**: Automatically deploys your music server from Git, ensures it stays running, handles updates
- **How it works**: Watches your Git repo, applies any changes to K8s cluster
- **Setup**: Install flux CLI, bootstrap to a Git repository

### Git
- **What**: Version control system
- **Why**: Store all your deployment configurations, track changes, reproducibility
- **Options**: Local Git repo or GitHub/GitLab
- **Contents**: Navidrome Helm chart values, K8s manifests, Flux configuration

### Helm
- **What**: Package manager for Kubernetes
- **Why**: Pre-built, reusable configurations for Navidrome (don't have to write raw manifests)
- **How**: Helm charts contain templates + values, Flux applies them to K8s
- **Setup**: Install Helm CLI

### Navidrome
- **What**: Lightweight music server application
- **Why**: Low resource footprint (perfect for old laptop), web interface, streaming support
- **Storage**: Reads music from local folder
- **Access**: Web UI + mobile/desktop clients