# EP3 — Introducción a Herramientas DevOps
## Innovatech Chile — Tienda de Perritos

Guía completa alineada con la **rúbrica EP3**, el proyecto real de este repositorio y **AWS Academy Learner Lab**.

---

## Objetivo

Implementar orquestación y automatización en AWS con:

- **Amazon EKS** — cluster Kubernetes
- **Amazon ECR** — registro de imágenes Docker
- **Kubernetes** — deploy de frontend, backend y MySQL
- **GitHub Actions** — pipeline CI/CD (build → push → deploy)
- **HPA** — autoscaling horizontal de pods
- **CloudWatch / kubectl logs** — observabilidad

---

## Arquitectura del proyecto

```
Usuario
   │
   ▼
LoadBalancer (AWS ELB)          ← Service tienda-frontend (público)
   │
   ▼
Frontend (Nginx :80)            ← 2 réplicas + HPA 60% CPU
   │  proxy /api/ →
   ▼
Backend (Node.js :3001)         ← 2 réplicas + HPA 70% CPU — ClusterIP (interno)
   │
   ▼
MySQL (tienda-db :3306)         ← 1 réplica — headless service (interno)
```

### Comunicación interna

| Origen | Destino | Cómo |
|---|---|---|
| Navegador | Frontend | URL pública del LoadBalancer |
| Frontend (Nginx) | Backend | `http://tienda-backend:3001` (DNS interno K8s) |
| Backend | MySQL | `tienda-db:3306` (variable `DB_HOST`) |

El backend **nunca** se expone públicamente (`ClusterIP`).

---

## Valores fijos de este proyecto

| Recurso | Nombre |
|---|---|
| Región AWS | `us-east-1` |
| Cluster EKS | `devopseks` |
| Node group | `devopseks-nodes` (sugerido) |
| Namespace K8s | `tienda` |
| ECR repos | `tienda-frontend`, `tienda-backend`, `tienda-db` |
| Tag imágenes (deploy manual) | `eks-v1` |
| Tag imágenes (CI/CD) | `eks-<run_number>` |
| Rama pipeline | `deploy` |

---

## Qué trae el Learner Lab vs qué creas tú

| Recurso | ¿Lo creas? | Notas |
|---|---|---|
| VPC + subnets | **No** | Ya viene una VPC del lab |
| Internet Gateway (IGW) | **No** | Ya asociado a la VPC |
| **NAT Gateway** | **No** | Learner Lab no trae NAT y **no debes crearlo** para este EP3 |
| Route tables | **No** | Verificar ruta `0.0.0.0/0 → IGW` |
| Roles IAM | **No** | Verificar `LabEKSClusterRole` y `LabEKSNodeRole` |
| Repos ECR (×3) | **Sí** | |
| Cluster EKS | **Sí** | |
| Node Group | **Sí** | Usar **subnets públicas** (sin NAT) |
| Tags en subnets | **Sí** | Para LoadBalancer |
| Deploy Kubernetes | **Sí** | `./deploy.sh` |
| Repo GitHub + CI/CD | **Sí** | Obligatorio para EP3 |

### ¿NAT Gateway? — No lo necesitas

En AWS Academy Learner Lab es **normal** ver esto en VPC → NAT gateways:

```
No NAT gateways found
```

**Eso está bien.** No hagas clic en **Create NAT gateway**.

| Gateway | ¿Lo necesitas? | Motivo |
|---|---|---|
| **Internet Gateway (IGW)** | **Sí** (ya existe) | Sale a internet vía subnets públicas |
| **NAT Gateway** | **No** | Solo haría falta si los nodos EKS estuvieran en subnets **privadas** sin IP pública |

Para este proyecto en el lab:
- Los **nodos EKS** van en **subnets públicas** (con ruta al IGW).
- Los **pods** descargan imágenes de ECR por esa misma ruta.
- El **LoadBalancer** del frontend se crea en subnets públicas etiquetadas.

