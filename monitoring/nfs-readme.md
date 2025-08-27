nfs docker compose on monitoring
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nfs-server

172.21.0.7

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=172.21.0.7 \
  --set nfs.path=/ \
  --set storageClass.name=nfs-macos \
  --set storageClass.defaultClass=true

