# mlflow-infra

Project: Mlops-infrastructure

Description

This repository contains a Helm chart to deploy MLflow (Bitnami reference chart) and ArgoCD Application manifests to automate deployments into a Kubernetes cluster.

Repository layout (key folders)

- `argocd-apps/` — ArgoCD Application manifests used by ArgoCD to sync:
  - `mlflow.yaml` — Application that deploys the `infra/mlflow` Helm chart using `my-values.yaml`.
  - `prom.yaml` — Application for `kube-prometheus-stack` (monitoring).

- `infra/mlflow/` — MLflow Helm chart with standard Bitnami templates and a custom `my-values.yaml`. Important subfolders:
  - `templates/` — Kubernetes manifests and templates (Deployment, Secrets, PVCs, Service, etc.).
  - `values.yaml` — full default configuration reference.
  - `my-values.yaml` — values file used by ArgoCD for this environment.

What this release does now

- In the current `my-values.yaml` the chart deploys internal services:
  - `postgresql.enabled: true` — PostgreSQL is deployed as a subchart inside the release.
  - `minio.enabled: true` — MinIO is used as the object store for artifacts.

- Because of that, the ArgoCD Application (`argocd-apps/mlflow.yaml`) does not point to an external DB or S3. The chart templates contain logic that will create Secrets only when you explicitly configure use of external services or disable the internal subcharts.

When the release is bound to external services

- To use an external PostgreSQL:
  1) set in `my-values.yaml`:
     - `postgresql.enabled: false`
     - fill `externalDatabase.host`, `externalDatabase.user`, `externalDatabase.password` (or provide `externalDatabase.existingSecret`).

- To use an external S3 / GCS / Azure Blob instead of MinIO:
  1) set `minio.enabled: false`
  2) configure `externalS3` (host, bucket, accessKeyID/secret or `existingSecret`), or `externalGCS` / `externalAzureBlob` accordingly.

- Templates create Secret objects only when values in `my-values.yaml` require them (for example, `externalS3.useCredentialsInSecret: true` together with `minio.enabled: false`).

How to validate locally what would be created

- Render the Helm templates (no cluster apply):

```bash
helm template mlflow ./infra/mlflow -f ./infra/mlflow/my-values.yaml
```

- Show only Secret objects from the rendered output:

```bash
helm template mlflow ./infra/mlflow -f ./infra/mlflow/my-values.yaml | yq eval '. | select(.kind=="Secret")' -
```

(Requires `yq` installed on your machine.)

- Dry-run the ArgoCD Application CR (client-side):

```bash
kubectl apply -f argocd-apps/mlflow.yaml --dry-run=client
```

Security notes

- Do not store plaintext passwords in `my-values.yaml`. Prefer `existingSecret` and manage secrets separately (SealedSecrets, ExternalSecrets, or ArgoCD repository credentials).
- If `argocd-apps/mlflow.yaml` uses an SSH git URL (`git@github.com:...`), ensure ArgoCD has the matching SSH key configured.

Options I can do next

- Prepare an example `my-values.yaml` that switches mlflow to an external PostgreSQL and external S3 (including `existingSecret` examples).
- Change `argocd-apps/mlflow.yaml` to use an HTTPS repo URL (to avoid requiring an SSH key in ArgoCD).
- Run `helm template` locally in this repository and extract the list of objects (Secrets, Deployments, Services) that will be created.

Summary

With the current values, ArgoCD will deploy MLflow together with an internal PostgreSQL and MinIO in the cluster — this is not a "configuration only" manifest; it's a full release including additional services. Choose whether you want to switch to external services or keep the internal components as-is.