> En producción real a veces se usa NAT Gateway para nodos en subnets privadas. En Learner Lab **no es necesario** y además consume créditos del lab innecesariamente.

---

## Estructura del repositorio

```
deploy-tienda-perritos/
├── frontend/                    # Nginx + HTML/JS
│   ├── Dockerfile
│   ├── default.conf             # proxy → tienda-backend:3001
│   ├── index.html
│   └── app.js
├── backend/                     # API Node.js/Express
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
├── db/                          # MySQL 8 + datos iniciales
│   ├── Dockerfile
│   └── init.sql
├── k8s/                         # Manifiestos Kubernetes
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
├── deploy.sh                    # Deploy manual completo
├── .github/workflows/
│   ├── deploy-backend.yml
│   └── deploy-frontend.yml
├── README.md
└── README_EP3_DEVOPS.md         # Esta guía
```

---

# Flujo completo paso a paso (desde cero)

---

## Fase 0 — Herramientas

### Opción A — AWS CloudShell (recomendado)

CloudShell incluye **AWS CLI**, **kubectl** y **Docker**. No necesitas Docker Desktop.

1. Activa Learner Lab → clic en **AWS** → abre **CloudShell** (icono `>_`).

### Opción B — PC local

Instala: AWS CLI, kubectl, Git, VS Code. Docker Desktop solo si harás `docker build` en tu PC (CloudShell es más simple).

Verificar:

```bash
aws --version
kubectl version --client
docker --version   # en CloudShell o con Docker Desktop
```

---

## Fase 1 — Activar AWS Academy Learner Lab

1. Entra a **AWS Academy Learner Lab**.
2. **Start Lab** → luz verde.
3. Clic en **AWS** (consola).
4. Región: **US East (N. Virginia) / us-east-1**.

En CloudShell:

```bash
aws sts get-caller-identity
```

Anota tu **Account ID** (12 dígitos).

> **Learner Lab:** las credenciales son temporales (STS). Incluyen `AWS_SESSION_TOKEN`. Cuando expire el lab, renueva los secrets en GitHub.

**Screenshot:** lab activo + salida de `get-caller-identity`.

---

## Fase 2 — Verificar red e IAM (NO crear VPC, IGW ni NAT)

### 2.1 VPC del lab

1. Consola → **VPC** → **Your VPCs** → verás **1 VPC** (la del lab).
2. Anota el **VPC ID** (ej. `vpc-0cbe652eb0ed6f544`).

### 2.2 Internet Gateway (IGW) — sí debe existir

1. VPC → **Internet gateways** → debe haber un IGW **Attached** a tu VPC.
2. **No crees otro** si ya está asociado.

### 2.3 NAT Gateway — NO crear (lista vacía es correcta)

1. VPC → **NAT gateways**.
2. Verás **"No NAT gateways found"** → **esto es normal y esperado**.
3. **No pulses "Create NAT gateway"** — no lo necesitas para este EP3.

### 2.4 Route table — identificar subnets públicas

1. VPC → **Route tables** → abre la route table de tu VPC.
2. Pestaña **Routes** → debe existir:

```
Destino: 0.0.0.0/0
Target:  igw-xxxxxxxx   ← Internet Gateway (subnet PÚBLICA)
```

3. Pestaña **Subnet associations** → anota qué subnets usan esa route table.
4. Esas subnets son las **públicas** que usarás para EKS y para los tags del LoadBalancer.

> Si la ruta `0.0.0.0/0` apunta a `igw-...` (no a `nat-...`), la subnet es pública. En Learner Lab **todas** suelen ser públicas.

### 2.5 Roles IAM

1. **IAM** → **Roles** → confirma:
   - `LabEKSClusterRole`
   - `LabEKSNodeRole`

**Screenshots:** VPC, IGW attached, NAT gateways vacío (OK), route table con IGW, roles IAM.

---

## Fase 3 — Etiquetar subnets públicas

Necesario para que el frontend obtenga URL pública (LoadBalancer).

