# 🚀 Pods en Kubernetes

## ¿Qué es un Pod?

Un **Pod** es la unidad mínima de despliegue en Kubernetes. Representa uno o más contenedores que comparten:

- La misma red (misma IP dentro del clúster)
- El mismo almacenamiento (volúmenes compartidos)
- El mismo ciclo de vida

> En la mayoría de casos prácticos, un Pod contiene **un solo contenedor**.

---

## 📁 Estructura del proyecto

```
📦 02-pods/
 ┣ 📄 Dockerfile
 ┗ 📄 nginx-pod.yaml
```

---

## 🐳 Dockerfile — Imagen personalizada de NGINX

Para este ejemplo se construyó una imagen Docker personalizada basada en **Ubuntu** con **NGINX** instalado.

```dockerfile
FROM ubuntu

RUN apt-get update

ENV TZ=Europe/Madrid

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN apt-get install -y nginx

RUN echo 'Ejemplo de POD con KUBERNETES y YAML' > /var/www/html/index.html

ENTRYPOINT ["/usr/sbin/nginx", "-g", "daemon off;"]

EXPOSE 80
```

### 🔍 Explicación paso a paso

| Instrucción | Descripción |
|---|---|
| `FROM ubuntu` | Usa Ubuntu como imagen base |
| `RUN apt-get update` | Actualiza los repositorios del sistema |
| `ENV TZ=Europe/Madrid` | Define la zona horaria como variable de entorno |
| `RUN ln -snf ...` | Configura el timezone en el sistema de archivos |
| `RUN apt-get install -y nginx` | Instala NGINX en el contenedor |
| `RUN echo '...' > index.html` | Crea una página HTML personalizada en el directorio raíz de NGINX |
| `ENTRYPOINT [...]` | Arranca NGINX en primer plano (obligatorio para contenedores) |
| `EXPOSE 80` | Documenta que el contenedor escucha en el puerto 80 |

> ⚠️ Se usa `ENTRYPOINT` en lugar de `CMD` para que el comando de arranque **no pueda ser sobreescrito** al crear el contenedor.

### 📦 Build y Push de la imagen

```bash
# Construir la imagen
docker build -t jhonajm/kubernetes-practice-nginx:v1 .

# Subir la imagen a Docker Hub
docker push jhonajm/kubernetes-practice-nginx:v1
```

---

## ☸️ Manifiesto YAML — Definición del Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    zone: prod
    version: v1
spec:
  containers:
   - name: nginx
     image: jhonajm/kubernetes-practice-nginx:v1
```

### 🔍 Explicación de los campos

| Campo | Descripción |
|---|---|
| `apiVersion: v1` | Versión de la API de Kubernetes para este recurso |
| `kind: Pod` | Tipo de recurso a crear |
| `metadata.name` | Nombre del Pod dentro del clúster |
| `metadata.labels` | Etiquetas clave-valor para identificar y organizar el Pod |
| `spec.containers` | Lista de contenedores que correrán dentro del Pod |
| `containers.name` | Nombre del contenedor dentro del Pod |
| `containers.image` | Imagen Docker a utilizar (publicada en Docker Hub) |

---

## ▶️ Comandos para desplegar y gestionar el Pod

```bash
# Crear el Pod en el clúster
kubectl apply -f nginx-pod.yaml

# Verificar que el Pod está corriendo
kubectl get pods

# Ver información detallada del Pod
kubectl describe pod nginx

# Ver los logs del contenedor
kubectl logs nginx

# Acceder al contenedor de forma interactiva
kubectl exec -it nginx -- bash

# Eliminar el Pod
kubectl delete pod nginx
# ó con el fichero
kubectl delete -f nginx-pod.yaml
```

---

## ✅ Resultado esperado

Al ejecutar `kubectl get pods` deberías ver algo como:

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s
```

El Pod estará corriendo un servidor NGINX que sirve la página:

```
Ejemplo de POD con KUBERNETES y YAML
```

---

## 📌 Conceptos clave aprendidos

- Qué es un Pod y su rol en Kubernetes
- Cómo construir y publicar una imagen Docker personalizada
- Estructura de un manifiesto YAML para Pods
- Uso de `labels` para identificar recursos
- Comandos esenciales de `kubectl` para gestionar Pods
