# Tienda de Perritos - DevOps AWS EKS

Aplicación web de una tienda de productos para perritos, utilizada como caso práctico para implementar un flujo DevOps completo: **frontend + backend + base de datos MySQL**, contenerización con Docker, publicación de imágenes en **Amazon ECR**, despliegue en **Amazon EKS** y automatización mediante **GitHub Actions**.

---

## 1. Descripción del proyecto

El sistema corresponde a una tienda simple de productos para mascotas. Permite visualizar, crear, editar y eliminar productos desde una interfaz web. La aplicación está organizada en tres componentes principales:

- **Frontend:** interfaz web servida con Nginx.
- **Backend:** API REST desarrollada con Node.js y Express.
- **Base de datos:** MySQL con una tabla de productos inicializada mediante `init.sql`.

El objetivo del proyecto es demostrar cómo una aplicación puede pasar desde el código fuente hasta un entorno cloud, utilizando herramientas de automatización, contenedores y orquestación.

---

## 2. Tecnologías utilizadas

- HTML, CSS y JavaScript
- Node.js
- Express
- MySQL 8
- Docker
- Nginx
- Kubernetes
- Amazon ECR
- Amazon EKS
- GitHub Actions
- Terraform
- AWS CLI
- kubectl

---

## 3. Estructura del repositorio

```text
Tienda-Perritos-E.Transversal/
├── .github/
│   └── workflows/
│       └── deploy-eks.yml
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── db/
│   ├── Dockerfile
│   └── init.sql
├── frontend/
│   ├── Dockerfile
│   ├── app.js
│   ├── default.conf
│   └── index.html
├── k8s/
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
└── tf-aws-eks-main/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── providers.tf
    ├── ecr.tf
    ├── backend.hcl.example
    └── terraform.tfvars.example
```

---

## 4. Arquitectura general

La aplicación utiliza una arquitectura de tres capas:

```text
Usuario
   ↓
LoadBalancer AWS
   ↓
Frontend Nginx
   ↓
Backend API Node.js / Express
   ↓
Base de datos MySQL
```

El frontend es el único componente expuesto hacia el usuario mediante un servicio de tipo `LoadBalancer`. El backend y la base de datos se comunican internamente dentro del clúster mediante servicios de Kubernetes.

El flujo DevOps utilizado es el siguiente:

```text
GitHub
   ↓
GitHub Actions
   ↓
Build de imágenes Docker
   ↓
Amazon ECR
   ↓
Amazon EKS
   ↓
Aplicación funcionando en la nube
```

---

## 5. Componentes de la aplicación

### Frontend

La carpeta `frontend/` contiene la interfaz de la tienda. Se sirve mediante Nginx y utiliza el archivo `default.conf` para redirigir las solicitudes `/api/` hacia el backend dentro del clúster.

Archivo principal:

```text
frontend/index.html
frontend/app.js
frontend/default.conf
frontend/Dockerfile
```

### Backend

La carpeta `backend/` contiene una API REST desarrollada con Node.js y Express. Esta API se conecta a MySQL mediante variables de entorno.

Endpoints principales:

```text
GET    /api/health
GET    /api/productos
GET    /api/productos/:id
POST   /api/productos
PUT    /api/productos/:id
DELETE /api/productos/:id
```

### Base de datos

La carpeta `db/` contiene la configuración de MySQL y el script `init.sql`, encargado de crear la base de datos `tienda_perritos`, la tabla `productos` y algunos registros iniciales.

---

## 6. Ejecución local con Docker

Para probar los contenedores de forma local, se puede crear una red Docker y levantar los servicios manualmente:

```bash
docker network create tienda-net
```

### Base de datos

```bash
docker build -t tienda-db ./db

docker run -d \
  --name tienda-db \
  --network tienda-net \
  -p 3306:3306 \
  tienda-db
```

### Backend

```bash
docker build -t tienda-backend ./backend

docker run -d \
  --name tienda-backend \
  --network tienda-net \
  -p 3001:3001 \
  -e DB_HOST=tienda-db \
  -e DB_USER=root \
  -e DB_PASSWORD=admin123 \
  -e DB_NAME=tienda_perritos \
  tienda-backend
```

### Frontend

```bash
docker build -t tienda-frontend ./frontend

docker run -d \
  --name tienda-frontend \
  --network tienda-net \
  -p 8080:80 \
  tienda-frontend
```

Luego abrir en el navegador:

```text
http://localhost:8080
```

Para probar el backend:

```text
http://localhost:3001/api/health
http://localhost:3001/api/productos
```

---

## 7. Infraestructura AWS con Terraform

La carpeta `tf-aws-eks-main/` contiene la infraestructura base para AWS. Esta configuración crea recursos necesarios para ejecutar la aplicación en EKS, incluyendo VPC, subredes, clúster EKS, repositorios ECR y configuración relacionada.

Comandos principales:

```bash
cd tf-aws-eks-main
terraform init -backend-config=backend.hcl
terraform validate
terraform plan --var-file=terraform.tfvars
terraform apply --var-file=terraform.tfvars
```

