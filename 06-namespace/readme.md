# 🗂️ Namespaces en Kubernetes

## 📁 Estructura del proyecto

```
📦 06-namespace/
 ┣ 📄 namespace.yaml
 ┣ 📄 deploy_elastic.yaml
 ┗ 📄 limitex.yaml
```

---

## 🧠 ¿Qué es un Namespace?

Un **Namespace** es una forma de dividir un clúster de Kubernetes en **entornos virtuales aislados** dentro del mismo clúster físico. Es como tener varios clústeres lógicos dentro de uno solo.

```
┌─────────────────────────────────────────────┐
│              CLÚSTER KUBERNETES             │
│                                             │
│   ┌──────────────┐   ┌──────────────┐      │
│   │  namespace:  │   │  namespace:  │      │
│   │   default    │   │     dev1     │      │
│   │              │   │              │      │
│   │  [Pod]       │   │  [Pod]       │      │
│   │  [Pod]       │   │  [Pod]       │      │
│   │  [Service]   │   │  [Deploy]    │      │
│   └──────────────┘   └──────────────┘      │
└─────────────────────────────────────────────┘
```

Los recursos dentro de namespaces distintos están **aislados entre sí** — un Deployment en `dev1` no interfiere con uno en `default`, aunque tengan el mismo nombre.

### ¿Para qué se usan en la vida real?

| Caso de uso | Ejemplo |
|---|---|
| Separar entornos | `dev`, `staging`, `production` |
| Separar equipos | `equipo-backend`, `equipo-frontend` |
| Limitar recursos | Asignar CPU/RAM máxima por equipo o entorno |
| Organizar proyectos | `proyecto-a`, `proyecto-b` |

### Namespaces que vienen por defecto

```bash
kubectl get namespace
```

| Namespace | Descripción |
|---|---|
| `default` | Donde se crean los recursos si no se especifica namespace |
| `kube-system` | Recursos internos de Kubernetes (DNS, scheduler, etc.) |
| `kube-public` | Accesible públicamente, usado para info del clúster |
| `kube-node-lease` | Mantenimiento de disponibilidad de los nodos |

---

## 1️⃣ Inspeccionar Namespaces

```bash
# Listar todos los namespaces
kubectl get namespace

# Ver detalles de un namespace específico
kubectl get namespace default
kubectl describe namespace default

# Ver pods dentro de un namespace específico
kubectl get pods -n default

# Ver pods en TODOS los namespaces
kubectl get pods --all-namespaces
```

---

## 2️⃣ Crear y Eliminar Namespaces

### Modo imperativo (desde terminal)

```bash
kubectl create namespace dev1
```

### Modo declarativo (desde YAML)

#### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev1
  labels:
     tipo: desarrollo
```

```bash
kubectl apply -f namespace.yaml
```

### Eliminar un namespace

```bash
kubectl delete namespace dev1
```

> ⚠️ **Cuidado:** Eliminar un namespace **destruye todos los recursos que contiene** (Pods, Deployments, Services, etc.). No hay confirmación adicional.

---

## 3️⃣ Crear recursos dentro de un Namespace

Al crear un recurso, si no se especifica namespace, va al namespace `default`. Para asignarlo a uno específico tienes dos opciones:

### Durante el apply (flag `--namespace`)

```bash
kubectl apply -f deploy_elastic.yaml --namespace=dev1
```

### Dentro del propio YAML (campo `namespace`)

```yaml
metadata:
  name: elastic
  namespace: dev1   # ← se despliega en dev1 siempre
```

### Verificar que el recurso está en el namespace correcto

```bash
kubectl describe deploy elastic -n dev1
kubectl get all -n dev1
```

---

## 4️⃣ Establecer un Namespace por defecto

Para no tener que escribir `-n dev1` en cada comando, puedes cambiar el namespace activo del contexto actual:

```bash
# Ver la configuración actual del clúster y contexto
kubectl config view

# Cambiar el namespace por defecto al contexto actual
kubectl config set-context --current --namespace=dev1

# A partir de aquí, todos los comandos apuntan a dev1 sin necesidad de -n
kubectl get pods   # ← ya busca en dev1
```

> 💡 Para volver al namespace `default`:
> ```bash
> kubectl config set-context --current --namespace=default
> ```

---

## 5️⃣ Deploy de Elasticsearch en el Namespace `dev1`

#### `deploy_elastic.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elastic
  labels:
    tipo: "desarrollo"
spec:
  selector:
    matchLabels:
      app: elastic
  replicas: 2
  strategy:
     type: RollingUpdate    # ← estrategia de actualización (ver nota abajo)
  minReadySeconds: 2
  template:
    metadata:
      labels:
        app: elastic
    spec:
      containers:
      - name: elastic
        image: elasticsearch:7.6.0
        ports:
        - containerPort: 9200
