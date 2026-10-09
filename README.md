# Kubernetes - Laboratorio de Aprendizaje

Repositorio de estudio y práctica de **Kubernetes (K8s)**, el sistema de código abierto que automatiza el despliegue, el escalado y la gestión de aplicaciones en contenedores.

El objetivo de este repo es aprender Kubernetes **desde los conceptos básicos hasta la exposición de servicios**, usando **Nginx** como aplicación de ejemplo y **Minikube** como cluster local. Los ejemplos evolucionan paso a paso: primero un Pod suelto, luego un Deployment con réplicas, después Services, Ingress, ConfigMaps, Secrets, Namespaces, recursos y probes.

> Este repositorio es material de laboratorio/apuntes. Nginx no es el tema: es solo la aplicación de prueba para entender cómo funciona Kubernetes.

---

## Tabla de contenidos

- [¿Qué es Kubernetes?](#qué-es-kubernetes)
- [Arquitectura](#arquitectura)
- [Conceptos fundamentales](#conceptos-fundamentales)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Recursos de Kubernetes usados](#recursos-de-kubernetes-usados)
- [Probes (comprobaciones de salud)](#probes-comprobaciones-de-salud)
- [Requests y Limits](#requests-y-limits)
- [Namespaces](#namespaces)
- [Instalación y puesta en marcha](#instalación-y-puesta-en-marcha)
- [Comandos útiles](#comandos-útiles)
- [Flujo de despliegue](#flujo-de-despliegue)
- [Ruta de aprendizaje](#ruta-de-aprendizaje)

---

## ¿Qué es Kubernetes?

Kubernetes, también conocido como **K8s**, es un sistema de código abierto para automatizar el despliegue, el escalado y la gestión de aplicaciones en contenedores.

Kubernetes agrupa los contenedores que conforman una aplicación en unidades lógicas para una fácil administración y descubrimiento. Se basa en más de 15 años de experiencia de Google ejecutando cargas de trabajo de producción, combinada con las mejores ideas y prácticas de la comunidad.

Sitio oficial: <https://kubernetes.io/es/>

### Kubernetes administrado en la nube

Los principales proveedores ofrecen Kubernetes administrado. Los conceptos que se aprenden localmente son prácticamente los mismos que en la nube:

```
              KUBERNETES
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
      EKS        AKS        GKE
      AWS       Azure      Google
```

Kubernetes **no necesita obligatoriamente una nube**. Puede ejecutarse localmente (Minikube / Kind), en servidores físicos (kubeadm) o en la nube:

```
                 KUBERNETES
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Tu PC        Servidores       Cloud
       │             │             │
   Minikube         kubeadm      AWS EKS
   / Kind           / etc.       Azure AKS
                                GCP GKE
```

---

## Arquitectura

Un **cluster** es el conjunto completo de Kubernetes: el Control Plane (cerebro) más los Nodes (workers) que ejecutan los Pods.

```
          CLUSTER KUBERNETES
                 │
         ┌───────▼────────┐
         │ CONTROL PLANE  │
         │  🧠 CEREBRO    │
         └───────┬────────┘
                 │  administra
     ┌───────────┼───────────┐
     ▼           ▼           ▼
  NODE 1      NODE 2      NODE 3
  Worker      Worker      Worker
    │           │           │
   Pods        Pods        Pods
    │
 Container
```

### Control Plane

El **Control Plane** es el "cerebro" de Kubernetes: decide y administra el cluster. Sus componentes principales son:

| Componente | Función | Pensalo como... |
| --- | --- | --- |
| `kube-apiserver` | Puerta de entrada. Recibe todas las órdenes y consultas de `kubectl`. | 🚪 Puerta de entrada |
| `etcd` | Base de datos. Guarda el estado del cluster. | 🗄️ Base de datos/estado |
| `kube-scheduler` | Decide en qué Node colocar cada Pod. | 📍 Decide dónde colocar Pods |
| `kube-controller-manager` | Mantiene el estado deseado (por ej. "quiero 3 Pods"). | 🔄 Busca cumplir el estado deseado |

Además del Control Plane, en el cluster corren **componentes de nodo y add-ons** (también visibles en `kube-system`, pero **no forman parte del Control Plane**):

| Componente | Función | Dónde corre |
| --- | --- | --- |
| `kubelet` | Agente que ejecuta y vigila los contenedores asignados a su Node. | En cada Node (Worker) |
| `kube-proxy` | Reglas de red para que los Services lleguen a los Pods. | En cada Node (DaemonSet) |
| `kindnet` | Componente de red/CNI para la conectividad de los Pods. | En cada Node (DaemonSet) |
| `coredns` | DNS interno del cluster. | Deployment en `kube-system` |
| `storage-provisioner` | Provisión de almacenamiento/volúmenes. | Add-on de Minikube |

Cuando ejecutás `kubectl get pods -A`, tanto los componentes del Control Plane como estos add-ons aparecen en el namespace **`kube-system`**, que está reservado para el sistema de Kubernetes.

### El ciclo de vida de una orden

`kubectl` **no levanta Kubernetes**: es solo el cliente que se comunica con el cluster a través del API Server.

```
minikube start
      │
      ▼
┌─────────────────┐
│   Kubernetes    │  ← está funcionando
│  API Server     │
│  Scheduler      │
│  Controller     │
│  Kubelet        │
└────────┬────────┘
         ▲
         │
      kubectl
         │
   "get nodes"
```

Ejemplo real de `kubectl run nginx --image=nginx`:

```
kubectl run nginx --image=nginx
              │
              ▼
        API Server
              │
              ▼
   Kubernetes registra el Pod
              │
              ▼
         Scheduler
              │
              ▼
      Node: minikube
              │
              ▼
          kubelet
              │
              ▼
      descarga Image "nginx"
              │
              ▼
       crea Container
              │
              ▼
      Pod → Container → Running
```

---

## Conceptos fundamentales

### 1. Cluster

El **cluster** es el conjunto completo de Kubernetes: pensalo como **todo el sistema** (Control Plane + Nodes). Puede tener uno o muchos Nodes.

### 2. Node / Worker

Un **Node** es una máquina que pertenece al cluster: física, virtual, un servidor, o (en este laboratorio) el entorno que Minikube creó sobre Docker. Proporciona CPU, memoria y recursos para ejecutar Pods.

**Worker no es algo distinto de Node**: cuando decimos *Worker Node* hablamos de un Node cuya función principal es ejecutar las aplicaciones/Pods.

Un Node puede tener **muchos Pods**. El Scheduler decide en qué Node colocar cada uno.

### 3. Pod

El **Pod es la unidad básica que Kubernetes administra** para ejecutar aplicaciones. Puede contener uno o varios containers, que comparten red y almacenamiento configurado.

```
Pod
└── Container
```

**Kubernetes administra Pods, no containers directamente.**

### 4. Container

El **container es donde corre la aplicación** (por ejemplo, un microservicio Spring Boot). Es una instancia ejecutándose a partir de una imagen.

```
Pod
└── Container
     └── Spring Boot   ← imagen: mi-backend:1.0
```

### 5. Namespace

El **Namespace es una separación lógica dentro del cluster**. No es una máquina: sirve para organizar recursos (aplicaciones, equipos o ambientes).

```
CLUSTER
│
├── Namespace: development
│     ├── Pod backend
│     └── Pod frontend
│
├── Namespace: production
│     ├── Pod backend
│     └── Pod frontend
│
└── Namespace: kube-system
      ├── Pod CoreDNS
      └── Pod kube-proxy
```

Los namespaces **no crean otros clusters**: seguís teniendo un solo cluster dividido lógicamente.

### 6. Deployment

Un **Deployment** administra el ciclo de vida de los Pods:

- definir cuántas réplicas querés;
- crear los Pods;
- reemplazar Pods que desaparecen;
- actualizar la versión de la aplicación;
- mantener el estado deseado.

```
Deployment  (replicas: 2)
    │  "quiero 2 Pods"
    ▼
ReplicaSet
    │  mantiene ese estado
    ▼
2 Pods
```

### 7. Service

Un **Service** da a los Pods un **punto de acceso estable**. Como los Pods son reemplazables (y cambian de IP), no podés depender de sus IPs.

```
          Service
       nginx-service
       IP estable
            │
    ┌───────┼───────┐
    ▼       ▼       ▼
   Pod     Pod     Pod
```

El Service encuentra sus Pods mediante **labels**:

```
Deployment  →  crea/mantiene Pods (labels: app: nginx)
Service     →  selector: app: nginx  →  les envía tráfico
```

Tipos de Service:

- **ClusterIP**: acceso solo dentro del cluster (IP interna estable).
- **NodePort**: expone el Service en un puerto de cada Node (acceso desde fuera). Ej: `80:31675/TCP` = puerto del Service `80`, NodePort `31675`.
- **LoadBalancer**: balanceador externo (típico en la nube).
- **Ingress**: ver más abajo.

> El Service **no** crea Pods, **no** los reemplaza y **no** decide réplicas. Eso es responsabilidad del Deployment/ReplicaSet. El Service solo da acceso estable.

### 8. Ingress

Un **Ingress** define reglas para recibir tráfico **HTTP/HTTPS** desde fuera del cluster y decidir a qué **Service** interno enviarlo.

```
                    INTERNET / CLIENTE
                           │
                           ▼
                       Ingress
                     /          \
                /users            /orders
                    │                │
                    ▼                ▼
            users-service      orders-service
                    │                │
                 ┌──┴──┐          ┌─┴──┐
                 ▼     ▼          ▼    ▼
                Pod   Pod        Pod  Pod
```

Clave: **Ingress no apunta directamente a los Pods, apunta a Services.**

Resuelve el problema de tener múltiples NodePort (uno por microservicio) usando una entrada común:

```
miapp.com/users         → users-service
miapp.com/orders        → orders-service
miapp.com/products      → products-service
```

Hay que separar dos cosas:

- **Ingress**: el recurso donde declarás las reglas.
- **Ingress Controller**: el componente que realmente recibe el tráfico y ejecuta esas reglas (en Minikube se instala con `minikube addons enable ingress`).

### 9. ConfigMap

Un **ConfigMap** guarda **configuración no sensible** de una aplicación, para no tener que meterla dentro de la imagen Docker.

```
Docker image ── aplicación
ConfigMap    ── configuración   ← DB_HOST=postgres, DB_PORT=5432, APP_MODE=production
```

Si cambia la configuración, no hace falta reconstruir la imagen.

> **ConfigMap NO es para información secreta.**

### 10. Secret

Un **Secret** guarda **información sensible**: usuarios, contraseñas, tokens, API keys, credenciales.

> Secret no significa que el dato quede mágicamente seguro: hay que configurar correctamente el almacenamiento y el acceso.

```
DB_HOST=postgres-service   ← ConfigMap
DB_PORT=5432               ← ConfigMap
DB_USERNAME=admin          ← Secret
DB_PASSWORD=123456         ← Secret
```

### 11. Requests y Limits

El **request** es el recurso que Kubernetes considera necesario para programar el Pod (scheduling). El **limit** es hasta dónde puede llegar el consumo.

```
resources:
  requests:                 limits:
    cpu: "100m"      ←→       cpu: "500m"
    memory: "128Mi"  ←→       memory: "256Mi"
       │                          │
   mínimo solicitado          máximo permitido
   para scheduling
```

Esto determina la **QoS Class**:

- `Burstable`: requests ≠ limits (garantía parcial, puede crecer hasta el límite).
- `Guaranteed`: requests == limits (garantía total).
- `BestEffort`: sin requests ni limits.

### 12. Probes (comprobaciones de salud)

Los **Probes** son comprobaciones que Kubernetes hace sobre los contenedores para saber cómo se encuentran. Que el contenedor esté levantado no significa que la aplicación funcione bien.

| Probe | Pregunta | Si falla |
| --- | --- | --- |
| **Liveness** | ¿Sigue funcionando? | Kubernetes puede **reiniciar** el contenedor. |
| **Readiness** | ¿Está listo para recibir tráfico? | Deja de **enviarle tráfico** vía Services. |
| **Startup** | ¿Terminó de arrancar? | Da tiempo al arranque; bloquea Liveness/Readiness hasta que pase. |

Ejemplo en Spring Boot, el endpoint `GET /actuator/health` responde `{"status":"UP"}`.

---

## Estructura del repositorio

```
.
├── README.md
└── kubernates-notion/
    ├── nginx.yaml                 # Pod simple
    ├── nginx-deployment.yaml      # Deployment (réplicas, resources, probes, env, volumen)
    ├── nginx-service.yaml         # Service tipo NodePort
    ├── nginx-ingress.yaml         # Ingress (regla /nginx → nginx-service)
    ├── configmap.yaml             # ConfigMap con nginx.conf
    ├── secret.yaml                # Secret con username/password
    └── NAMESPACE DEV/
        └── nginx-dev.yaml         # Pod dentro del namespace "dev"
```

El repo sigue una estructura típica por aplicación/servicio, pensada para escalar a microservicios:

```
users/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
└── secret.yaml
```

---

## Recursos de Kubernetes usados

### Pod (`nginx.yaml`)

```yaml
apiVersion: v1     # Versión de la API
kind: Pod          # Tipo de recurso

metadata:
  name: nginx      # Nombre del recurso

spec:
  containers:
    - name: nginx
      image: nginx
```

### Deployment (`nginx-deployment.yaml`)

Define 3 réplicas de Nginx con recursos, las tres probes, variables de entorno desde un Secret y un volumen desde un ConfigMap.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 12
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          env:
            - name: USERNAME
              valueFrom:
                secretKeyRef:
                  name: nginx-secret
                  key: username
            - name: PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nginx-secret
                  key: password
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
```

### Service (`nginx-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### Ingress (`nginx-ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
    - http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

### ConfigMap (`configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        return 200 "Hola desde Kubernetes + Nginx + ConfigMap\n";
      }
    }
```

### Secret (`secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
stringData:
  username: admin
  password: mi-password-123
```

### Pod en namespace `dev` (`NAMESPACE DEV/nginx-dev.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
```

---

## Instalación y puesta en marcha

### Instalar `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Permisos
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

### Iniciar Minikube

Minikube crea y ejecuta un cluster Kubernetes local. En este laboratorio se usa **Docker Desktop** como driver:

```bash
minikube start --driver=docker
```

```
Docker Desktop
     │  Docker Engine
     ▼
┌─────────────────────┐
│ Contenedor Minikube │
│  Kubernetes         │
│  ├─ API Server      │
│  ├─ Scheduler       │
│  ├─ Controller      │
│  ├─ etcd            │
│  └─ Kubelet         │
└─────────────────────┘
        │
        ▼
      Pods
```

Una vez configurado el driver, alcanza con `minikube start`. Verificá con `minikube status`.

### Habilitar el Ingress Controller

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

El Ingress Controller vive en el namespace **`ingress-nginx`**, no en `default`. El flag `-n` indica el namespace:

```bash
kubectl get pods -n ingress-nginx
```

---

## Comandos útiles

### kubectl

```bash
# Versión del cliente
kubectl version --client

# Consultas
kubectl get nodes
kubectl get pods
kubectl get pods -A                        # todos los namespaces
kubectl get pods -o wide                   # datos adicionales
kubectl get deployments
kubectl get replicasets
kubectl get services
kubectl get ingress
kubectl get namespaces
kubectl get configmap nginx-config
kubectl get endpointslices -l kubernetes.io/service-name=nginx-service -o yaml

# Detalle / inspección
kubectl describe pod nginx
kubectl describe pods -l app=nginx
kubectl get pod nginx -o yaml

# Creación
kubectl run nginx --image=nginx            # crea un Pod directamente
kubectl apply -f nginx.yaml                # crea desde un archivo YAML
kubectl apply --dry-run=client -f nginx-deployment.yaml

# Namespaces
kubectl create namespace dev
kubectl get pods -n dev

# Rollout
kubectl rollout status deployment/nginx-deployment

# Ejecución dentro de un Pod
kubectl exec -it <pod> -- cat /etc/nginx/conf.d/nginx.conf
kubectl exec <pod> -- env | grep -E 'USERNAME|PASSWORD'

# Eliminar
kubectl delete pod nginx
```

### Minikube

```bash
minikube version
minikube start --driver=docker
minikube start
minikube status
minikube stop
minikube ip
minikube service nginx-service
minikube addons list
minikube addons list | grep ingress
minikube addons enable ingress
```

### `kubectl run` vs YAML

- `kubectl run nginx --image=nginx` → le decís a Kubernetes **qué hacer** directamente.
- `nginx.yaml` → **describís cómo querés que sea el recurso**; Kubernetes lee el archivo y lo crea.

---

## Flujo de despliegue

### Pod → Deployment

```bash
kubectl apply -f kubernates-notion/nginx.yaml
kubectl apply -f kubernates-notion/nginx-deployment.yaml
```

### ConfigMap + Deployment

El ConfigMap se monta como volumen en `/etc/nginx/conf.d`:

```
ConfigMap (nginx.conf)
    │
    ▼
Deployment
    │
    ▼
Pod
    │
    ▼
/etc/nginx/conf.d/nginx.conf
    │
    ▼
Nginx  →  "Hola desde Kubernetes + Nginx + ConfigMap"
```

Comprobación:

```bash
kubectl exec -it <pod-del-deployment> -- cat /etc/nginx/conf.d/nginx.conf
kubectl exec -it <pod-del-deployment> -- curl http://localhost
```

### Secret + Deployment

Las credenciales se inyectan como variables de entorno:

```bash
kubectl apply -f kubernates-notion/secret.yaml
kubectl apply -f kubernates-notion/nginx-deployment.yaml
kubectl exec <pod> -- env | grep -E 'USERNAME|PASSWORD'
# USERNAME=admin
# PASSWORD=mi-password-123
```

### Exponer con NodePort

```bash
kubectl apply -f kubernates-notion/nginx-service.yaml
kubectl get service nginx-service
# PORT(S): 80:31675/TCP   → 80 interno, 31675 NodePort

minikube ip                 # IP del Node, ej: 192.168.49.2
minikube service nginx-service   # crea un túnel a 127.0.0.1:xxxxx
```

> Con el driver Docker en Linux, la IP del Node (`192.168.49.2`) pertenece a la red interna de Minikube/Docker y no es accesible directamente desde tu PC. `minikube service` crea un **túnel temporal** hacia `127.0.0.1` para poder acceder.

```
Tu navegador
     │  127.0.0.1:38203
     ▼
┌─────────────────┐
│ Túnel de Minikube│  ← temporal
└────────┬────────┘
         ▼
   NodePort 192.168.49.2:31675
         ▼
   nginx-service:80
         ▼
   Pod / Pod / Pod
```

### Exponer con Ingress

```bash
kubectl apply -f kubernates-notion/nginx-ingress.yaml
kubectl get ingress
# nginx-ingress  nginx  *  192.168.49.2  80
```

---

## Namespaces

Crear y usar un namespace:

```bash
kubectl create namespace dev
kubectl apply -f "kubernates-notion/NAMESPACE DEV/nginx-dev.yaml"
kubectl get pods -n dev
kubectl get pods -A
```

Un mismo nombre (`nginx`) puede existir en distintos namespaces:

```bash
kubectl get pod nginx          # namespace default
kubectl get pod nginx -n dev   # namespace dev
```

```
              CLUSTER KUBERNETES
                     │
       ┌─────────────┼─────────────┐
       │             │             │
     dev           stage          prod
  namespace      namespace      namespace
       │             │             │
     Pods           Pods           Pods
     Services       Services       Services
     Secrets        Secrets        Secrets
```

---

## Ruta de aprendizaje

Este repositorio sigue el camino: **PC → Minikube → aprender Kubernetes → microservicios → nube**.

El orden de estudio de los recursos es:

1. **Pod** — la unidad básica.
2. **Deployment** — réplicas y ciclo de vida.
3. **Service** — acceso estable (ClusterIP / NodePort / LoadBalancer).
4. **Ingress** — entrada HTTP/HTTPS con reglas.
5. **ConfigMap** — configuración no sensible.
6. **Secret** — información sensible.
7. **Namespace** — separación lógica.
8. **Requests y Limits** — recursos y QoS.
9. **Probes** — Liveness, Readiness y Startup.

### Objetivo final

Aplicar estos conceptos a varios microservicios (por ejemplo, `users`, `orders`, `products`) y exponerlos con Ingress hacia la nube.

```
-> Tus 5 micros -> AWS EKS
```

---

## Referencias

- Kubernetes: <https://kubernetes.io/es/>
- Minikube: <https://minikube.sigs.k8s.io/docs/>
- kubectl: <https://kubernetes.io/docs/reference/kubectl/>
