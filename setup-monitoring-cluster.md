- Import cluster in ui.
- Configure KubeDB with `global.featureGates.ClickHouse=true`. 
- Install Minio.
```shell
kubectl create ns minio

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./private.key -out ./public.crt -subj "/CN=localhost/O=kubedb" -addext "subjectAltName=DNS:localhost,DNS:minio.minio.svc.cluster.local,DNS:minio.local"
kubectl create secret generic tls-ssl-minio -n minio --from-file=private.key --from-file=public.crt

helm repo add minio-comm https://charts.min.io/
helm upgrade -i minio --namespace minio --create-namespace --set rootUser=rootuser,rootPassword=rootpass123 minio-comm/minio -f ./minio/minio-values.yaml
```

