**Phase 1: Foundation (Get K8s Running)**
1. Install Docker Desktop on old laptop
2. Enable Kubernetes in Docker Desktop settings
3. Verify it works: `kubectl cluster-info`
4. Goal: Have a working local K8s cluster

**Phase 2: Git Setup**
5. Create a Git repository (local or GitHub)
6. Create basic folder structure for your configs
7. Make your first commit
8. Goal: Have version control ready

**Phase 3: Install Flux on Laptop**
9. Install Flux CLI with `choco install flux` or `winget install flux`
10. Verify Flux is installed: `flux version`
11. Bootstrap Flux to your Git repo
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=music-server-gitops \
  --branch=main \
  --path=clusters/my-laptop \
  --personal
Go to github.com > Settings > Developer settings > Personal access tokens > Tokens (classic) > Generate new token
Give it permissions: repo and workflow
Copy the token, paste it when the terminal asks
12. Verify Flux is running: `flux get all` and `kubectl get pods`
13. Goal: Flux is watching your Git repo

**Phase 4: Deploy Navidrome**
1.  Install Helm CLI on laptop with `winget install Helm.Helm`
2.  Verify Helm is installed: `helm version`
3.  Create Navidrome Helm values file
4.  Create Flux HelmRelease manifest
5.  Push to Git, watch Flux deploy it
6.  Goal: Navidrome running in K8s

**Phase 5: Storage & Access**
18. Configure persistent storage for music folder
19. Point Navidrome to your music files
20. Access from laptop: `http://localhost:4533`
21. Goal: Actually streaming music

**Phase 6: Remote Access**
22. Install Tailscale on laptop
23. Install Tailscale on phone/main PC
24. Access music server remotely via Tailscale IP
25. Goal: Stream music from anywhere