Al finalizar, Terraform entrega información útil como el nombre del clúster, endpoint y repositorios ECR.

Para configurar `kubectl` contra el clúster:

```bash
aws eks update-kubeconfig --region us-east-1 --name <NOMBRE_DEL_CLUSTER>
```

---

## 8. Despliegue en Kubernetes / EKS

Los manifiestos de Kubernetes están dentro de la carpeta `k8s/`.

Aplicación manual de recursos:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/mysql-secret.yaml -n tienda
kubectl apply -f k8s/mysql-deployment.yaml -n tienda
kubectl apply -f k8s/mysql-service.yaml -n tienda
kubectl apply -f k8s/backend-deployment.yaml -n tienda
kubectl apply -f k8s/backend-service.yaml -n tienda
kubectl apply -f k8s/frontend-deployment.yaml -n tienda
kubectl apply -f k8s/frontend-service.yaml -n tienda
kubectl apply -f k8s/backend-hpa.yaml -n tienda
kubectl apply -f k8s/frontend-hpa.yaml -n tienda
```

Verificar recursos:

```bash
kubectl get pods -n tienda
kubectl get deployments -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
```

El servicio `tienda-frontend` es de tipo `LoadBalancer`. Para acceder a la aplicación, copiar el `EXTERNAL-IP` o DNS generado por AWS:

```bash
kubectl get svc tienda-frontend -n tienda
```

---

## 9. Pipeline CI/CD con GitHub Actions

El flujo de CI/CD se encuentra en:

```text
.github/workflows/deploy-eks.yml
```

Este workflow se ejecuta automáticamente al hacer `push` a la rama `main` o manualmente desde la pestaña **Actions** de GitHub.

Etapas principales del pipeline:

1. Descarga el código del repositorio.
2. Configura credenciales AWS usando GitHub Secrets.
3. Inicia sesión en Amazon ECR.
4. Define un tag de imagen basado en el commit.
5. Construye y publica las imágenes Docker del frontend, backend y base de datos.
6. Configura `kubectl` contra el clúster EKS.
7. Aplica los manifiestos de Kubernetes.
8. Actualiza las imágenes del frontend y backend en EKS.
9. Aplica HPA.
10. Muestra el estado final de pods, servicios y escaladores.

---

## 10. Secrets requeridos en GitHub

Los secretos deben configurarse en:

```text
GitHub → Settings → Secrets and variables → Actions → New repository secret
```

Secrets necesarios:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_REGION
EKS_CLUSTER_NAME
EKS_NAMESPACE
```

Ejemplo de valores no sensibles:

```text
AWS_REGION=us-east-1
EKS_NAMESPACE=tienda
```

> Importante: las credenciales reales de AWS no deben escribirse en el código fuente, README, commits ni archivos `.yml`. Siempre deben manejarse mediante GitHub Secrets.

---

## 11. Registro de imágenes en Amazon ECR

El pipeline publica las imágenes en los siguientes repositorios ECR:

```text
dev-eks-lab-frontend-repo
dev-eks-lab-backend-repo
dev-eks-lab-bd-repo
```

El tag utilizado por el workflow se genera automáticamente desde el commit de GitHub:

```text
IMAGE_TAG=${GITHUB_SHA::7}
```

Esto permite mantener trazabilidad entre una ejecución del pipeline, un commit específico y las imágenes desplegadas.

---

## 12. Seguridad básica aplicada

El proyecto considera prácticas básicas de seguridad:

- Uso de GitHub Secrets para credenciales AWS.
- Uso de Kubernetes Secret para la contraseña de MySQL.
- Backend y base de datos no expuestos directamente hacia internet.
- Frontend expuesto mediante LoadBalancer.
- Comunicación interna mediante servicios de Kubernetes.
- Recursos con requests y limits definidos para frontend y backend.
- Health checks con readinessProbe y livenessProbe.

Como mejora productiva, las credenciales podrían administrarse con AWS Secrets Manager y políticas IAM más específicas.

---

## 13. Observabilidad y verificación

Comandos útiles para verificar el estado del sistema:

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
kubectl logs deployment/tienda-backend -n tienda
kubectl logs deployment/tienda-frontend -n tienda
```

Validación de backend:

```text
/api/health
/api/productos
```

Validación de frontend:

```text
http://<EXTERNAL-IP-O-DNS-DEL-LOADBALANCER>
```

También se puede revisar la ejecución del pipeline desde GitHub Actions, donde quedan registrados los logs de build, push y deploy.

---

## 14. Integrantes

- Integrante 1: Sebastián Orellana Balbi
- Integrante 2: Pablo Olate Carreño

---

## 15. Estado del proyecto

El proyecto cuenta con:

- Aplicación frontend funcional.
- Backend API funcional.
- Base de datos MySQL inicializada.
- Dockerfile por componente.
- Manifiestos Kubernetes.
- Despliegue en Amazon EKS.
- Publicación de imágenes en Amazon ECR.
- Pipeline GitHub Actions ejecutado correctamente.
- Evidencia visual de la aplicación desplegada.

