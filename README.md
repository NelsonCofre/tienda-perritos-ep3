# Tienda de Perritos — Deploy EKS (EP3)

Aplicación web de 3 capas desplegada en **Amazon EKS** con imágenes en **ECR** y pipeline **GitHub Actions**.

## Arquitectura

```
Usuario → LoadBalancer (AWS) → Frontend (Nginx) → Backend (Node.js) → MySQL
```

| Componente | Tecnología | Exposición |
|---|---|---|
| Frontend | Nginx + HTML/JS | Público (LoadBalancer) |
| Backend | Node.js / Express | Interno (ClusterIP) |
| DB | MySQL 8 | Interno (headless service) |

- Namespace Kubernetes: `tienda`
- Cluster EKS: `devopseks` (región `us-east-1`)

## Estructura del repositorio

```
├── frontend/          # UI estática + Nginx
├── backend/           # API REST Node.js
├── db/                # MySQL + init.sql
├── k8s/               # Manifiestos Kubernetes
├── deploy.sh          # Deploy manual completo
└── .github/workflows/ # CI/CD GitHub Actions
```

## Prerrequisitos

- Cuenta **AWS Academy Learner Lab** activa
- Cluster **EKS** con node group
- Repositorios **ECR**: `tienda-frontend`, `tienda-backend`, `tienda-db`
- Roles IAM: `LabEKSClusterRole`, `LabEKSNodeRole`
- **AWS CLI**, **kubectl**, **Docker** (CloudShell recomendado)

## Deploy manual (primera vez)

Desde **AWS CloudShell**, en la raíz del proyecto:

```bash
chmod +x deploy.sh
./deploy.sh
```

El script ejecuta: kubeconfig → Metrics Server → build/push ECR → deploy DB → backend → frontend → HPA → URL del LoadBalancer.

## Deploy manual por pasos

```bash
export REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_URL="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

aws eks update-kubeconfig --region us-east-1 --name devopseks

find ./k8s -type f -name "*.yaml" -exec sed -i "s|{{ECR_URL}}|${ECR_URL}|g" {} \;

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_URL}
# build + push frontend, backend, db (ver deploy.sh)

kubectl apply -f ./k8s/namespace.yaml
kubectl apply -f ./k8s/mysql-secret.yaml
kubectl apply -f ./k8s/mysql-deployment.yaml
kubectl apply -f ./k8s/mysql-service.yaml
kubectl rollout status deployment/tienda-db -n tienda --timeout=300s

kubectl apply -f ./k8s/backend-deployment.yaml
kubectl apply -f ./k8s/backend-service.yaml
kubectl apply -f ./k8s/frontend-deployment.yaml
kubectl apply -f ./k8s/frontend-service.yaml
kubectl apply -f ./k8s/backend-hpa.yaml
kubectl apply -f ./k8s/frontend-hpa.yaml
```

## Validación

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
kubectl top pods -n tienda
```

Abrir en navegador la URL del Service `tienda-frontend` (EXTERNAL-IP o hostname ELB).

- `/` — tienda
- `/api/productos` — JSON productos
- `/api/health` — health check backend

## CI/CD — GitHub Actions

Flujo: **push rama `deploy`** → build Docker → push ECR → rolling update en EKS.

### Secrets requeridos en GitHub

| Secret | Valor |
|---|---|
| `AWS_ACCESS_KEY_ID` | Learner Lab |
| `AWS_SECRET_ACCESS_KEY` | Learner Lab |
| `AWS_SESSION_TOKEN` | Learner Lab (obligatorio) |
| `AWS_REGION` | `us-east-1` |
| `AWS_ACCOUNT_ID` | Account ID AWS |
| `EKS_CLUSTER_NAME` | `devopseks` |
| `EKS_NAMESPACE` | `tienda` |

> Renovar los 3 secrets de AWS cuando expire el Learner Lab.

### Disparar pipeline

```bash
git checkout deploy
git add .
git commit -m "ci: deploy backend/frontend"
git push origin deploy
```

## Autoscaling (HPA)

- Backend: CPU 70%, min 2, max 10 réplicas
- Frontend: CPU 60%, min 2, max 6 réplicas

Requiere **Metrics Server** instalado en el cluster.

## Secrets Kubernetes

- `mysql-secret`: password root MySQL (usado por backend vía `secretKeyRef`)

## Troubleshooting

| Problema | Solución |
|---|---|
| Pod Pending | Revisar node group y subnets |
| ImagePullBackOff | Verificar imagen en ECR y placeholder `{{ECR_URL}}` |
| Backend CrashLoop | Esperar que MySQL esté Ready |
| Sin EXTERNAL-IP | Etiquetar subnets públicas con tags ELB de Kubernetes |