En **2 subnets públicas** de distintas AZ (ej. `us-east-1a` y `us-east-1b`):

1. VPC → **Subnets** → selecciona subnet → **Tags** → **Manage tags**.

| Key | Value |
|---|---|
| `kubernetes.io/cluster/devopseks` | `shared` |
| `kubernetes.io/role/elb` | `1` |

**Screenshot:** tags en ambas subnets.

---

## Fase 4 — Crear repositorios ECR

En CloudShell:

```bash
aws ecr create-repository --repository-name tienda-frontend --region us-east-1
aws ecr create-repository --repository-name tienda-backend --region us-east-1
aws ecr create-repository --repository-name tienda-db --region us-east-1
```

Verificar:

```bash
aws ecr describe-repositories --region us-east-1 \
  --query 'repositories[].repositoryName' --output table
```

**Screenshot:** 3 repos en ECR.

---

## Fase 5 — Crear cluster EKS

1. Consola → **EKS** → **Clusters** → **Create cluster**.

### Configure cluster

| Campo | Valor |
|---|---|
| Name | `devopseks` |
| Cluster IAM role | `LabEKSClusterRole` |
| Kubernetes version | Default |

### Networking

| Campo | Valor |
|---|---|
| VPC | La VPC del Learner Lab |
| Subnets | Al menos 2 en distintas AZ |
| Cluster endpoint access | Public and private |

### Observability

Activar logs: `api`, `audit`, `authenticator`, `controllerManager`, `scheduler`.

### Add-ons

- Amazon VPC CNI
- **Metrics Server** (requerido para HPA)
- Amazon CloudWatch Observability (opcional)

**Create** → esperar estado **Active** (~10–15 min).

**Screenshot:** cluster `devopseks` Active.

---

## Fase 6 — Crear Node Group

1. EKS → `devopseks` → **Compute** → **Add node group**.

| Campo | Valor |
|---|---|
| Name | `devopseks-nodes` |
| Node IAM role | `LabEKSNodeRole` |
| AMI type | Amazon Linux 2 (x86) |
| Capacity type | **Spot** |
| Instance types | `t3.large` |
| Desired / Min / Max | `1` / `1` / `3` |
| Subnets | **Subnets públicas** (las que tienen ruta `0.0.0.0/0 → igw-...`) |

> **Sin NAT Gateway:** los workers deben ir en subnets **públicas** para poder descargar imágenes de ECR e iniciar pods. Selecciona las mismas subnets donde pusiste los tags ELB (Fase 3).

Esperar estado **Active** (~5 min).

**Screenshot:** node group Active.

---

## Fase 7 — Conectar kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name devopseks
kubectl get nodes
```

Debe mostrar nodo(s) **Ready**.

**Screenshot:** `kubectl get nodes`.

---

## Fase 8 — Subir proyecto a CloudShell

**Opción A — ZIP:** comprime el proyecto (sin `node_modules`) → CloudShell → Actions → Upload file.

```bash
unzip deploy-tienda-perritos.zip
cd deploy-tienda-perritos
```

**Opción B — GitHub:**

```bash
git clone https://github.com/TU-USUARIO/tienda-perritos-ep3.git
cd tienda-perritos-ep3
```

---

## Fase 9 — Deploy automático (recomendado)

```bash
chmod +x deploy.sh
./deploy.sh
```

El script ejecuta en orden:

1. Conecta kubectl al cluster
2. Instala Metrics Server
3. Reemplaza `{{ECR_URL}}` en manifests
4. Login ECR + build/push de 3 imágenes (`eks-v1`)
5. Deploy: namespace → **MySQL** → **backend** → **frontend** → **HPA**
6. Muestra URL del LoadBalancer

Tiempo estimado: **15–20 min**.

---

## Fase 10 — Deploy manual (alternativa)

```bash
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URL="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

aws eks update-kubeconfig --region us-east-1 --name devopseks

# Reemplazar placeholder ECR en manifests
find ./k8s -type f -name "*.yaml" \
  -exec sed -i "s|{{ECR_URL}}|${ECR_URL}|g" {} \;

