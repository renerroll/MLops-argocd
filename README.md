# MLOps Infrastructure with ArgoCD and MLflow

Цей репозиторій містить MLOps інфраструктуру, розгорнуту на AWS EKS з використанням Terraform та ArgoCD.

## Структура проекту

```
├── argocd/                     # Terraform конфігурація для ArgoCD
│   ├── main.tf                 # Основна конфігурація Helm релізу ArgoCD
│   ├── variables.tf            # Змінні для Terraform
│   ├── outputs.tf              # Вихідні значення
│   ├── terraform.tf            # Конфігурація провайдерів
│   ├── backend.tf              # Локальний Terraform State
│   └── values/
│       └── argocd-values.yaml  # Helm values для ArgoCD
└── applications/
    └── application.yaml        # ArgoCD Application з MLflow ресурсами
```

## Розгортання

### 1. Розгортання ArgoCD через Terraform

```bash
cd argocd
terraform init
terraform plan
terraform apply
```

### 2. Отримання паролю ArgoCD

```bash
kubectl -n infra-tools get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

### 3. Доступ до ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n infra-tools 8080:80
```

ArgoCD UI: http://localhost:8080

- Логін: `admin`
- Пароль: отриманий на кроці 2

### 4. Розгортання MLflow через GitOps

```bash
kubectl apply -f application/application.yaml
```

### 5. Доступ до MLflow UI

```bash
kubectl port-forward -n mlflow svc/mlflow-service 5000:5000
```

MLflow UI: http://localhost:5000

## Основні команди

### Моніторинг стану

```bash
# Перевірка статусу ArgoCD додатків
kubectl get applications -n infra-tools

# Перевірка статусу MLflow
kubectl get all -n mlflow

# Перегляд логів MLflow
kubectl logs -n mlflow deployment/mlflow-server --tail=50
```

### Керування додатками

```bash
# Примусова синхронізація ArgoCD
kubectl annotate app mlflow-app -n infra-tools argocd.argoproj.io/refresh=hard --overwrite

# Перезапуск MLflow deployment
kubectl rollout restart deployment/mlflow-server -n mlflow
```
