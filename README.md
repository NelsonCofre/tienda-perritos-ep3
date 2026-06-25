# Tienda de Perritos — EKS

Aplicación web CRUD para gestionar productos de una tienda de alimentos para perros. El sistema corre en **Amazon EKS** con imágenes en **Amazon ECR** y despliegues automatizados mediante **GitHub Actions**.

## Arquitectura

```
Usuario (navegador)
        │
        ▼
┌───────────────────┐
│  Frontend (Nginx) │  Service: LoadBalancer (público)
└─────────┬─────────┘
          │ /api/*
          ▼
┌───────────────────┐
│  Backend (Node.js)│  Service: ClusterIP (interno)
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  MySQL 8          │  Service: ClusterIP (interno)
└───────────────────┘
```

| Componente | Tecnología | Puerto |
|---|---|---|
| Frontend | Nginx + HTML/JS | 80 |
| Backend | Node.js + Express | 3001 |
| Base de datos | MySQL 8 | 3306 |

**Namespace Kubernetes:** `tienda`  
**Cluster EKS:** `devopseks` (región `us-east-1`)

## Estructura del repositorio

```
├── backend/              # API REST (productos CRUD)
├── frontend/             # Interfaz web + proxy Nginx al backend
├── db/                   # MySQL + script de inicialización
├── k8s/                  # Manifiestos Kubernetes (deployments, services, HPA, secrets)
├── .github/workflows/    # Pipelines CI/CD (backend y frontend)
└── deploy.sh             # Script de despliegue inicial (build, push ECR, apply k8s)
```

## Requisitos previos

- Cuenta **AWS Academy Learner Lab** activa
- Cluster EKS creado con roles `LabEKSClusterRole` y `LabEKSNodeRole`
- Subnets públicas etiquetadas para Load Balancer
- Herramientas: AWS CLI, kubectl, Docker, Git

## Despliegue inicial

Desde **AWS CloudShell** (con el repositorio clonado):

```bash
chmod +x deploy.sh
./deploy.sh
```

El script realiza:

1. Conexión al cluster EKS
2. Configuración de Metrics Server (si no existe)
3. Build y push de imágenes a ECR (`tienda-frontend`, `tienda-backend`, `tienda-db`)
4. Aplicación de manifiestos en el namespace `tienda`
5. Configuración de HPA para frontend y backend

Obtener la URL pública:

```bash
kubectl get svc tienda-frontend -n tienda
```

Abrir el valor de `EXTERNAL-IP` en el navegador.

## CI/CD con GitHub Actions

Los pipelines se ejecutan al hacer **push a la rama `deploy`** cuando hay cambios en `backend/` o `frontend/`.

| Workflow | Disparador | Acción |
|---|---|---|
| `CI/CD Backend EKS` | Cambios en `backend/**` | Build → ECR → rolling update |
| `CI/CD Frontend EKS` | Cambios en `frontend/**` | Build → ECR → rolling update |

### Secrets requeridos en GitHub

Configurar en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial temporal del Learner Lab |
| `AWS_SECRET_ACCESS_KEY` | Credencial temporal del Learner Lab |
| `AWS_SESSION_TOKEN` | Token STS (obligatorio en Learner Lab) |
| `AWS_REGION` | `us-east-1` |
| `AWS_ACCOUNT_ID` | ID de la cuenta AWS |
| `EKS_CLUSTER_NAME` | `devopseks` |
| `EKS_NAMESPACE` | `tienda` |

> Las credenciales del Learner Lab expiran. Renovar los tres secrets AWS cuando caduquen.

### Flujo de trabajo

```bash
git checkout deploy
# editar backend/ o frontend/
git add .
git commit -m "feat: descripción del cambio"
git push origin deploy
```

Las imágenes en ECR se etiquetan como `eks-<número_de_ejecución>`.

## Autoscaling (HPA)

Horizontal Pod Autoscaler configurado para frontend y backend usando métricas de CPU (Metrics Server).

```bash
kubectl get hpa -n tienda
kubectl top pods -n tienda
```

## Verificación y logs

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl rollout status deployment/tienda-backend -n tienda
kubectl rollout status deployment/tienda-frontend -n tienda
kubectl logs deployment/tienda-backend -n tienda --tail=50
```

Health check del backend:

```bash
curl http://<URL-LOADBALANCER>/api/health
```

## Endpoints de la API

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/productos` | Listar productos |
| GET | `/api/productos/:id` | Obtener un producto |
| POST | `/api/productos` | Crear producto |
| PUT | `/api/productos/:id` | Actualizar producto |
| DELETE | `/api/productos/:id` | Eliminar producto |
| GET | `/api/health` | Estado del backend |

## Credenciales de base de datos

Las credenciales MySQL se almacenan en el Secret de Kubernetes `mysql-secret` (`k8s/mysql-secret.yaml`). No deben incluirse en el código fuente.
