
# fuse
# Install k3s 
k3d cluster create dev-cluster \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"

kubectl -n kube-system get pods | grep traefik
# If default traefik is running, disable by deleting or via k3d args

# Create traefik namespace
kubectl create namespace traefik

# Add Helm repo and update
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik
helm install traefik traefik/traefik --namespace traefik

# Install Jenkins on k3s
helm repo add jenkins https://charts.jenkins.io
helm repo update
kubectl create namespace jenkins
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.serviceType=NodePort \
  --set controller.nodePort=32000 \
  --set controller.admin.username=admin \
  --set controller.admin.password=admin123


#Ingress for jenkins
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    # Use Traefik ingress class
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


# Install Argocd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

kubectl rollout restart deployment argocd-server -n argocd





apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/frontend-entry-points: web
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    argocd.argoproj.io/insecure: "true"
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


### Image updater
kubectl create namespace argocd-image-updater

#kubectl apply -n argocd-image-updater -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml


#Sync docker hub
kubectl create secret generic dockerhub-creds \
  --from-literal=username=binod1243 \
  --from-literal=password=<docker hub password>\
  -n argocd
# update helm image tag
kubectl create secret generic git-creds \
  --from-literal=username=binod132 \
  --from-literal=password=<git PAT> \
  -n argocd
kubectl create secret docker-registry dockerhub-creds-image-2 \
  --docker-server=https://registry-1.docker.io \
  --docker-username=binod1243 \
  --docker-password='<docker hub pass>' \
  --docker-email=adhikaribinod132@gmail.com \
  -n argocd-image-updater

#Role binding
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
    namespace: argocd-image-updater

#for private docker hub
kubectl create configmap argocd-image-updater-registries \
  --namespace argocd-image-updater \
  --from-literal=registries.conf='registries:
  - name: Docker Hub
    prefix: docker.io
    credentials: secret:argocd/dockerhub-creds-image
    default: true'

#Update config
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd-image-updater
  labels:
    app.kubernetes.io/name: argocd-image-updater-config
    app.kubernetes.io/part-of: argocd-image-updater
data:
  registries.conf: |
    registries:
      - name: Docker Hub
        api_url: https://index.docker.io/v1/
        prefix: docker.io
        credentials: secret:argocd-image-updater/dockerhub-creds-image
        default: true

kubectl apply -f image-updater-config.yaml
kubectl rollout restart deployment argocd-image-updater -n argocd-image-updater
kubectl logs deployment/argocd-image-updater -n argocd-image-updater -f



[](../../../../wp-content/uploads/2023/07/image-62.png)


## Monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

