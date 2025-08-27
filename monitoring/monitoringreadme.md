helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f values.yaml



helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Loki for logs
helm install loki grafana/loki-stack --namespace monitoring

# Tempo for traces
helm install tempo grafana/tempo --namespace monitoring


#########
