# CI/CD Pipeline Implementation Report

---

## 1. Introduction

This document describes the implementation of a modern, automated CI/CD pipeline for a Kubernetes-based application using:

- **Jenkins** for Continuous Integration (CI)
- **ArgoCD** for Continuous Deployment (CD) via GitOps
- **Docker Hub** as the container registry
- **Helm** for Kubernetes package management
- **k3s (via k3d)** for lightweight local Kubernetes cluster

The design prioritizes secure, efficient builds and declarative, version-controlled deployments.

---

## 2. Cluster Setup

### Platform: K3s via `k3d` (on macOS)

```bash
k3d cluster create dev-cluster \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```
## Continuous Integration (CI) – Jenkins
### Jenkins Installation on Kubernetes
```bash
# Create namespace
kubectl create namespace jenkins

# Add Jenkins chart and install
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.serviceType=NodePort \
  --set controller.nodePort=32000 \
  --set controller.admin.username=admin \
  --set controller.admin.password=admin123
```
#### Access Jenkins
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: jenkins.fusemachine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```
### Jenkins pipeline
```bash
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      tty: true
      command:
        - sleep
      args:
        - infinity
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
  volumes:
    - name: kaniko-secret
      secret:
        secretName: dockercrednew
        items:
        - key: .dockerconfigjson
          path: config.json
"""
      defaultContainer 'kaniko'
    }
  }

  environment {
    imageNameDev = "binod1243/fuse"
    imageNameProd = "binod1243/fuse"
  }

  stages {
    stage('Build & Push - Dev') {
      when { expression { env.BRANCH_NAME ==~ /(dev)/ } }
      steps {
        script {
          def DATE_TAG = java.time.LocalDate.now().toString().replaceAll("-", ".")
          // Run Kaniko executor inside the running kaniko container
          container('kaniko') {
            sh """
              /kaniko/executor \
                --dockerfile=Dockerfile \
                --context=\$WORKSPACE \
                --destination=${imageNameDev}:${DATE_TAG}-${BUILD_NUMBER} \
                --destination=${imageNameDev}:dev-latest \
                --verbosity=info
            """
          }
        }
      }
    }

    stage('Build & Push - Prod') {
      when { expression { env.BRANCH_NAME ==~ /(main)/ } }
      steps {
        script {
          def DATE_TAG = java.time.LocalDate.now().toString().replaceAll("-", ".")
          container('kaniko') {
            sh """
              /kaniko/executor \
                --dockerfile=Dockerfile \
                --context=\$WORKSPACE \
                --destination=${imageNameProd}:${DATE_TAG}-${BUILD_NUMBER} \
                --destination=${imageNameProd}:latest \
                --verbosity=info
            """
          }
        }
      }
    }
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }
}
```
***Note: kaniko is used for docker cli inside kubernets pod***

## Continuous Deployment (CD) – ArgoCD
### ArgoCD Installation
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Enable insecure access for ease of local use
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

kubectl rollout restart deployment argocd-server -n argocd
```
### Access Argocd
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: argocd.fusemachine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```
## GitOps with ArgoCD Image Updater
### Install ArgoCD Image Updater
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml
```
### Setup Docker Hub and Git Credentials
```bash
kubectl create secret generic dockerhub-creds \
  --from-literal=username=binod1243 \
  --from-literal=password=<docker_hub_password> \
  -n argocd

kubectl create secret generic git-creds \
  --from-literal=username=binod132 \
  --from-literal=password=<git_PAT> \
  -n argocd
```
### Docker Registry for Private Images
```bash
kubectl create secret docker-registry dockerhub-creds-image-2 \
  --docker-server=https://registry-1.docker.io \
  --docker-username=binod1243 \
  --docker-password='<docker_hub_pass>' \
  --docker-email=adhikaribinod132@gmail.com \
  -n argocd
```
### Role Binding to access argocd
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-image-updater-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-application-controller
subjects:
  - kind: ServiceAccount
    name: argocd-image-updater
    namespace: argocd
```
### Image Updater Configuration
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd-image-updater
data:
  log.level: debug
  registries.conf: |
    registries:
      - name: Docker Hub
        api_url: https://registry-1.docker.io
        credentials: pullsecret:argocd/dockerhub-creds-image-2
        default: true
        defaultns: library
```
```bash
kubectl apply -f image-updater-config.yaml
kubectl rollout restart deployment argocd-image-updater -n argocd-image-updater
```
### Argocd Application
```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fuse-dev
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: myimage=binod1243/fuse
    argocd-image-updater.argoproj.io/myimage.update-strategy: alphabetical
    # argocd-image-updater.argoproj.io/binod1243.fuse.image-name: image.name
    # argocd-image-updater.argoproj.io/binod1243.fuse.image-tag: image.tag
    argocd-image-updater.argoproj.io/myimage.allow-tags: regexp:^\d{4}\.\d{2}\.\d{2}-\d+$
    argocd-image-updater.argoproj.io/myimage.pull-secret: pullsecret:argocd/dockerhub-creds-image-2
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-credentials: secret:argocd/git-creds
    argocd-image-updater.argoproj.io/git-credentials: secret:argocd/git-creds
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  source:
    repoURL: https://github.com/binod132/fuse
    targetRevision: main
    path: helm-microservice
    helm:
      valueFiles:
        - ../dev/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```
## CI/CD Pipeline Workflow

### ✅ Step 1: Developer Code Commit

- Developer pushes code to the `dev` branch of the GitHub repository.
- A GitHub webhook is configured to trigger the Jenkins pipeline upon a new commit.

---

### 🔨 Step 2: Jenkins CI Pipeline

#### Pipeline Stages:

1. **Clone Repository**
   - Jenkins checks out the latest code from GitHub.

2. **Multi-Stage Docker Build**
   - Dockerfile is structured with two stages:
     - **Stage 1:** Build the application using a `node` builder image.
     - **Stage 2:** Copy final production-ready build into a minimal `nginx:alpine` base image.

3. **Run Unit Tests**
   - Automated test scripts (unit and integration) are executed inside Jenkins.
   - Pipeline halts if tests fail, ensuring only valid code proceeds.

4. **Push Docker Image to Registry**
   - Image is tagged with a timestamp (e.g., `2025.08.28-42`) for version traceability.
   - Successfully built image is pushed to Docker Hub.

5. **GitOps Trigger**
   - [ArgoCD Image Updater](https://github.com/argoproj-labs/argocd-image-updater) checks for new image tags every 2 minutes.
   - On detecting a new image, it automatically commits the updated image tag to the Helm `values.yaml` file in the Git repository.

---

### 🚀 Step 3: ArgoCD CD Pipeline

- ArgoCD watches the Git repository for changes in:
  - `/helm-microservice` (Helm chart location)
  - `/dev/values.yaml` and `/uat/values.yaml` (environment-specific overrides)

- When a change is detected:
  - ArgoCD pulls the updated Helm chart and values.
  - It compares the **desired state** (from Git) with the **live state** in the Kubernetes cluster.
  - If drift is detected, ArgoCD applies the changes declaratively, ensuring cluster state matches the Git repository.

---



