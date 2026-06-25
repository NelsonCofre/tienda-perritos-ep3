# Guía completa EP3 — AWS + Kubernetes + GitHub CI/CD + Evidencias

Guía unificada para dejar funcionando **Tienda de Perritos** y cumplir la rúbrica EP3.

| Valor fijo | Detalle |
|---|---|
| Región | `us-east-1` |
| Cluster EKS | `devopseks` |
| Namespace K8s | `tienda` |
| Account ID (ejemplo) | `071583947801` |
| Rama CI/CD | `deploy` |

---

## Mapa del flujo completo

```
FASE A — AWS (Learner Lab)     Pasos 1–8
  VPC/IGW (no crear) → tags subnets → ECR → EKS → node group → kubectl

FASE B — GitHub (tu PC)        Paso 9
  Crear repo → push main + rama deploy → 7 secrets

FASE C — Deploy Kubernetes     Pasos 10–14  (CloudShell)
  git clone → ./deploy.sh → verificar app → HPA → logs

FASE D — CI/CD automático      Paso 15
  push a deploy → GitHub Actions → rolling update en EKS
```

### Kubernetes + CI/CD — cómo encajan

| Acción | Herramienta | Cuándo |
|---|---|---|
| **Primera vez** — namespace, MySQL, backend, frontend, HPA, LoadBalancer | `./deploy.sh` | Paso 10 (obligatorio) |
| **Actualizaciones** — nueva imagen backend/frontend | GitHub Actions | Paso 15 (cada push a `deploy`) |

> Los workflows de CI/CD hacen `kubectl set image` (rolling update). **No crean** el cluster ni los deployments desde cero.

### Dos tipos de secrets (no confundir)

| Tipo | Dónde | Para qué |
|---|---|---|
| **GitHub Secrets** (7) | GitHub → Settings → Actions | Pipeline accede a AWS (Learner Lab) |
| **mysql-secret** | `k8s/mysql-secret.yaml` → cluster | Password MySQL dentro de Kubernetes |

---

## Qué creas tú vs qué trae el Learner Lab

| Recurso | ¿Crear? | Notas |
|---|---|---|
| VPC + subnets | **No** | Una VPC del lab |
| Internet Gateway | **No** | Ya attached |
| **NAT Gateway** | **No** | "No NAT gateways found" es **normal** |
| Roles IAM | **No** | Verificar `LabEKSClusterRole`, `LabEKSNodeRole` |
| Tags en subnets | **Sí** | Para LoadBalancer |
| 3 repos ECR | **Sí** | frontend, backend, db |
| Cluster EKS | **Sí** | Custom configuration |
| Node Group | **Sí** | Subnets **públicas** (sin NAT) |
| Repo GitHub + secrets | **Sí** | Obligatorio EP3 |
| Deploy `./deploy.sh` | **Sí** | Primera vez |
| CI/CD GitHub Actions | **Sí** | Después del deploy inicial |

---

## Antes de empezar

- Tiempo estimado: **2–3 horas** (primera vez).
- **CloudShell** para AWS/deploy (icono `>_` en consola).
- **Tu PC** para GitHub (PowerShell + Git).
- Carpeta `evidencias-ep3/` para screenshots numeradas (E01–E44).

---

# FASE A — Infraestructura AWS

---

# PASO 1 — Activar Learner Lab

1. AWS Academy → **Learner Lab** → **Start Lab** (luz **verde**).
2. Clic en **AWS** → región **us-east-1**.
3. Abre **CloudShell**.

```bash
aws sts get-caller-identity
```

Anota tu **Account ID** (12 dígitos).

| ID | Screenshot |
|---|---|
| **E01** | Lab verde |
| **E02** | Consola región us-east-1 |
| **E03** | Salida `get-caller-identity` |

---

# PASO 2 — Verificar red (NO crear VPC, IGW ni NAT)

### 2.1 VPC

VPC → **Your VPCs** → 1 VPC del lab. Anota VPC ID.

### 2.2 Internet Gateway — debe existir

VPC → **Internet gateways** → IGW **Attached** a tu VPC. **No crear otro.**

### 2.3 NAT Gateway — NO crear

VPC → **NAT gateways** → verás:

```
No NAT gateways found
```

**Es correcto.** No pulses **Create NAT gateway**. En el lab usas **subnets públicas** con IGW.

### 2.4 Route table — identificar subnets públicas

Route tables → tu VPC → **Routes**:

