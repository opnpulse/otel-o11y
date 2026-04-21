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

**Settings → TelemetryStack → Create new stack**

---

## Step 6: Enable the OpenTelemetry Feature on a Spoke Cluster

1. Import a new cluster as a **spoke**.
2. Go to the cluster's **Observability** feature set.
3. Enable the **`appscode-otel-stack`** feature.

---

## Step 7: Create a Client Organization

From the **Site Administration** page, create a new client organization.