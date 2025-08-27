📘 CI/CD Pipeline Design: Jenkins + ArgoCD (GitOps)
🔧 Cluster Setup

Platform: Kubernetes K3s using k3d on MAC OS

Tools Installed:

Jenkins (CI) – for building, testing, and pushing container images

ArgoCD (CD) – for GitOps-based declarative deployment

Docker Registry (Docker Hub)

🧩 CI/CD Workflow Overview
✅ 1. Code Commit (Developer Pushes Code to GitHub)

Dev pushes changes dev branch.

Commit triggers Jenkins pipeline.

🔨 2. Jenkins CI Pipeline

Stage 1: Clone Repository

Jenkins checks out the latest code from GitHub.

Stage 2: Build Docker Image (Multi-Stage Build)

Uses a Dockerfile with multi-stage build:

First stage: Build the app using builder image node.

Second stage: Copy binaries into a minimal base image alpine nginx.

Reduces image size and removes build-time tools for security.

Stage 3: Run Unit/Integration Tests

Automated tests executed in Jenkins pipeline using testing frameworks.

Pipeline fails early if tests fail.

Stage 4: Push Docker Image

Tags image with date-based tag (e.g., 2025.08.27-37)

Pushes to registry Docker Hub

Stage 5: Update Helm Values via GitOps

ArgoCD Image Updator is used to update image TAG on helm values via automatic commit.

Git push triggers ArgoCD

🚀 3. ArgoCD CD Pipeline (GitOps)

ArgoCD watches Git repo (Helm chart paths):

/helm-microservice → chart

/dev/values.yaml, /uat/values.yaml → environment overrides

When Jenkins pushes a new image tag commit:
Argocd Image Updater sync new tag every 2 minitue. If new image tag is found, it push tag update on helm values repo.
ArgoCD automatically detects that change.

It pulls the updated chart + values.

It compares with live cluster state.

If drift is detected → it applies updates declaratively.

🛡️ Why Use Multi-Stage Docker Builds?
Benefit	Description
Smaller image size	Only includes runtime dependencies.
More secure	No compilers/tools in final image.
Faster deployment	Lightweight images pull faster in Kubernetes.
Better caching	Reusable build layers reduce rebuild time.
⚙️ GitOps with ArgoCD: Benefits
GitOps Principle	How ArgoCD Helps
Declarative state	All manifests stored in Git.
Version-controlled infra	Git history tracks all changes.
Automated sync	ArgoCD syncs Git → Cluster.
Rollback & Audit	Easy rollback via Git revert.
Separation of concerns	CI handles builds, CD handles delivery.
🧪 Summary of Pipeline Flow