```
0.0.0.0/0  →  igw-...   (público — usa estas subnets)
```

### 2.5 Subnets

Anota **2 subnets** en distintas AZ (ej. `us-east-1a`, `us-east-1b`).

| ID | Screenshot |
|---|---|
| **E04** | Your VPCs |
| **E05** | IGW attached |
| **E06** | NAT gateways vacío (OK) |
| **E07** | Route `0.0.0.0/0 → igw` |
| **E08** | Subnets en 2+ AZ |

---

# PASO 3 — Verificar roles IAM

IAM → **Roles** → confirmar:

- `LabEKSClusterRole`
- `LabEKSNodeRole`

| ID | Screenshot |
|---|---|
| **E09** | Ambos roles en lista |
| **E10** | Permissions de `LabEKSClusterRole` |

---

# PASO 4 — Tags en subnets públicas

En **2 subnets públicas** (distintas AZ):

| Key | Value |
|---|---|
| `kubernetes.io/cluster/devopseks` | `shared` |
| `kubernetes.io/role/elb` | `1` |

| ID | Screenshot |
|---|---|
| **E11** | Tags subnet 1 |
| **E12** | Tags subnet 2 |

---

# PASO 5 — Crear repositorios ECR

CloudShell:

```bash
aws ecr create-repository --repository-name tienda-frontend --region us-east-1
aws ecr create-repository --repository-name tienda-backend --region us-east-1
aws ecr create-repository --repository-name tienda-db --region us-east-1

aws ecr describe-repositories --region us-east-1 \
  --query 'repositories[].repositoryName' --output table
```

| ID | Screenshot |
|---|---|
| **E13** | ECR con 3 repos |

---

# PASO 6 — Crear cluster EKS

EKS → **Clusters** → **Create cluster**.

### Elegir tipo de configuración

| Opción | ¿Usar? |
|---|---|
| Quick configuration (Auto Mode) | **NO** |
| **Custom configuration** | **SÍ** |

### Configure cluster

| Campo | Valor |
|---|---|
| Name | `devopseks` |
| Cluster IAM role | `LabEKSClusterRole` |

### Networking

| Campo | Valor |
|---|---|
| VPC | VPC del Learner Lab |
| Subnets | Al menos 2 en distintas AZ |
| Endpoint access | Public and private |

### Si aparece Auto Mode / Node IAM role

| Campo | Valor |
|---|---|
| Cluster IAM role | `LabEKSClusterRole` |
| Node IAM role | `LabEKSNodeRole` |

> Si puedes **desactivar Auto Mode**, hazlo y crea el Node Group manualmente en el Paso 7 (mejor para la rúbrica).

### Observability

Activar logs: api, audit, authenticator, controllerManager, scheduler.

### Add-ons

- [x] Amazon VPC CNI
- [x] **Metrics Server**
- [x] CloudWatch Observability (opcional)

**Create** → esperar **Active** (~10–15 min).

| ID | Screenshot |
|---|---|
| **E14** | Custom configuration seleccionado |
| **E15** | Nombre `devopseks` + `LabEKSClusterRole` |
| **E16** | Metrics Server en add-ons |
| **E17** | Cluster **Active** |

---

# PASO 7 — Crear Node Group

EKS → `devopseks` → **Compute** → **Add node group**.

| Campo | Valor |
|---|---|
| Name | `devopseks-nodes` |
| Node IAM role | `LabEKSNodeRole` |
| Capacity type | **Spot** |
| Instance types | `t3.large` |
| Desired / Min / Max | 1 / 1 / 3 |
| Subnets | **Subnets públicas** (ruta a IGW, sin NAT) |

| ID | Screenshot |
|---|---|
| **E18** | Spot + t3.large |
| **E19** | Node group **Active** |

---

# PASO 8 — Conectar kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name devopseks
kubectl get nodes
```

Debe mostrar nodo(s) **Ready**.

| ID | Screenshot |
|---|---|
| **E20** | `kubectl get nodes` Ready |

---

# FASE B — GitHub (desde tu PC)

> **Obligatorio para EP3** (IE4 CI/CD + entrega AVA). Hazlo **antes o en paralelo** al deploy, pero **antes de entregar**.

---

# PASO 9 — Crear repositorio y secrets GitHub

### 9.1 Crear repo en github.com

1. **New repository** → nombre: `tienda-perritos-ep3`
2. Private o Public (según docente)
3. **Sin** README inicial → **Create**

### 9.2 Subir código desde PowerShell

```powershell
cd "c:\Users\mipc\Downloads\VPC+ECR+EKS+Deploy\2_CloudShell\deploy-tienda-perritos"