# Login ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ECR_URL}

# Build + push (desde raíz del proyecto)
docker build -t tienda-frontend ./frontend
docker tag tienda-frontend:latest ${ECR_URL}/tienda-frontend:eks-v1
docker push ${ECR_URL}/tienda-frontend:eks-v1

docker build -t tienda-backend ./backend
docker tag tienda-backend:latest ${ECR_URL}/tienda-backend:eks-v1
docker push ${ECR_URL}/tienda-backend:eks-v1

docker build -t tienda-db ./db
docker tag tienda-db:latest ${ECR_URL}/tienda-db:eks-v1
docker push ${ECR_URL}/tienda-db:eks-v1

# Kubernetes — ORDEN OBLIGATORIO
kubectl apply -f ./k8s/namespace.yaml

kubectl apply -f ./k8s/mysql-secret.yaml
kubectl apply -f ./k8s/mysql-deployment.yaml
kubectl apply -f ./k8s/mysql-service.yaml
kubectl rollout status deployment/tienda-db -n tienda --timeout=300s

kubectl apply -f ./k8s/backend-deployment.yaml
kubectl apply -f ./k8s/backend-service.yaml
kubectl rollout status deployment/tienda-backend -n tienda --timeout=300s

kubectl apply -f ./k8s/frontend-deployment.yaml
kubectl apply -f ./k8s/frontend-service.yaml
kubectl rollout status deployment/tienda-frontend -n tienda --timeout=300s

kubectl apply -f ./k8s/backend-hpa.yaml
kubectl apply -f ./k8s/frontend-hpa.yaml
```

> **No uses** `kubectl apply -f k8s/` a ciegas: el orden importa (DB antes que backend) y los manifests llevan `{{ECR_URL}}` hasta reemplazarlo.

---

## Fase 11 — Validar despliegue

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
```

Obtener URL pública:

```bash
kubectl get svc tienda-frontend -n tienda
```

Espera **EXTERNAL-IP** o hostname `xxxx.elb.amazonaws.com` (1–3 min).

### Probar en navegador

| URL | Resultado esperado |
|---|---|
| `http://<URL>/` | Tienda con productos |
| `http://<URL>/api/productos` | JSON con productos |
| `http://<URL>/api/health` | `{"status":"ok",...}` |

Probar **CRUD**: crear, editar y eliminar un producto.

**Screenshots:** pods Running, svc con EXTERNAL-IP, tienda en navegador, API JSON.

---

## Fase 12 — Autoscaling (HPA)

Configuración en este proyecto:

| HPA | Deployment | CPU objetivo | Min | Max |
|---|---|---|---|---|
| `tienda-backend-hpa` | `tienda-backend` | 70% | 2 | 10 |
| `tienda-frontend-hpa` | `tienda-frontend` | 60% | 2 | 6 |

Verificar Metrics Server:

```bash
kubectl top nodes
kubectl top pods -n tienda
kubectl get hpa -n tienda
```

Simular carga (opcional):

```bash
kubectl run load-test --image=busybox -n tienda --restart=Never -- \
  sh -c "while true; do wget -q -O- http://tienda-backend:3001/api/productos; done"

# Esperar 3-5 min, luego:
kubectl get hpa -n tienda
kubectl get pods -n tienda

kubectl delete pod load-test -n tienda
```

**Justificación para la defensa:** backend al 70% (API + MySQL, más sensible); frontend al 60% (Nginx ligero, escala antes).

**Screenshots:** HPA, `kubectl top pods`, réplicas si escalan.

---

## Fase 13 — Logs y CloudWatch (IE6)

### kubectl logs

```bash
kubectl logs deployment/tienda-backend -n tienda --tail=50
kubectl logs deployment/tienda-frontend -n tienda --tail=50
kubectl logs deployment/tienda-db -n tienda --tail=50
```

### CloudWatch

1. Consola → **CloudWatch** → **Log groups**.
2. Buscar `/aws/eks/devopseks/cluster` (logs del control plane).

