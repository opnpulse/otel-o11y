- Deploy ACE. 
- Configure KubeDB with `global.featureGates.ClickHouse=true`. 
- Import spoke cluster in ACE UI.
- Install Minio on ACE cluster.
```shell
kubectl create ns minio

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./private.key -out ./public.crt -subj "/CN=localhost/O=kubedb" -addext "subjectAltName=DNS:localhost,DNS:minio.minio.svc.cluster.local,DNS:minio.local"
kubectl create secret generic tls-ssl-minio -n minio --from-file=private.key --from-file=public.crt

helm repo add minio-comm https://charts.min.io/
helm upgrade -i minio --namespace minio --create-namespace --set rootUser=rootuser,rootPassword=rootpass123 minio-comm/minio -f ./minio/minio-values.yaml
```

- Create `test` bucket in minio.
```shell
kubectl port-forward -n minio svc/minio-console 9001:9001

username: `rootuser`
password: `rootpass123`
```

- Deploy Clickhouse on ACE cluster.
```shell
kubectl create secret generic -n monitoring my-config-xml --from-file=./clickhouse/custom-config.xml
kubectl create secret generic tls-ssl-minio -n monitoring --from-file=private.key --from-file=public.crt
kubectl apply -f ./clickhouse/clickhouse.yaml
```

- Deploy Thanos operator on ACE cluster.
```shell
kubectl apply -f https://raw.githubusercontent.com/appscode-cloud/installer/refs/heads/master/charts/license-proxyserver/crds/monitoring.coreos.com_servicemonitors.yaml
kubectl apply -f https://raw.githubusercontent.com/ops-center/alerts/refs/heads/master/charts/cassandra-alerts/crds/monitoring.coreos.com_prometheusrules.yaml

kubectl create ns thanos-operator-system
kubectl apply -f ./thanos-operator/crds
kubectl create secret generic tls-ssl-minio -n thanos-operator-system --from-file=private.key --from-file=public.crt
kubectl apply -f ./thanos-operator/thanos-operator.yaml
```

- Deploy Tenant Operator on ACE cluster.
```shell
kubectl apply -f ./tenant-operator/crds/
kubectl apply -f ./tenant-operator/manager.yaml

kubectl apply -f ./tenant-operator/thanos-object-storage.yaml
kubectl apply -f ./tenant-operator/thanos-deployment.yaml
```

- Expose Thanos receiver and Clickhouse
```shell
kubectl apply -f ./ingress/thanos-ingress.yaml
kubectl patch cm ace-ingress-tcp -n ace --type merge -p '{"data":{"9000":"monitoring/ch:9000"}}'
```

- Edit `ace-ingress` service from ace namespace.
```shell
  - name: 9000-tcp
    port: 9000
    protocol: TCP
    targetPort: 9000

```

- Go to imported cluster. Change `clusterName`, `prometheusremotewrite.endpoint`, `clickhouse.endpoint`, `clickhouse.password`, `attributes/example.actions.value` from /otel/values.yaml file.
Also change the password in /otel/ch-auth.yaml.

```shell
kubectl apply -f ./otel/target-allocator-rbac.yaml
kubectl apply -f ./otel/ch-auth.yaml

helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade -i opentelemetry-kube-stack -n monitoring open-telemetry/opentelemetry-kube-stack \
--set opentelemetry-operator.admissionWebhooks.certManager.enabled=false \
--set admissionWebhooks.autoGenerateCert.enabled=true \
--values=./otel/values.yaml   
```

- Now go to the ACE UI. Create a client org with name `tenant1`.
- Apply this yaml in ACE cluster. [you need to change the spec.tenantID to the client-org name]
```shell
kubectl apply -f ./tenant-operator/tenant.yaml
```

- install prom-label-proxy on spoke/imported clusters

```
helm upgrade -install prom-label-proxy prometheus-community/prom-label-proxy --set image.pullPolicy=Always --set image.repository=rokibulhasan114/prom-label-proxy --set image.tag=v0.0.1 --set config.upstream=https://10.2.0.187/querier --set config.label=tenant_id --namespace=monitoring
```
```
env:
- name: LOGS_URL
  value: https://10.2.0.82/logs
- name: TRACES_URL
  value: https://10.2.0.82/traces
- name: PLATFORM_APISERVER_DOMAIN
  value: https://10.2.0.88/

```