git init
git add .
git commit -m "feat: proyecto EP3 - EKS, Kubernetes, GitHub Actions"

git remote add origin https://github.com/TU-USUARIO/tienda-perritos-ep3.git
git branch -M main
git push -u origin main

git checkout -b deploy
git push -u origin deploy
```

Debe incluir: `frontend/`, `backend/`, `db/`, `k8s/`, `deploy.sh`, `.github/workflows/`.

### 9.3 Obtener credenciales AWS (Learner Lab)

Learner Lab → **AWS Details** → copia:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN` ← **obligatorio**

### 9.4 Crear 7 secrets en GitHub

Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret | Valor |
|---|---|
| `AWS_ACCESS_KEY_ID` | Learner Lab |
| `AWS_SECRET_ACCESS_KEY` | Learner Lab |
| `AWS_SESSION_TOKEN` | Learner Lab |
| `AWS_REGION` | `us-east-1` |
| `AWS_ACCOUNT_ID` | Tu Account ID |
| `EKS_CLUSTER_NAME` | `devopseks` |
| `EKS_NAMESPACE` | `tienda` |

> Cuando expire el lab, **actualiza los 3 secrets de AWS**.

| ID | Screenshot |
|---|---|
| **E40** | 7 secrets configurados (valores ocultos) |

---

# FASE C — Deploy Kubernetes (CloudShell)

---

# PASO 10 — Clonar repo y ejecutar deploy.sh

### 10.1 Limpiar ZIP viejo (opcional)

Si antes subiste ZIP a CloudShell:

```bash
cd ~
rm -rf backend db frontend k8s deploy.sh deploy-tienda-perritos.zip
```

### 10.2 Clonar desde GitHub (recomendado)

```bash
git clone https://github.com/TU-USUARIO/tienda-perritos-ep3.git
cd tienda-perritos-ep3
ls -la
```

Debes ver: `frontend/`, `backend/`, `db/`, `k8s/`, `deploy.sh`, `.github/`.

> **Nota ZIP:** si usas ZIP, los archivos quedan en `~` directamente. **No** ejecutes `cd deploy-tienda-perritos` (esa carpeta no existe). Usa `ls` y `./deploy.sh` desde `~`.

### 10.3 Verificar cluster

```bash
aws eks list-clusters --region us-east-1
# debe aparecer: devopseks
```

### 10.4 Deploy completo

```bash
chmod +x deploy.sh
./deploy.sh
```

**Qué hace el script:**

1. Conecta kubectl
2. Metrics Server — **lo omite si ya existe** (add-on EKS)
3. Reemplaza `{{ECR_URL}}` en manifests
4. Build + push 3 imágenes ECR (tag `eks-v1`)
5. Deploy orden: namespace → **MySQL** → **backend** → **frontend** → **HPA**
6. Imprime URL del LoadBalancer

**Tiempo:** 15–20 min. Monitoreo opcional:

```bash
watch kubectl get pods -n tienda
```

### 10.5 Si deploy.sh falló antes (Metrics Server)

El script actualizado ya lo evita. Si falló a medias, continúa manualmente o borra y redeploy:

```bash
kubectl delete namespace tienda          # borra app, no borra EKS
./deploy.sh                              # reintentar
```

| ID | Screenshot |
|---|---|
| **E21** | docker push exitoso |
| **E22** | ECR tienda-frontend `eks-v1` |
| **E23** | ECR tienda-backend `eks-v1` |
| **E24** | ECR tienda-db `eks-v1` |

---

# PASO 11 — Verificar pods y services

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
kubectl get secrets -n tienda
```

**Pods esperados (todos Running):**

```
tienda-db          1/1
tienda-backend     2/2
tienda-frontend    2/2
```

**Services esperados:**

```
tienda-backend    ClusterIP      (interno)
tienda-db         ClusterIP      (interno)
tienda-frontend   LoadBalancer   (EXTERNAL-IP o hostname ELB)
```

| ID | Screenshot |
|---|---|
| **E25** | Pods Running |
| **E26** | svc frontend con EXTERNAL-IP |
| **E27** | svc backend ClusterIP |
| **E28** | EC2 → Load Balancers |
| **E38** | `kubectl get secrets` → mysql-secret |

---

# PASO 12 — Probar en navegador

```bash
kubectl get svc tienda-frontend -n tienda \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}{"\n"}'
```

| URL | Resultado |
|---|---|
| `http://<URL>/` | Tienda con productos |
| `http://<URL>/api/productos` | JSON productos |
| `http://<URL>/api/health` | `{"status":"ok"}` |
| CRUD en la web | Crear / editar / eliminar producto |