**Screenshots:** logs backend sin errores, log group CloudWatch.

---

## Fase 14 — GitHub + CI/CD (obligatorio EP3)

### 14.1 Crear repositorio

1. GitHub → **New repository** → `tienda-perritos-ep3`.
2. Subir código desde tu PC:

```powershell
cd "ruta\deploy-tienda-perritos"
git init
git add .
git commit -m "feat: proyecto EP3 - EKS, k8s, CI/CD"
git remote add origin https://github.com/TU-USUARIO/tienda-perritos-ep3.git
git branch -M main
git push -u origin main
git checkout -b deploy
git push -u origin deploy
```

### 14.2 Secrets en GitHub

Repo → **Settings** → **Secrets and variables** → **Actions**:

| Secret | Valor |
|---|---|
| `AWS_ACCESS_KEY_ID` | Learner Lab → AWS Details |
| `AWS_SECRET_ACCESS_KEY` | Learner Lab |
| `AWS_SESSION_TOKEN` | Learner Lab (**obligatorio**) |
| `AWS_REGION` | `us-east-1` |
| `AWS_ACCOUNT_ID` | Tu Account ID |
| `EKS_CLUSTER_NAME` | `devopseks` |
| `EKS_NAMESPACE` | `tienda` |

> Renovar los 3 secrets de AWS cuando expire el Learner Lab.

### 14.3 Workflows incluidos

| Archivo | Dispara con push a `deploy` en |
|---|---|
| `.github/workflows/deploy-backend.yml` | cambios en `backend/` |
| `.github/workflows/deploy-frontend.yml` | cambios en `frontend/` |

Flujo: **commit → build Docker → push ECR → kubectl set image → rolling update**.

### 14.4 Probar pipeline

1. Learner Lab **activo**.
2. Cambio en `backend/server.js` o `frontend/index.html`.
3. Push a rama `deploy`:

```bash
git add .
git commit -m "ci: test pipeline deploy"
git push origin deploy
```

4. GitHub → **Actions** → workflow en verde.

```bash
kubectl rollout status deployment/tienda-backend -n tienda
kubectl get pods -n tienda
```

**Screenshots:** secrets, Actions verde, nueva imagen `eks-N` en ECR.

---

## Fase 15 — Secrets Kubernetes (IE5)

| Secret | Uso |
|---|---|
| `mysql-secret` | Password root MySQL (`admin123` en base64) |
| `backend-deployment` | `DB_PASSWORD` vía `secretKeyRef` — no hardcodeado en el manifest |

Verificar:

```bash
kubectl get secrets -n tienda
```

**Screenshot:** secret listado (sin mostrar valores decodificados).

---

## Detalle de manifiestos Kubernetes

| Archivo | Recurso | Descripción |
|---|---|---|
| `namespace.yaml` | Namespace `tienda` | Aislamiento lógico |
| `mysql-secret.yaml` | Secret | Credenciales MySQL |
| `mysql-deployment.yaml` | Deployment `tienda-db` | MySQL 8 desde ECR |
| `mysql-service.yaml` | Service headless `tienda-db` | DNS interno :3306 |
| `backend-deployment.yaml` | Deployment `tienda-backend` | 2 réplicas, puerto 3001, env DB |
| `backend-service.yaml` | Service ClusterIP `tienda-backend` | Solo interno |
| `backend-hpa.yaml` | HPA backend | CPU 70% |
| `frontend-deployment.yaml` | Deployment `tienda-frontend` | 2 réplicas, Nginx :80 |
| `frontend-service.yaml` | Service LoadBalancer `tienda-frontend` | Público |
| `frontend-hpa.yaml` | HPA frontend | CPU 60% |

---

## Troubleshooting

