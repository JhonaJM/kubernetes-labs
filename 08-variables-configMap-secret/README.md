# Variables de Entorno, ConfigMaps y Secrets en Kubernetes

Este laboratorio cubre cómo gestionar la configuración y los datos sensibles en Kubernetes, desde el enfoque más simple (hardcodeado) hasta las mejores prácticas de producción.

## Estructura del laboratorio

```
08-variables-configMap-secret/
├── 01-env-vars-hardcoded/          Variables de entorno definidas directamente en el manifiesto
├── 02-configmap-imperativo/        ConfigMap creado con comandos kubectl (sin YAML)
├── 03-configmap-desde-fichero/     ConfigMap desde fichero: bloque único vs variables individuales
├── 04-configmap-declarativo/       ConfigMap declarativo en YAML + configMapKeyRef
├── 05-configmap-volumen/           ConfigMap montado como volumen dentro del contenedor
├── 06-secret-declarativo/          Secret declarativo: data (Base64) vs stringData (texto plano)
├── 07-secret-fichero/              Secret creado desde un fichero de texto
├── 08-secret-volumen/              Secret montado como volumen
├── 09-secret-configmap-combinados/ Mejor práctica: Secret para sensibles + ConfigMap para el resto
└── 10-secret-docker-registry/      Secret de tipo docker-registry para registros privados
```

---

## 01 — Variables de entorno hardcodeadas (el antipatrón)

### Concepto

El punto de partida y el **antipatrón** a superar. Las variables de entorno se definen directamente en el `spec` del contenedor con `env[].value`. Los problemas:
- Los valores quedan acoplados al manifiesto YAML
- Las credenciales quedan en texto claro en el repositorio
- Cambiar cualquier valor requiere modificar el YAML y hacer un nuevo despliegue

### Archivos

- `var1.yaml` — Pod simple con dos variables: `NOMBRE` y `PROPIETARIO`
- `mysql.yaml` — Deployment de MySQL con todas las credenciales hardcodeadas

### Comandos

```bash
# Crear el pod de ejemplo con variables simples
kubectl apply -f 01-env-vars-hardcoded/var1.yaml

# Crear el deployment MySQL con variables hardcodeadas
kubectl apply -f 01-env-vars-hardcoded/mysql.yaml

# Ver los pods creados
kubectl get pods

# Leer las variables desde dentro del pod
kubectl exec -it var-ejemplo -- printenv NOMBRE
kubectl exec -it var-ejemplo -- printenv PROPIETARIO

# Ver TODAS las variables del contenedor MySQL
kubectl exec -it <nombre-pod-mysql> -- bash -c "printenv | grep MYSQL"
```

### Verificación

```bash
# El pod debe estar Running
kubectl get pod var-ejemplo

# Salida esperada al leer NOMBRE:
# CURSO DE KUBERNETES

# El problema de este enfoque: las credenciales están en el YAML en claro
kubectl get deployment mysql-deploy -o yaml | grep -A8 "env:"
```

### Limpieza

```bash
kubectl delete -f 01-env-vars-hardcoded/
```

---

## 02 — ConfigMap imperativo (sin YAML)

### Concepto

La forma más rápida de crear un ConfigMap es con `kubectl create configmap`. Útil para pruebas rápidas. Los datos se pasan directamente como literales en la línea de comandos. Kubernetes almacena la información como un objeto de tipo `ConfigMap` en el cluster.

No hay archivos YAML en esta carpeta: todo se hace con comandos.

### Comandos

```bash
# Crear un ConfigMap con literales
kubectl create configmap cf1 \
  --from-literal=usuario=usu1 \
  --from-literal=password=pass1

# Listar ConfigMaps
kubectl get configmap
kubectl get cm          # abreviatura

# Ver el contenido completo
kubectl describe cm cf1

# Ver en formato YAML (muestra cómo lo almacena Kubernetes internamente)
kubectl get cm cf1 -o yaml
```

### Verificación

```bash
# Debe aparecer "cf1" en la lista
kubectl get cm

# describe debe mostrar:
# Data
# ====
# password: pass1
# usuario:  usu1
```

### Limpieza

```bash
kubectl delete cm cf1
```

---