| ID | Screenshot |
|---|---|
| **E29** | Tienda en navegador |
| **E30** | /api/productos JSON |
| **E31** | /api/health |
| **E32** | CRUD funcionando |

---

# PASO 13 — HPA y métricas

```bash
kubectl get hpa -n tienda
kubectl top pods -n tienda
kubectl top nodes
```

Carga opcional:

```bash
kubectl run load-test --image=busybox -n tienda --restart=Never -- \
  sh -c "while true; do wget -q -O- http://tienda-backend:3001/api/productos; done"
# esperar 3-5 min → kubectl get hpa -n tienda
kubectl delete pod load-test -n tienda
```

| ID | Screenshot |
|---|---|
| **E33** | kubectl get hpa |
| **E34** | kubectl top pods |
| **E35** | (Opcional) HPA tras carga |

---

# PASO 14 — Logs y CloudWatch

```bash
kubectl logs deployment/tienda-backend -n tienda --tail=40
kubectl logs deployment/tienda-db -n tienda --tail=40
```

Buscar: `Pool de conexiones MySQL inicializado.`

CloudWatch → Log groups → `/aws/eks/devopseks/cluster`

| ID | Screenshot |
|---|---|
| **E36** | Logs backend OK |
| **E37** | Logs MySQL ready |
| **E39** | CloudWatch log group |

---

# FASE D — Probar CI/CD (GitHub Actions)

---

# PASO 15 — Pipeline automático

### Requisitos previos

- [ ] Paso 10 completado (deployments existen en namespace `tienda`)
- [ ] Paso 9 completado (repo + 7 secrets)
- [ ] **Learner Lab activo** (credenciales vigentes)

> **Importante:** El pipeline corre desde **GitHub**, no desde CloudShell.  
> Cambios hechos solo en CloudShell **no** activan CI/CD. Debes hacer commit/push desde tu **PC**.

### Qué rama dispara el pipeline

| Acción | ¿Dispara CI/CD? |
|---|---|
| Push a `main` | **No** |
| Push a `deploy` | **Sí** (si cambias `backend/` o `frontend/`) |
| Merge `main` → `deploy` + push | **Sí** |

### 15.1 Sincronizar fixes desde PC (main → deploy)

Los archivos ya incluyen:
- `k8s/frontend-service.yaml` → LoadBalancer `internet-facing`
- `backend/server.js` → mensaje `/api/health` para verificar el deploy

En **PowerShell**:

```powershell
cd "C:\Users\mipc\Downloads\VPC+ECR+EKS+Deploy\2_CloudShell\deploy-tienda-perritos"

git status
git add k8s/frontend-service.yaml backend/server.js deploy.sh
git commit -m "fix: LoadBalancer internet-facing + mensaje health para CI/CD"

git push origin main

git checkout deploy
git merge main
git push origin deploy
```

> El push a **`deploy`** dispara **CI/CD Backend EKS** (porque cambió `backend/server.js`).

### 15.2 Verificar pipeline en GitHub

GitHub → **Actions** → **CI/CD Backend EKS** → debe quedar **verde**.

### 15.3 Verificar en AWS

CloudShell:

```bash
kubectl rollout status deployment/tienda-backend -n tienda
kubectl get pods -n tienda
```

Navegador — comprobar el deploy del pipeline:

```
http://<TU-URL-ELB>/api/health
```

Debe mostrar:

```json
{"status":"ok","message":"Backend EP3 - deploy via GitHub Actions"}
```

ECR → `tienda-backend` → imagen nueva tag `eks-1`, `eks-2`, etc.

### 15.4 (Opcional) Pipeline frontend

Edita algo visible en `frontend/index.html`, luego:

```powershell
git checkout deploy
git add frontend/index.html
git commit -m "ci: test pipeline frontend EKS"
git push origin deploy
```

GitHub → **CI/CD Frontend EKS** → verde.

| ID | Screenshot |
|---|---|
| **E41** | GitHub Actions verde |
| **E42** | Step Rolling update OK |
| **E43** | ECR imagen `eks-N` |
| **E44** | `/api/health` con mensaje nuevo + rollout success |

