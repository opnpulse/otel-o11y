# Observability Stack Setup Guide

## Prerequisites

- Clone or navigate to the [otel-o11y](https://github.com/opnpulse/otel-o11y) repository first

---

## Step 1: Create the Self-Hosted Installer

Set up a self-hosted installer for your environment.

---

## Step 2: Create the Observability Cluster

Create a new cluster and import it using the **Observability Cluster** cluster profile.

---

## Step 3: Install MinIO

Install MinIO on the observability cluster:

```bash
kubectl create ns minio

kubectl create secret generic tls-ssl-minio -n minio \
  --from-file=private.key \
  --from-file=public.crt

helm repo add minio-comm https://charts.min.io/

helm upgrade -i minio \
  --namespace minio \
  --create-namespace \
  --set rootUser=rootuser,rootPassword=rootpass123 \
  minio-comm/minio \
  -f ./minio/minio-values.yaml
```

---

## Step 4: Create the `telemetry` Bucket

Port-forward the MinIO console and create the bucket via the UI:

```bash
kubectl port-forward -n minio svc/minio-console 9001:9001
```

Open `http://localhost:9001` and log in:

| Field    | Value          |
|----------|----------------|
| Username | `rootuser`     |
| Password | `rootpass123`  |

Create a bucket named **`telemetry`**.

---

## Step 5: Create a TelemetryStack

In the observability cluster, navigate to:

**Settings → TelemetryStack → Create a new Telemetry stack**

Fill in the configuration sections as shown below:

**Configure Metrics (Compact, Store, Query, Router, Ingester):**

![Create TelemetryStack - Metrics config](images/Screenshot%20from%202026-05-25%2017-41-50.png)

**Ruler, Additional Config, and Configure Clickhouse:**

![Create TelemetryStack - Additional config](images/Screenshot%20from%202026-05-25%2017-41-58.png)

**Configure Clickhouse (Standalone / Create Topology), Configure S3, and Configure ID:**

![Create TelemetryStack - Clickhouse, S3, and ID config](images/Screenshot%20from%202026-05-25%2017-42-03.png)

---

## Step 6: Enable the OpenTelemetry Feature on a Spoke Cluster

1. Import a new cluster as a **spoke**.
2. `helm upgrade -i kubedb-perses-dashboards oci://ghcr.io/appscode-charts/kubedb-perses-dashboards -n monitoring --create-namespace --version=v2026.4.27`
3. Go to the cluster's **Observability** feature set.
4. Enable the **`appscode-otel-stack`** feature.

---

## Step 7: Create a Client Organization

From the **Site Administration** page, create a new client organization.