## 03 — ConfigMap desde fichero

### Concepto

Kubernetes puede crear un ConfigMap a partir de un fichero. Hay **dos modos** con comportamiento muy diferente que se contrastan en esta carpeta:

| Modo | Flag | Resultado | Cuándo usar |
|------|------|-----------|-------------|
| Bloque único | `--from-file` | El fichero completo queda como un único string bajo una clave igual al nombre del fichero | La app espera leer un fichero completo |
| Variables individuales | `--from-env-file` | Cada línea `CLAVE=VALOR` se convierte en una clave independiente | Inyectar múltiples variables desde un `.properties` |

### Archivos

- `datos_mysql.properties` — Fichero con 4 variables MySQL en formato `KEY=VALUE`
- `game.properties` — Fichero de configuración alternativo
- `pod1.yaml` — Pod que usa `--from-file`: referencia el fichero completo como un único valor (`configMapKeyRef`)
- `pod2.yaml` — Pod que usa `--from-env-file`: inyecta todas las claves como variables individuales (`envFrom`)

### Comandos — Modo 1: bloque único (`--from-file`)

```bash
# Crear el ConfigMap: todo el fichero como un único string
kubectl create configmap datos-mysql \
  --from-file=03-configmap-desde-fichero/datos_mysql.properties

# Inspeccionar: la clave es el nombre del fichero
kubectl get cm datos-mysql -o yaml

# Crear el pod que lo consume
kubectl apply -f 03-configmap-desde-fichero/pod1.yaml

# Verificar: DATOS_MYSQL contiene todo el fichero como un único string
kubectl exec -it pod1 -- sh -c "printenv DATOS_MYSQL"
```

### Comandos — Modo 2: variables individuales (`--from-env-file`)

```bash
# Crear el ConfigMap: cada KEY=VALUE se convierte en clave independiente
kubectl create configmap datos-mysql-env \
  --from-env-file=03-configmap-desde-fichero/datos_mysql.properties

# Inspeccionar: ahora hay 4 claves separadas
kubectl get cm datos-mysql-env -o yaml

# Crear el pod que lo consume con envFrom
kubectl apply -f 03-configmap-desde-fichero/pod2.yaml

# Verificar: cada variable aparece por separado
kubectl exec -it pod1 -- sh -c "printenv | grep MYSQL"
```

### Verificación

```bash
# Con --from-file: DATOS_MYSQL es un único bloque de texto (no son variables separadas)
kubectl exec -it pod1 -- sh -c "printenv DATOS_MYSQL"
# Salida: MYSQL_ROOT_PASSWORD=kubernetes\nMYSQL_USER=usudb\n... (todo como string)

# Con --from-env-file: variables individuales accesibles por nombre
kubectl exec -it pod1 -- sh -c "printenv MYSQL_USER"
# Salida: usudb
```

### Limpieza

```bash
kubectl delete pod pod1
kubectl delete cm datos-mysql datos-mysql-env
```

---

## 04 — ConfigMap declarativo con `configMapKeyRef`

### Concepto

Un ConfigMap puede definirse como un recurso YAML y aplicarse con `kubectl apply`. El Deployment referencia cada clave individualmente con `configMapKeyRef`. Esto da **control fino**: se elige exactamente qué claves inyectar y con qué nombre de variable de entorno.

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      configMapKeyRef:
        name: datos-mysql-env   # nombre del ConfigMap
        key: MYSQL_ROOT_PASSWORD # clave dentro del ConfigMap
```

### Archivos

- `datos-mysql-env.yaml` — ConfigMap YAML con las 4 variables MySQL
- `datos_mysql.properties` — Fichero alternativo para creación imperativa
- `mysql.yaml` — Deployment que usa `configMapKeyRef` para cada variable

### Comandos

```bash
# Opción A: crear el ConfigMap desde el YAML declarativo
kubectl apply -f 04-configmap-declarativo/datos-mysql-env.yaml

# Opción B: crear el ConfigMap desde el fichero de propiedades
kubectl create configmap datos-mysql-env \
  --from-env-file=04-configmap-declarativo/datos_mysql.properties

# Inspeccionar el ConfigMap
kubectl get cm datos-mysql-env -o yaml