| Problema | Causa probable | Solución |
|---|---|---|
| "No NAT gateways found" | Normal en Learner Lab | **No crear NAT** — usar subnets públicas + IGW |
| Nodos no descargan imágenes ECR | Node group en subnet privada sin NAT | Recrear node group en **subnets públicas** |
| Pod `Pending` | Sin nodos o subnets mal configuradas | `kubectl describe pod -n tienda <pod>` |
| `ImagePullBackOff` | Imagen no en ECR o `{{ECR_URL}}` sin reemplazar | Verificar ECR y ejecutar `sed` o `./deploy.sh` |
| Backend `CrashLoopBackOff` | MySQL no listo | Esperar `tienda-db` Running antes del backend |
| Sin EXTERNAL-IP | Subnets sin tags ELB | Agregar tags de Fase 3, esperar 5 min |
| `kubectl top` falla | Metrics Server no listo | Esperar 2 min o reinstalar Metrics Server |
| Pipeline falla | Credenciales expiradas | Actualizar secrets AWS en GitHub |

---

## Checklist rúbrica EP3

### IE1 — Cluster AWS (25%)
- [ ] EKS `devopseks` Active
- [ ] Roles IAM `LabEKSClusterRole`, `LabEKSNodeRole`
- [ ] VPC del lab + subnets públicas + IGW (sin NAT Gateway)
- [ ] Node group Spot t3.large operativo
- [ ] `kubectl get nodes` → Ready

### IE2 — Deploy Front + Back (25%)
- [ ] 3 imágenes en ECR
- [ ] Frontend LoadBalancer con URL pública
- [ ] Backend ClusterIP (interno)
- [ ] MySQL desplegado y conectado
- [ ] Variables de entorno correctas

### IE3 — Autoscaling (10%)
- [ ] Metrics Server funcionando
- [ ] HPA backend y frontend aplicados
- [ ] `kubectl top pods` con métricas
- [ ] Justificación de umbrales CPU

### IE4 — Pipeline CI/CD (15%)
- [ ] Repo GitHub con workflows
- [ ] 7 secrets configurados
- [ ] Push a rama `deploy` → Actions verde
- [ ] Rolling update exitoso en EKS

### IE5 — Secrets (5%)
- [ ] `mysql-secret` en Kubernetes
- [ ] GitHub Secrets sin credenciales en código
- [ ] `AWS_SESSION_TOKEN` en pipeline

### IE6 — Logs y métricas (10%)
- [ ] `kubectl logs` revisados
- [ ] CloudWatch log groups
- [ ] Tiempos del pipeline documentados

### IE7 — Validación funcional (10%)
- [ ] Tienda carga productos desde MySQL
- [ ] CRUD operativo
- [ ] Front → Back vía proxy Nginx
- [ ] Recuperación post redeploy demostrada

---

## Screenshots mínimas (15)

1. Learner Lab activo
2. `aws sts get-caller-identity`
3. Roles IAM LabEKS*
4. ECR con 3 repos e imágenes
5. EKS cluster Active
6. Node group Active
7. `kubectl get nodes` Ready
8. `kubectl get pods -n tienda` Running
9. `kubectl get svc -n tienda` con EXTERNAL-IP
10. Tienda en navegador
11. `/api/productos` JSON
12. `kubectl get hpa -n tienda`
13. GitHub Secrets configurados
14. GitHub Actions verde
15. Logs backend sin errores

---

## Orden resumido

```
1.  Start Lab + CloudShell
2.  Verificar VPC, IGW, IAM (no crear red)
3.  Tags en 2 subnets públicas
4.  Crear 3 repos ECR
5.  Crear EKS devopseks + node group
6.  kubectl get nodes
7.  Subir proyecto → ./deploy.sh
8.  Probar URL + CRUD
9.  Verificar HPA + logs
10. Crear repo GitHub + secrets
11. Push rama deploy → Actions verde
12. Entregar en AVA + preparar presentación
```

---

## Entrega AVA

- URL del repositorio GitHub
- README documentado (este archivo + `README.md`)
- Commits descriptivos (`feat:`, `ci:`, `fix:`)
- Presentación individual con arquitectura, pipeline, HPA y problemas resueltos
