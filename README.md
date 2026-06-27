# BookMyShow — DevOps CI/CD Pipeline

A movie-ticketing web app deployed through a fully automated CI/CD pipeline on AWS, going from a `git push` to a live, monitored, rollback-capable deployment on Kubernetes — with zero manual steps in between.

**Live app:** http://bookmyshow.hsmo.online

---

## What this actually is

This started as a React frontend with a half-written CI/CD setup. I rebuilt the deployment pipeline end-to-end and got it running on real AWS infrastructure: Jenkins triggers on every push, runs code-quality and security scans, builds a production Docker image, pushes it to ECR, and deploys it to an EKS cluster via Ansible — with Prometheus/Grafana monitoring the result and a tested rollback path if anything goes wrong.

The interesting part isn't the list of tools. It's that almost nothing worked on the first try, and the fixes required actually understanding *why* — not just copy-pasting a different command. A few examples below.

## Architecture

```
 Developer push (main branch)
        │
        ▼
   GitHub Webhook ──────────────────────────────┐
        │                                       │
        ▼                                       │
   ┌─────────────────────────────────────────┐  │
   │  Jenkins (EC2)                          │  │
   │  ── Checkout                            │  │
   │  ── SonarQube scan (code quality)       │  │
   │  ── Trivy scan (filesystem)             │  │
   │  ── Docker build (multi-stage, nginx)   │  │
   │  ── Trivy scan (image)                  │  │
   │  ── Push to ECR (tagged build-N)        │  │
   │  ── Ansible playbook → deploy           │  │
   └─────────────────────────────────────────┘  │
        │                                       │
        ▼                                       │
   AWS EKS Cluster ◄──────────────────────────────
   ┌─────────────────────────────┐
   │ Deployment (rolling update) │
   │ Service (LoadBalancer)      │
   │ HPA / PodDisruptionBudget   │
   └─────────────────────────────┘
        │
        ▼
   Route 53 → bookmyshow.hsmo.online

   Prometheus + Grafana (Helm) watch the cluster throughout
```

**Why an EC2 box *and* a separate EKS cluster?** The EC2 instance is purely the control plane — it runs Jenkins, SonarQube, and the CLI tooling (`kubectl`, `aws`, `ansible`, `docker`). It doesn't serve any traffic. The actual application runs on EKS, which is the real, managed Kubernetes cluster Jenkins deploys to.

## Tech stack

| Layer | Tool |
|---|---|
| Source control / triggers | Git, GitHub Webhooks |
| CI/CD orchestration | Jenkins |
| Code quality | SonarQube |
| Security scanning | Trivy (filesystem + image) |
| Containerization | Docker (multi-stage build) |
| Image registry | AWS ECR |
| Orchestration | AWS EKS (Kubernetes 1.34) |
| Configuration-managed deploy | Ansible |
| Monitoring | Prometheus + Grafana (kube-prometheus-stack) |
| DNS | AWS Route 53 |

## Pipeline stages

1. GitHub webhook fires on push to `main`
2. Jenkins checks out the latest code
3. SonarQube analyzes the source for quality issues
4. Trivy scans the filesystem for known vulnerabilities
5. Docker builds a production image (React build → served by nginx)
6. Trivy scans the built image
7. Image is tagged `build-<N>` and pushed to ECR (alongside `latest`)
8. Ansible playbook deploys the new tag to EKS
9. Kubernetes performs a zero-downtime rolling update
10. Prometheus/Grafana continuously monitor the result

## Problems I actually hit (and how I fixed them)

A few of the more interesting ones — full root-cause writeups are in [`docs/PROJECT_DOCUMENTATION.md`](docs/PROJECT_DOCUMENTATION.md).

- **Container crashed on startup with a missing `chokidar` module.** Turned out to be an optional npm dependency that gets silently skipped in some Linux container builds. Fixed by explicitly installing it in the Dockerfile.

- **Pods stuck in `CrashLoopBackOff`, autoscaler making it worse, not better.** The app was running React's *development* server inside the container — memory-hungry and never meant for production. The autoscaler kept spinning up more pods to compensate, which only added more memory pressure. Fixed by rewriting the Dockerfile as a proper multi-stage build: compile the React app, then serve the static output with nginx. Memory footprint dropped from 500MB+ per pod to under 20MB.

- **LoadBalancer stuck at `<pending>` indefinitely.** `kubectl describe svc` showed AWS rejecting it: `unsupported load balancer affinity: ClientIP`. The Service had sticky-session config left over from a template, which AWS's load balancer doesn't support. Removed it — the app is stateless static files, so it was never needed anyway.

- **Jenkins could run `kubectl` with full IAM permissions and still got `the server has asked for the client to provide credentials`.** IAM permissions and Kubernetes RBAC are two separate authorization systems on EKS. Had to explicitly create an EKS Access Entry granting the Jenkins IAM role cluster-level access — IAM alone doesn't grant you anything inside the cluster.

## Rollback

Every build is tagged with its Jenkins build number (`build-<N>`), not just `latest`. Rolling back to any previous version is one command, using the same Ansible playbook the pipeline itself uses:

```bash
ansible-playbook deploy.yml -e "image_tag=build-7"
```

Verified in practice — see the documentation for the full before/after `kubectl` output.

## Repo structure

```
.
├── bookmyshow-app/        # React application source + Dockerfile
├── k8s/                   # Kubernetes manifests (Deployment, Service, HPA, etc.)
├── deploy.yml             # Ansible playbook used for EKS deployment
├── Jenkinsfile            # Pipeline definition
└── docs/
    ├── PROJECT_DOCUMENTATION.md   # Full build narrative, every bug + fix
    └── RUNBOOK.md                 # Step-by-step rebuild guide from scratch
```

## Running it yourself

The full rebuild guide — every command, in order, from a blank EC2 instance to a working pipeline — is in [`docs/RUNBOOK.md`](docs/RUNBOOK.md).

## What I'd do differently with more time

- Move session/state-sensitive resource limits onto a `kind`-based local test pass before touching the real cluster, to catch resource-starvation issues earlier and cheaper
- Use Helm for the application itself, not just the monitoring stack, for consistent templating
- Add a staging namespace with an automated smoke test gate before production rollout, rather than relying on manual verification
- Switch from Classic Load Balancer to an ALB Ingress Controller for proper path-based routing as the app grows