# Desplegar MySQL usando el ConfigMap
kubectl apply -f 04-configmap-declarativo/mysql.yaml

# Ver el deployment y el pod
kubectl get deployment mysql-deploy
kubectl get pods -l app=mysql
```

### Verificación

```bash
# El pod MySQL debe estar Running
kubectl get pods -l app=mysql

# Verificar las variables dentro del contenedor
kubectl exec -it <nombre-pod-mysql> -- bash -c "printenv | grep MYSQL"
# Salida esperada:
# MYSQL_ROOT_PASSWORD=kubernetes
# MYSQL_USER=usudb
# MYSQL_PASSWORD=usupass
# MYSQL_DATABASE=kubernetes

# Conectar a MySQL para confirmar que arrancó correctamente
kubectl exec -it <nombre-pod-mysql> -- mysql -u usudb -pusupass kubernetes -e "SHOW DATABASES;"
```

### Limpieza

```bash
kubectl delete -f 04-configmap-declarativo/
```

---

## 05 — ConfigMap montado como volumen

### Concepto

Además de inyectarse como variables de entorno, un ConfigMap puede **montarse como un directorio** dentro del contenedor. Kubernetes crea un fichero por cada clave del ConfigMap:
- El nombre del fichero = la clave
- El contenido del fichero = el valor

Útil para aplicaciones que leen su configuración desde ficheros en disco (nginx, prometheus, etc.).

```yaml
volumeMounts:
  - name: volumen-config-map
    mountPath: /etc/config-map   # directorio donde aparecen los ficheros
volumes:
  - name: volumen-config-map
    configMap:
      name: config-volumen       # nombre del ConfigMap
```

### Archivos

- `configmap.yaml` — ConfigMap con dos claves: `ENTORNO` y `VERSION`
- `pod1.yaml` — Pod que monta el ConfigMap en `/etc/config-map`

### Comandos

```bash
# Crear el ConfigMap
kubectl apply -f 05-configmap-volumen/configmap.yaml

# Crear el pod con el volumen montado
kubectl apply -f 05-configmap-volumen/pod1.yaml

# Explorar el punto de montaje dentro del contenedor
kubectl exec -it pod1 -- sh

# Dentro del contenedor:
ls /etc/config-map
# Salida: ENTORNO  VERSION

cat /etc/config-map/ENTORNO
# Salida: desarrollo

cat /etc/config-map/VERSION
# Salida: 1.0
```

### Verificación

```bash
# Comprobar que el volumen está montado
kubectl describe pod pod1 | grep -A5 "Mounts:"
# Debe mostrar: /etc/config-map from volumen-config-map (ro)

# Los datos NO aparecen como variables de entorno (son ficheros, no env vars)
kubectl exec -it pod1 -- sh -c "printenv | grep ENTORNO"
# No debe mostrar nada (es un fichero, no una variable)
```

### Limpieza

```bash
kubectl delete -f 05-configmap-volumen/
```

---

## 06 — Secrets declarativos (`data` vs `stringData`)

### Concepto

Los Secrets funcionan como ConfigMaps pero están diseñados para **datos sensibles**. Kubernetes los almacena codificados en Base64 (no es cifrado, solo ofuscación). La diferencia al crearlos declarativamente:

| Campo | Formato de entrada | Qué hace Kubernetes |
|-------|-------------------|---------------------|
| `data` | Base64 | Lo almacena tal cual |
| `stringData` | Texto plano | Lo codifica en Base64 automáticamente |

Codificar/decodificar Base64 manualmente:
```bash
echo -n "usu1" | base64          # codificar → dXN1MQ==
echo "dXN1MQ==" | base64 --decode  # decodificar → usu1
```

### Archivos

- `secreto1.yaml` — Secret con `data` (valores ya en Base64)
- `secreto2.yaml` — Secret con `stringData` (valores en texto plano)
- `pod1.yaml` — Pod que usa `envFrom` para cargar ambos secrets como variables de entorno

### Comandos

```bash
# Crear ambos Secrets
kubectl apply -f 06-secret-declarativo/secreto1.yaml
kubectl apply -f 06-secret-declarativo/secreto2.yaml