### Flujo CI/CD

```
PC: git push rama deploy
    → GitHub Actions
    → docker build + push ECR (tag eks-N)
    → kubectl set image
    → rolling update en EKS
    → /api/health muestra mensaje nuevo
```

> Push a `main` solo **no** ejecuta CI/CD. Debe llegar a **`deploy`** (directo o vía merge).

---

# Limpiar / revertir (si necesitas redeploy)

| Qué borrar | Comando | ¿Borrar EKS? |
|---|---|---|
| Solo la app K8s | `kubectl delete namespace tienda` | No |
| Archivos CloudShell | `rm -rf backend db frontend k8s deploy.sh` | No |
| Cluster EKS | Consola EKS → Delete | Solo si quieres empezar infra de cero |

**No borres** EKS, ECR ni VPC para pasar a GitHub — reutilízalos.

---

# Orden de ejecución (checklist)

```
FASE A — AWS
☐ 1  Start Lab + CloudShell + Account ID
☐ 2  Verificar VPC, IGW, NAT vacío (OK), subnets públicas
☐ 3  Verificar roles IAM LabEKS*
☐ 4  Tags en 2 subnets
☐ 5  Crear 3 repos ECR
☐ 6  EKS devopseks (Custom configuration)
☐ 7  Node group Spot t3.large en subnets públicas
☐ 8  kubectl get nodes → Ready

FASE B — GitHub (PC)
☐ 9  Crear repo + push main + rama deploy + 7 secrets

FASE C — Kubernetes (CloudShell)
☐ 10 git clone → ./deploy.sh
☐ 11 kubectl get pods/svc/secrets
☐ 12 Probar URL + CRUD
☐ 13 HPA + kubectl top
☐ 14 Logs + CloudWatch

FASE D — CI/CD
☐ 15 Push a deploy → Actions verde → verificar ECR/EKS
```

---

# Troubleshooting

| Problema | Solución |
|---|---|
| "No NAT gateways found" | **Normal** — no crear NAT |
| Quick config / Auto Mode | Usar **Custom configuration** |
| Node IAM role requerido | Seleccionar `LabEKSNodeRole` |
| Metrics Server duplicate error | Script lo omite si ya existe; o continúa manualmente |
| `cd deploy-tienda-perritos` falla | ZIP extrae en `~` — no hay subcarpeta |
| Pod Pending | `kubectl describe pod -n tienda <pod>` |
| ImagePullBackOff | Verificar ECR + `./deploy.sh` |
| Backend CrashLoop | Esperar MySQL Ready primero |
| Sin EXTERNAL-IP | Tags subnets Paso 4; esperar 5 min |
| Pipeline falla AWS auth | Renovar KEY + SECRET + **TOKEN** en GitHub |
| `deployment not found` en CI/CD | Falta `./deploy.sh` inicial (Paso 10) |
| Push a main no corre Actions | CI/CD solo en rama **`deploy`** |

---

# Screenshots mínimas (15)

- [ ] E01 Lab activo
- [ ] E03 get-caller-identity
- [ ] E06 NAT vacío (OK)
- [ ] E09 Roles IAM
- [ ] E13 ECR 3 repos
- [ ] E17 EKS Active
- [ ] E20 Nodos Ready
- [ ] E25 Pods Running
- [ ] E26 LoadBalancer URL
- [ ] E29 Tienda navegador
- [ ] E30 /api/productos
- [ ] E33 HPA
- [ ] E38 mysql-secret
- [ ] E40 GitHub secrets
- [ ] E41 Actions verde

---

# Mapeo screenshots → rúbrica

| Rúbrica | Screenshots |
|---|---|
| **IE1** Cluster | E04–E12, E14–E20 |
| **IE2** Deploy | E13, E21–E28 |
| **IE3** HPA | E33–E35 |
| **IE4** CI/CD | E40–E44 |
| **IE5** Secrets | E38, E40 |
| **IE6** Logs | E36, E37, E39, E41 |
| **IE7** Validación | E29–E32, E44 |

---

# Documentos relacionados

| Archivo | Contenido |
|---|---|
| `README_EP3_DEVOPS.md` | Guía técnica completa + arquitectura |
| `README.md` | Referencia rápida del proyecto |
| `GUIA_AWS_EVIDENCIAS.md` | Esta guía (flujo unificado) |

**Entrega AVA:** URL del repo GitHub + evidencias + presentación individual.