```

### 🔍 Campos nuevos en este Deployment

| Campo | Descripción |
|---|---|
| `strategy.type: RollingUpdate` | Define cómo se actualizan los Pods al cambiar la imagen (ver nota) |
| `minReadySeconds: 2` | Tiempo mínimo que un Pod debe estar listo antes de considerarse disponible |
| `containerPort: 9200` | Puerto por defecto de la API REST de Elasticsearch |

> 📌 **Nota sobre `RollingUpdate`:** Esta estrategia reemplaza los Pods de forma gradual, uno a uno, sin causar downtime. Se verá en detalle en el capítulo **07-rolling-updates**. Aquí el profesor lo incluyó como adelanto en el Deployment de Elasticsearch.

```bash
# Ver el historial de actualizaciones del deployment
kubectl rollout history deploy elastic -n dev1
```

---

## 6️⃣ Limitar recursos con LimitRange

Un **LimitRange** permite controlar cuánta CPU y memoria pueden consumir los contenedores dentro de un namespace. Es útil para evitar que un Pod consuma todos los recursos del clúster.

#### `limitex.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: recursos
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 1
    defaultRequest:
      memory: 256Mi
      cpu: 0.5
    max:
      memory: 1Gi
      cpu: 4
    min:
      memory: 128Mi
      cpu: 0.5
    type: Container
```

### 🔍 Explicación de los campos

| Campo | Valor | Descripción |
|---|---|---|
| `default` | 512Mi / 1 cpu | Límite asignado si el contenedor **no especifica** límites |
| `defaultRequest` | 256Mi / 0.5 cpu | Recursos **solicitados** si el contenedor no especifica nada |
| `max` | 1Gi / 4 cpu | Máximo que un contenedor puede **pedir o usar** |
| `min` | 128Mi / 0.5 cpu | Mínimo que un contenedor debe **solicitar** |
| `type: Container` | — | Aplica las reglas a nivel de contenedor |

```bash
# Aplicar el LimitRange al namespace dev1
kubectl apply -f limitex.yaml --namespace=dev1

# Verificar que se aplicó
kubectl describe limitrange recursos -n dev1
```

> 💡 Si un Pod intenta arrancar en el namespace con más recursos de los permitidos por el `max`, Kubernetes lo rechaza con error. Si no especifica recursos, se le asignan los valores `default` automáticamente.

---

## 7️⃣ Controlar eventos dentro de un Namespace

Los eventos son registros de lo que ocurre dentro del clúster: Pods que no arrancan, fallas de scheduling, errores de imagen, etc.

```bash
# Ver todos los eventos del namespace dev1
kubectl get events --namespace dev1

# Filtrar solo advertencias
kubectl get events --namespace dev1 --field-selector type="Warning"

# Filtrar por razón específica
kubectl get events --namespace dev1 --field-selector reason="Failed"
```

---

## 8️⃣ Panel visual de Kubernetes (Dashboard)

Para levantar el Dashboard web de Kubernetes en Minikube:

```bash
# Lanzar el dashboard en el navegador
minikube dashboard

# Si quieres solo la URL sin abrir el navegador
minikube dashboard --url
```

> El Dashboard permite ver y gestionar todos los recursos del clúster (Pods, Deployments, Namespaces, LimitRanges, eventos, etc.) de forma visual desde el navegador.

---

## 🧰 Referencia rápida de comandos

```bash
# Namespaces
kubectl get namespace
kubectl create namespace <nombre>
kubectl delete namespace <nombre>
kubectl apply -f namespace.yaml

# Recursos en namespace específico
kubectl apply -f <archivo>.yaml --namespace=<ns>
kubectl get all -n <ns>
kubectl describe deploy <nombre> -n <ns>

# Cambiar namespace por defecto
kubectl config set-context --current --namespace=<ns>

# LimitRange
kubectl apply -f limitex.yaml --namespace=<ns>
kubectl describe limitrange <nombre> -n <ns>

# Eventos
kubectl get events --namespace <ns>
kubectl get events --namespace <ns> --field-selector type="Warning"

# Dashboard
minikube dashboard
```

---

## ✅ Conceptos clave aprendidos

- Qué es un **Namespace** y para qué se usa en equipos y entornos reales
- Cómo crear namespaces de forma **imperativa y declarativa**
- Cómo **desplegar recursos en un namespace específico**
- Cómo **cambiar el namespace activo** para no repetir `-n` en cada comando
- Qué es un **LimitRange** y cómo protege el clúster de consumo excesivo de recursos
- Cómo **monitorear eventos** dentro de un namespace
- Cómo levantar el **Dashboard visual** de Kubernetes en Minikube
- Primera mención de `RollingUpdate` como estrategia de actualización (próximo capítulo)