# Listar Secrets (el contenido no se muestra en la lista)
kubectl get secrets

# Inspeccionar el contenido (siempre en Base64)
kubectl get secret secreto1 -o yaml
kubectl get secret secreto2 -o yaml

# Decodificar un valor manualmente
kubectl get secret secreto1 -o jsonpath='{.data.usuario1}' | base64 --decode
# Salida: usu1

# Crear el pod que consume ambos secrets
kubectl apply -f 06-secret-declarativo/pod1.yaml

# Verificar variables dentro del contenedor (Kubernetes decodifica automáticamente)
kubectl exec -it pod1 -- sh -c "printenv usuario1"
kubectl exec -it pod1 -- sh -c "printenv usuario2"
```

### Verificación

```bash
# Los valores dentro del contenedor deben estar decodificados
kubectl exec -it pod1 -- sh -c "printenv usuario1"
# Salida: usu1

kubectl exec -it pod1 -- sh -c "printenv usuario2"
# Salida: usu2

# Pero en el objeto Secret de Kubernetes siempre están en Base64
kubectl get secret secreto1 -o jsonpath='{.data.usuario1}'
# Salida: dXN1MQ==
```

### Limpieza

```bash
kubectl delete -f 06-secret-declarativo/
```

---

## 07 — Secret desde fichero

### Concepto

Un Secret puede crearse a partir del contenido de un fichero. Kubernetes codifica el contenido en Base64 automáticamente. El fichero entero se convierte en una clave cuyo nombre es el nombre del fichero. Al inyectarlo como variable de entorno, Kubernetes decodifica el Base64 y el contenedor recibe el texto original.

### Archivos

- `datos.txt` — Fichero de texto con contenido de ejemplo
- `pod1.yaml` — Pod que inyecta el contenido del fichero como variable `DATOS` vía `secretKeyRef`

### Comandos

```bash
# Ver el contenido del fichero antes de crear el secret
cat 07-secret-fichero/datos.txt

# Crear el Secret desde el fichero (Kubernetes codifica en Base64 automáticamente)
kubectl create secret generic datos \
  --from-file=07-secret-fichero/datos.txt

# Inspeccionar: el valor aparece en Base64
kubectl get secret datos -o yaml

# Decodificar manualmente para verificar
kubectl get secret datos -o jsonpath='{.data.datos\.txt}' | base64 --decode

# Crear el pod
kubectl apply -f 07-secret-fichero/pod1.yaml

# Verificar (Kubernetes decodifica automáticamente)
kubectl exec -it pod1 -- sh -c "printenv DATOS"
```

### Verificación

```bash
# Decodificar desde Kubernetes debe dar el mismo resultado que leer el fichero
kubectl get secret datos -o jsonpath='{.data.datos\.txt}' | base64 --decode
# Salida:
# Esto es un ejemplo de secrets
# incorporaos desde fichero
# dentro de un contenedor

# Dentro del pod, la variable DATOS tiene el contenido decodificado
kubectl exec -it pod1 -- sh -c "printenv DATOS"
# Misma salida que arriba
```

### Limpieza

```bash
kubectl delete pod pod1
kubectl delete secret datos
```

---

## 08 — Secret montado como volumen

### Concepto

Al igual que los ConfigMaps, los Secrets pueden montarse como volúmenes. Kubernetes crea un fichero por cada clave del Secret con el valor decodificado como contenido. Útil para certificados TLS, ficheros de credenciales o claves SSH que la aplicación espera leer desde disco.

```yaml
volumeMounts:
  - name: volumen-secretos
    mountPath: /tmp/datos
volumes:
  - name: volumen-secretos
    secret:
      secretName: secreto-volumen   # nombre del Secret
```

### Archivos

- `secretoVolumen.yaml` — Secret con dos claves: `DATO1` y `DATO2` (usando `stringData`)
- `pod1.yaml` — Pod que monta el Secret en `/tmp/datos`

### Comandos

```bash
# Crear el Secret
kubectl apply -f 08-secret-volumen/secretoVolumen.yaml

# Crear el pod con el volumen de secretos montado
kubectl apply -f 08-secret-volumen/pod1.yaml

# Explorar el punto de montaje
kubectl exec -it pod1 -- sh

