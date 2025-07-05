### Install Minio:

```bash
helm repo add minio https://operator.min.io/
````
```bash
helm upgrade --install --namespace minio-operator \
  --create-namespace minio minio/operator \
  --set operator.replicaCount=1 \
  --wait
```


```bash
helm upgrade --install --namespace minio --create-namespace minio-tenant minio/tenant \
  --set tenant.pools[0].servers=1 \
  --set tenant.pools[0].volumesPerServer=1 \
  --set tenant.pools[0].size=1Gi \
  --set tenant.certificate.requestAutoCert=false \
  --set tenant.pools[0].name="default" \
  --set tenant.buckets[0].name="test" \
  --wait
```

#### Accessing MinIO

- Access Key: `minio`
- Secret Key: `minio123`
- Endpoint: `http://minio.minio.svc.cluster.local:80`

### Setup ClickHouse

#### Install KubeDB

```bash
helm install kubedb oci://ghcr.io/appscode-charts/kubedb \
 --version v2025.3.24 \
 --namespace kubedb --create-namespace \
 --set-file global.license= /path/to/the/license.txt \
 --set global.featureGates.ClickHouse=true \
 --wait --burst-limit=10000 --debug
```

#### Create Namespace

```bash
kubectl create namespace monitoring
```

#### Create Custom Config

```bash
kubectl create secret generic -n monitoring my-config-xml --from-file=./clickhouse/custom-config.xml

```
#### Deploy ClickHouse DB

```bash
kubectl apply -f ./clickhouse/clickhouse.yaml
```

#### Create Schema

```bash
kubectl exec -it -n monitoring ch-0 -- bash
chi-clickhouse-cluster-0-0-0:/# clickhouse-client
```
```sql
CREATE TABLE default.otel_logs
(
        `Timestamp` DateTime64(9) CODEC(Delta(8), ZSTD(1)),
        `TraceId` String CODEC(ZSTD(1)),
        `SpanId` String CODEC(ZSTD(1)),
        `TraceFlags` UInt32 CODEC(ZSTD(1)),
        `SeverityText` LowCardinality(String) CODEC(ZSTD(1)),
        `SeverityNumber` Int32 CODEC(ZSTD(1)),
        `ServiceName` LowCardinality(String) CODEC(ZSTD(1)),
        `Body` String CODEC(ZSTD(1)),
        `ResourceSchemaUrl` String CODEC(ZSTD(1)),
        `ResourceAttributes` Map(LowCardinality(String), String) CODEC(ZSTD(1)),
        `ScopeSchemaUrl` String CODEC(ZSTD(1)),
        `ScopeName` String CODEC(ZSTD(1)),
        `ScopeVersion` String CODEC(ZSTD(1)),
        `ScopeAttributes` Map(LowCardinality(String), String) CODEC(ZSTD(1)),
        `LogAttributes` Map(LowCardinality(String), String) CODEC(ZSTD(1)),
        INDEX idx_trace_id TraceId TYPE bloom_filter(0.001) GRANULARITY 1,
        INDEX idx_res_attr_key mapKeys(ResourceAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_res_attr_value mapValues(ResourceAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_scope_attr_key mapKeys(ScopeAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_scope_attr_value mapValues(ScopeAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_log_attr_key mapKeys(LogAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_log_attr_value mapValues(LogAttributes) TYPE bloom_filter(0.01) GRANULARITY 1,
        INDEX idx_body Body TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1
)
ENGINE = MergeTree
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SeverityText, toUnixTimestamp(Timestamp), TraceId)
TTL toDateTime(Timestamp) + INTERVAL 1 MINUTE TO VOLUME 'cold'
SETTINGS storage_policy = 'tiered';
```


### Deploy Thanos

#### Create Namespace

```bash
kubectl create namespace thanos
```

#### Deploy Thanos

```bash
kubectl -n thanos create secret generic thanos-objstore-config \
  --from-file=objstore.yml=./thanos/s3.yaml
```

```bash
helm upgrade -i thanos oci://registry-1.docker.io/bitnamicharts/thanos \
  --version 16.0.4 \
  --namespace thanos --create-namespace \
  --values=./thanos/values.yaml
```

#### Query metrics via Thanos Querier

```bash
kubectl port-forward svc/thanos-query -n monitoring 9090:9090
```
Visit http://localhost:9090 and execute a test query (e.g., up).

### Deploy Opentelmetry Stack

```
kubectl apply -f targetallocator-rbac.yaml
```

```bash
helm install opentelemetry-kube-stack -n monitoring open-telemetry/opentelemetry-kube-stack \
--set opentelemetry-operator.admissionWebhooks.certManager.enabled=false \
--set admissionWebhooks.autoGenerateCert.enabled=true \
--values=./values.yaml
```