# Dentro del contenedor:
ls /tmp/datos
# Salida: DATO1  DATO2

cat /tmp/datos/DATO1
# Salida: dato1
```

### Verificación

```bash
# Comprobar el volumen montado
kubectl describe pod pod1 | grep -A5 "Mounts:"
# Debe mostrar: /tmp/datos from volumen-secretos (ro)

# Los datos NO aparecen como variables de entorno
kubectl exec -it pod1 -- sh -c "printenv | grep DATO"
# No debe mostrar nada

# Los datos SÍ existen como ficheros
kubectl exec -it pod1 -- cat /tmp/datos/DATO1
# Salida: dato1
```

### Limpieza

```bash
kubectl delete -f 08-secret-volumen/
```

---

## 09 — Mejor práctica: Secret + ConfigMap combinados

### Concepto

La **mejor práctica en producción** es separar la configuración por sensibilidad:

- **ConfigMap** → datos no sensibles: nombre de base de datos, usuario, host, puerto
- **Secret** → datos sensibles: contraseñas, tokens, claves API

Ventajas:
- El ConfigMap puede versionarse en git sin riesgo
- El Secret se gestiona con controles de acceso distintos (Vault, RBAC, Sealed Secrets)
- Kubernetes aplica políticas de auditoría distintas a Secrets

El Deployment mezcla ambos tipos de referencia en el mismo contenedor:

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:         # viene del Secret
        name: passwords
        key: pass-root
  - name: MYSQL_USER
    valueFrom:
      configMapKeyRef:      # viene del ConfigMap
        name: datos-mysql-env
        key: MYSQL_USER
```

### Archivos

- `datos-mysql-env.yaml` — ConfigMap con solo `MYSQL_USER` y `MYSQL_DATABASE` (sin contraseñas)
- `mysql.yaml` — Deployment que mezcla `secretKeyRef` para contraseñas y `configMapKeyRef` para el resto

### Comandos

```bash
# 1. Crear el ConfigMap con los datos no sensibles
kubectl apply -f 09-secret-configmap-combinados/datos-mysql-env.yaml

# 2. Crear el Secret con las contraseñas (imperativo para NO guardar en git)
kubectl create secret generic passwords \
  --from-literal=pass-root=kubernetes \
  --from-literal=pass-usu=usupass

# Verificar que el secret se creó correctamente
kubectl get secret passwords -o yaml

# 3. Desplegar MySQL que usa ambos recursos
kubectl apply -f 09-secret-configmap-combinados/mysql.yaml

# Ver el estado del deployment
kubectl get deployment mysql-deploy
kubectl get pods -l app=mysql
```

### Verificación

```bash
# El pod MySQL debe estar Running
kubectl get pods -l app=mysql

# Verificar las variables (provienen de dos fuentes distintas)
kubectl exec -it <nombre-pod-mysql> -- bash -c "printenv | grep MYSQL"
# MYSQL_ROOT_PASSWORD=kubernetes  <- vino del Secret
# MYSQL_PASSWORD=usupass          <- vino del Secret
# MYSQL_USER=usudb                <- vino del ConfigMap
# MYSQL_DATABASE=kubernetes       <- vino del ConfigMap

# Conectar a MySQL
kubectl exec -it <nombre-pod-mysql> -- mysql -u usudb -pusupass kubernetes -e "SHOW DATABASES;"
```

### Limpieza

```bash
kubectl delete -f 09-secret-configmap-combinados/
kubectl delete secret passwords
```

---

## 10 — Secret de registro Docker privado (`imagePullSecrets`)

### Concepto

Kubernetes necesita credenciales para descargar imágenes de registros privados. Se usa un Secret de tipo `docker-registry` y se referencia en el Pod con `imagePullSecrets`. Sin este secret, el pod fallará con `ImagePullBackOff` o `ErrImagePull`.

```yaml
spec:
  containers:
    - name: ejemplo
      image: apasoft/web1      # imagen en registro privado
  imagePullSecrets:
    - name: midocker           # nombre del Secret de tipo docker-registry
```

### Archivos

- `pod-secret-docker.yaml` — Pod que usa `imagePullSecrets` para acceder a `apasoft/web1` en Docker Hub privado

### Comandos

```bash
# Crear el Secret de tipo docker-registry (imperativo)
kubectl create secret docker-registry midocker \
  --docker-server=docker.io \
  --docker-username=TU_USUARIO \
  --docker-password=TU_PASSWORD \
  --docker-email=TU_EMAIL

# Inspeccionar el secret creado
kubectl get secret midocker
kubectl get secret midocker -o yaml

# Desplegar el pod que usa el registry secret
kubectl apply -f 10-secret-docker-registry/pod-secret-docker.yaml

# Ver el estado del pod
kubectl get pod ejemplo
kubectl describe pod ejemplo
```

### Verificación

```bash
# Si el secret es válido, el pod debe quedar Running
kubectl get pod ejemplo

# Si hay problemas con las credenciales se verá en los eventos:
kubectl describe pod ejemplo | grep -A5 "Events:"
# ErrImagePull o ImagePullBackOff indica credenciales incorrectas

# Verificar que el secret está referenciado en el pod
kubectl get pod ejemplo -o yaml | grep -A3 "imagePullSecrets"
# Debe mostrar: - name: midocker
```

### Limpieza

```bash
kubectl delete pod ejemplo
kubectl delete secret midocker
```

---

## Resumen comparativo

| Concepto | Tipo de dato | Cómo se crea | Cómo se inyecta | Caso de uso |
|----------|-------------|--------------|-----------------|-------------|
| `env` directo | Cualquiera | En el YAML del Pod | `value:` directamente | Antipatrón — solo pruebas locales |
| ConfigMap literal | No sensible | `--from-literal` | `configMapKeyRef` / `envFrom` | Configuración simple |
| ConfigMap fichero bloque | No sensible | `--from-file` | `configMapKeyRef` (clave = nombre fichero) | App lee fichero completo como string |
| ConfigMap env-file | No sensible | `--from-env-file` | `envFrom.configMapRef` | Múltiples vars desde `.properties` |
| ConfigMap declarativo | No sensible | `kubectl apply -f` | `configMapKeyRef` por clave | Control fino de variables |
| ConfigMap volumen | No sensible | `kubectl apply -f` | `volumeMounts` | App lee configuración desde disco |
| Secret declarativo | Sensible | YAML con `data` o `stringData` | `secretKeyRef` / `envFrom.secretRef` | Credenciales en YAML versionado |
| Secret fichero | Sensible | `--from-file` | `secretKeyRef` | Certificados, ficheros de token |
| Secret volumen | Sensible | `kubectl apply -f` | `volumeMounts` | Certs TLS, SSH keys en disco |
| Secret + ConfigMap | Mixto | Creados por separado | Mezcla de `secretKeyRef` + `configMapKeyRef` | Mejor práctica en producción |
| `docker-registry` | Credenciales | `--docker-*` flags | `imagePullSecrets` | Registros de imágenes privados |

---

## Comandos de referencia rápida

```bash
# --- ConfigMaps ---
kubectl get cm
kubectl get cm <nombre> -o yaml
kubectl describe cm <nombre>
kubectl delete cm <nombre>

# Creación imperativa
kubectl create configmap <nombre> --from-literal=CLAVE=VALOR
kubectl create configmap <nombre> --from-file=<fichero>
kubectl create configmap <nombre> --from-env-file=<fichero>

# --- Secrets ---
kubectl get secrets
kubectl get secret <nombre> -o yaml
kubectl describe secret <nombre>
kubectl delete secret <nombre>

# Decodificar un valor de un secret
kubectl get secret <nombre> -o jsonpath='{.data.<clave>}' | base64 --decode

# Creación imperativa
kubectl create secret generic <nombre> --from-literal=CLAVE=VALOR
kubectl create secret generic <nombre> --from-file=<fichero>
kubectl create secret docker-registry <nombre> \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<pass> \
  --docker-email=<email>

# --- Verificación dentro de un pod ---
kubectl exec -it <pod> -- printenv
kubectl exec -it <pod> -- printenv | grep MYSQL
kubectl exec -it <pod> -- ls /etc/config-map
kubectl exec -it <pod> -- cat /etc/config-map/<clave>
```
