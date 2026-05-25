# Super Lab: Gestión del Estado, Persistencia y Reconstrucción de AKS Store Demo desde Cero

## Propósito

> **Versión editada:** esta guía fue ajustada para un escenario donde se eliminaron por error los recursos de Azure. Ahora el laboratorio incluye creación de Resource Group, Azure Container Registry, AKS, namespace, despliegue del AKS Store Demo y Redis antes de iniciar los ejercicios de arquitectura.

Este laboratorio usa **AKS Store Demo** como aplicación base para enseñar, con una aplicación real en AKS, los principales retos de arquitectura en microservicios:

- Estado distribuido
- Persistencia en microservicios
- Mensajería asíncrona
- Eventual consistency
- Fallos parciales
- Sagas conceptuales
- Cache distribuido con Redis
- CAP Theorem aplicado
- Observabilidad básica con logs y eventos de Kubernetes
- Discusión arquitectónica: 2PC vs Sagas vs eventos

La idea no es solo “desplegar una app”, sino **romperla controladamente** para que los estudiantes entiendan qué ocurre cuando los sistemas distribuidos fallan.

---

## Duración estimada

**6 horas si se crea todo desde cero**  
**5 horas si el instructor crea AKS/ACR antes de la clase**

| Bloque | Duración |
|---|---:|
| 0. Reconstrucción de Azure: RG, ACR, AKS y conexión | 60–75 min |
| 1. Deploy de AKS Store Demo | 30 min |
| 2. Exploración de arquitectura | 35 min |
| 3. Event-driven con RabbitMQ | 40 min |
| 4. Fallo parcial: makeline-service | 40 min |
| 5. Fallo parcial: RabbitMQ | 35 min |
| 6. Fallo parcial: MongoDB | 30 min |
| 7. Redis Cache Lab | 40 min |
| 8. CAP + Sagas + discusión guiada | 30 min |
| 9. Checklist, conclusiones y limpieza | 20 min |

> Si solo tienes 5 horas, el instructor debe ejecutar la sección 0 antes de clase y comenzar con el bloque 1.

---|---:|
| 0. Preparación del entorno | 20 min |
| 1. Deploy de AKS Store Demo | 30 min |
| 2. Exploración de arquitectura | 35 min |
| 3. Event-driven con RabbitMQ | 40 min |
| 4. Fallo parcial: makeline-service | 40 min |
| 5. Fallo parcial: RabbitMQ | 35 min |
| 6. Fallo parcial: MongoDB | 30 min |
| 7. Redis Cache Lab | 40 min |
| 8. CAP + Sagas + discusión guiada | 30 min |
| 9. Checklist, conclusiones y limpieza | 20 min |

---

## Arquitectura base del laboratorio

Usaremos el repositorio:

```text
https://github.com/Azure-Samples/aks-store-demo
```

La aplicación tiene estos componentes principales:

| Componente | Rol |
|---

## Decisión sobre ACR en este laboratorio

El laboratorio crea un **Azure Container Registry (ACR)** aunque el AKS Store Demo usa imágenes públicas desde `ghcr.io`.

Motivo:

- dejar el ambiente preparado para prácticas de CI/CD
- permitir futuras prácticas con Jenkins o GitHub Actions
- enseñar integración AKS + ACR
- mantener una arquitectura enterprise más realista

Para este ejercicio, el despliegue inicial usa imágenes públicas para reducir tiempo y evitar pasos de build innecesarios.

---

|---|
| `store-front` | Frontend para clientes |
| `store-admin` | UI administrativa para empleados |
| `order-service` | Servicio para colocar órdenes |
| `product-service` | Servicio para productos |
| `makeline-service` | Procesa órdenes desde RabbitMQ |
| `virtual-customer` | Simula creación de órdenes |
| `virtual-worker` | Simula procesamiento de órdenes |
| `mongodb` | Persistencia de datos |
| `rabbitmq` | Cola de órdenes |
| `redis` | Se agregará como extensión del laboratorio |

---

## Narrativa pedagógica

El flujo base será:

```text
store-front / virtual-customer
        |
        v
order-service
        |
        v
RabbitMQ queue: orders
        |
        v
makeline-service
        |
        v
MongoDB
```

Mensaje clave:

> La orden no necesariamente se completa en una sola transacción. Se recibe, se encola, se procesa después y eventualmente llega a un estado final.

---

# 0. Reconstrucción del entorno en Azure Cloud Shell

## 0.1 Abrir Azure Cloud Shell

Desde Azure Portal:

1. Abrir **Cloud Shell**
2. Seleccionar **Bash**
3. Validar que estás en la suscripción correcta

```bash
az account show -o table
```

Ver todas las suscripciones disponibles:

```bash
az account list -o table
```

Si debes cambiar de suscripción:

```bash
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
```

---

## 0.2 Definir variables del laboratorio

> Ajusta `LOCATION` si tu suscripción tiene restricciones de capacidad. Si `eastus2` genera error de capacidad, prueba `westeurope`, `northeurope`, `centralus` o la región asignada por el instructor.

```bash
export PREFIX="ericssonstate"
export RANDOM_ID=$RANDOM

export LOCATION="eastus2"
export RG_NAME="rg-${PREFIX}-${RANDOM_ID}"
export ACR_NAME="${PREFIX}acr${RANDOM_ID}"
export AKS_NAME="aks-${PREFIX}-${RANDOM_ID}"
export NS="aks-store-state-lab"

export NODE_COUNT=2
export NODE_SIZE="Standard_D4s_v5"
```

Validar variables:

```bash
echo "Resource Group: $RG_NAME"
echo "ACR:            $ACR_NAME"
echo "AKS:            $AKS_NAME"
echo "Namespace:      $NS"
echo "Location:       $LOCATION"
echo "Node size:      $NODE_SIZE"
```

---

## 0.3 Registrar proveedores de recursos

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.Insights
```

Validar:

```bash
az provider show --namespace Microsoft.ContainerService --query registrationState -o tsv
az provider show --namespace Microsoft.ContainerRegistry --query registrationState -o tsv
```

Resultado esperado:

```text
Registered
```

---

## 0.4 Crear Resource Group

```bash
az group create \
  --name $RG_NAME \
  --location $LOCATION
```

Validar:

```bash
az group show --name $RG_NAME -o table
```

---

## 0.5 Crear Azure Container Registry

> El AKS Store Demo oficial usa imágenes públicas desde `ghcr.io/azure-samples/aks-store-demo`. Aun así, creamos ACR porque es útil para prácticas posteriores de CI/CD, Jenkins, GitHub Actions, importación de imágenes o despliegues privados.

```bash
az acr create \
  --resource-group $RG_NAME \
  --name $ACR_NAME \
  --sku Basic \
  --admin-enabled false
```

Validar ACR:

```bash
az acr show \
  --resource-group $RG_NAME \
  --name $ACR_NAME \
  -o table
```

Obtener login server:

```bash
export ACR_LOGIN_SERVER=$(az acr show \
  --resource-group $RG_NAME \
  --name $ACR_NAME \
  --query loginServer -o tsv)

echo $ACR_LOGIN_SERVER
```

---

## 0.6 Crear AKS y asociarlo al ACR

> El parámetro `--attach-acr` permite que AKS tenga permisos `AcrPull` sobre el ACR. Esto deja el ambiente listo para futuros laboratorios con imágenes privadas.

```bash
az aks create \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_SIZE \
  --enable-managed-identity \
  --attach-acr $ACR_NAME \
  --generate-ssh-keys
```

Este paso puede tardar varios minutos.

---

## 0.7 Si falla por capacidad o SKU no disponible

Si ves un error como:

```text
SkuNotAvailable
```

o:

```text
The requested VM size is not available
```

cambia el tamaño de VM y vuelve a ejecutar `az aks create`.

Opciones comunes para laboratorio:

```bash
export NODE_SIZE="Standard_D2s_v5"
```

o:

```bash
export NODE_SIZE="Standard_DS3_v2"
```

Para consultar tamaños disponibles:

```bash
az vm list-skus \
  --location $LOCATION \
  --size Standard_D \
  --all \
  -o table
```

---

## 0.8 Conectarse al AKS

```bash
az aks get-credentials \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --overwrite-existing
```

Validar conexión:

```bash
kubectl get nodes -o wide
kubectl get ns
```

Resultado esperado:

```text
NAME                                STATUS   ROLES
aks-nodepool1-xxxxxx                Ready    agent
```

---

## 0.9 Crear namespace del laboratorio

```bash
kubectl create namespace $NS
kubectl config set-context --current --namespace=$NS
```

Validar:

```bash
kubectl get ns
kubectl config view --minify | grep namespace
```

---

## 0.10 Crear estructura local y guardar variables

```bash
cd ~/clouddrive

mkdir -p Ericsson_microservicios_labs
cd Ericsson_microservicios_labs

mkdir -p state-persistence-lab
cd state-persistence-lab

mkdir -p manifests scripts notes screenshots
```

Guardar variables para reutilizarlas:

```bash
cat <<EOF > scripts/env.sh
export PREFIX="$PREFIX"
export LOCATION="$LOCATION"
export RG_NAME="$RG_NAME"
export ACR_NAME="$ACR_NAME"
export AKS_NAME="$AKS_NAME"
export NS="$NS"
export NODE_COUNT="$NODE_COUNT"
export NODE_SIZE="$NODE_SIZE"
export ACR_LOGIN_SERVER="$ACR_LOGIN_SERVER"
EOF
```

Cargar variables en nuevas terminales:

```bash
source scripts/env.sh
```

Validar:

```bash
cat scripts/env.sh
```

---

## 0.11 Script rápido de reconstrucción para el instructor

> Si el instructor quiere reconstruir el ambiente rápidamente antes de clase, puede usar este script.

```bash
cat <<'EOF' > scripts/rebuild-aks-store-demo.sh
#!/bin/bash
set -e

export PREFIX="${PREFIX:-ericssonstate}"
export RANDOM_ID="${RANDOM_ID:-$RANDOM}"
export LOCATION="${LOCATION:-eastus2}"
export RG_NAME="${RG_NAME:-rg-${PREFIX}-${RANDOM_ID}}"
export ACR_NAME="${ACR_NAME:-${PREFIX}acr${RANDOM_ID}}"
export AKS_NAME="${AKS_NAME:-aks-${PREFIX}-${RANDOM_ID}}"
export NS="${NS:-aks-store-state-lab}"
export NODE_COUNT="${NODE_COUNT:-2}"
export NODE_SIZE="${NODE_SIZE:-Standard_D4s_v5}"

echo "Creating resource group: $RG_NAME"
az group create --name $RG_NAME --location $LOCATION

echo "Creating ACR: $ACR_NAME"
az acr create --resource-group $RG_NAME --name $ACR_NAME --sku Basic --admin-enabled false

echo "Creating AKS: $AKS_NAME"
az aks create \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_SIZE \
  --enable-managed-identity \
  --attach-acr $ACR_NAME \
  --generate-ssh-keys

echo "Getting credentials"
az aks get-credentials --resource-group $RG_NAME --name $AKS_NAME --overwrite-existing

echo "Creating namespace"
kubectl create namespace $NS || true
kubectl config set-context --current --namespace=$NS

export ACR_LOGIN_SERVER=$(az acr show --resource-group $RG_NAME --name $ACR_NAME --query loginServer -o tsv)

mkdir -p manifests scripts notes screenshots

cat <<ENV > scripts/env.sh
export PREFIX="$PREFIX"
export LOCATION="$LOCATION"
export RG_NAME="$RG_NAME"
export ACR_NAME="$ACR_NAME"
export AKS_NAME="$AKS_NAME"
export NS="$NS"
export NODE_COUNT="$NODE_COUNT"
export NODE_SIZE="$NODE_SIZE"
export ACR_LOGIN_SERVER="$ACR_LOGIN_SERVER"
ENV

echo "Done."
echo "Run: source scripts/env.sh"
EOF
```

Dar permisos:

```bash
chmod +x scripts/rebuild-aks-store-demo.sh
```

Ejecutar:

```bash
./scripts/rebuild-aks-store-demo.sh
```

---

# 1. Descargar y preparar AKS Store Demo

## 1.1 Descargar manifiesto oficial

> Usaremos el manifiesto `aks-store-all-in-one.yaml` del repositorio oficial Azure-Samples. Esta versión incluye RabbitMQ, MongoDB, servicios internos, frontends y componentes virtuales para generar actividad.

```bash
source scripts/env.sh

curl -L \
  https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml \
  -o manifests/aks-store-all-in-one.yaml
```

Validar:

```bash
head -20 manifests/aks-store-all-in-one.yaml
```

---

## 1.2 Decisión de exposición para el laboratorio

El manifiesto oficial expone `store-front` y `store-admin` como `LoadBalancer`.

Para este laboratorio **sí vamos a permitir LoadBalancer**, porque facilita que los alumnos accedan a las interfaces desde navegador sin depender de `port-forward`.

Validar que el manifiesto contiene servicios `LoadBalancer`:

```bash
grep -n "type: LoadBalancer" manifests/aks-store-all-in-one.yaml
```

Si por alguna razón quieres evitar IPs públicas, puedes cambiarlo a `ClusterIP`:

```bash
sed -i 's/type: LoadBalancer/type: ClusterIP/g' manifests/aks-store-all-in-one.yaml
```

Pero para la clase se recomienda mantener `LoadBalancer`.

---

## 1.3 Desplegar aplicación

```bash
kubectl apply -f manifests/aks-store-all-in-one.yaml -n $NS
```

Ver recursos:

```bash
kubectl get all -n $NS
```

Esperar a que todo esté listo:

```bash
kubectl wait --for=condition=Ready pod --all -n $NS --timeout=300s
```

---

## 1.4 Validar deployments y statefulsets

```bash
kubectl get deploy -n $NS
kubectl get statefulset -n $NS
kubectl get svc -n $NS
```

Resultado esperado:

```text
deployment.apps/order-service
deployment.apps/product-service
deployment.apps/makeline-service
deployment.apps/store-front
deployment.apps/store-admin
deployment.apps/virtual-customer
deployment.apps/virtual-worker

statefulset.apps/mongodb
statefulset.apps/rabbitmq
```

---

# 2. Explorar arquitectura de microservicios

## 2.1 Ver pods

```bash
kubectl get pods -n $NS -o wide
```

Pregunta para estudiantes:

> ¿Qué servicios son stateless y cuáles tienen estado?

Respuesta esperada:

| Recurso | Tipo |
|---|---|
| `store-front` | Stateless |
| `store-admin` | Stateless |
| `order-service` | Stateless |
| `product-service` | Stateless |
| `makeline-service` | Stateless |
| `virtual-customer` | Stateless |
| `virtual-worker` | Stateless |
| `mongodb` | Stateful |
| `rabbitmq` | Stateful |

---

## 2.2 Ver servicios internos

```bash
kubectl get svc -n $NS
```

Explicar:

- `ClusterIP` permite comunicación interna
- No todos los servicios deben estar expuestos públicamente
- Los frontends se pueden abrir temporalmente con `port-forward`

---

## 2.3 Acceder a store-front (SE PUEDE ACCEDER A TRAVÉS DE IP PÚBLICA)

En una terminal:

```bash
kubectl port-forward svc/store-front 8080:80 -n $NS
```

Abrir en navegador:

```text
http://localhost:8080
```

En Cloud Shell, usar **Web Preview** sobre el puerto `8080`.

---

## 2.4 Acceder a store-admin (SE PUEDE ACCEDER A TRAVÉS DE IP PÚBLICA)

En otra terminal:

```bash
kubectl port-forward svc/store-admin 8081:80 -n $NS
```

Abrir:

```text
http://localhost:8081
```

En Cloud Shell, usar **Web Preview** sobre el puerto `8081`.

---

## 2.5 Ver logs iniciales

```bash
kubectl logs deploy/order-service -n $NS --tail=50
kubectl logs deploy/makeline-service -n $NS --tail=50
kubectl logs deploy/product-service -n $NS --tail=50
```

Mensaje clave:

> En microservicios, los logs son parte de la arquitectura. Si no puedes seguir una operación, no puedes operar el sistema.

---

# 3. Event-driven con RabbitMQ

## 3.1 Acceder a RabbitMQ Management UI (ACCEDER A TRAVÉS DE LA IP PÚBLICA) 

```bash
kubectl patch svc rabbitmq -n aks-store-state-lab -p '{"spec":{"type":"LoadBalancer"}}'
```

Credenciales del manifiesto:

```text
username: username
password: password
```

---

## 3.2 Observar la cola `orders`

En RabbitMQ UI:

1. Ir a **Queues and Streams**
2. Buscar la cola `orders`
3. Observar:
   - Ready
   - Unacked
   - Total
   - Consumers

---

## 3.3 Generar actividad

La aplicación ya incluye `virtual-customer`, que genera órdenes automáticamente.

Ver logs:

```bash
kubectl logs deploy/virtual-customer -n $NS --tail=100
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
```

---

## 3.4 Explicación

Flujo conceptual:

```text
virtual-customer
      |
      v
order-service
      |
      v
RabbitMQ orders queue
      |
      v
makeline-service
      |
      v
mongodb
```

Mensaje clave:

> RabbitMQ desacopla la recepción de la orden del procesamiento final. Esto mejora disponibilidad, pero introduce consistencia eventual.

---

## 3.5 Pregunta de discusión

> Si `order-service` acepta la orden pero `makeline-service` aún no la procesa, ¿la orden existe o no existe?

Respuesta guiada:

- Existe como intención o mensaje en cola
- Puede no existir todavía como orden final procesada
- El sistema está en estado intermedio
- Esto es consistencia eventual

---

# 4. Fallo parcial: detener makeline-service

## 4.1 Escenario

Vamos a detener el servicio que procesa órdenes desde RabbitMQ.

Esto simula:

- Worker caído
- Consumidor indisponible
- Backlog de mensajes
- Eventual consistency visible

---

## 4.2 Escalar makeline-service a cero

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get deploy makeline-service -n $NS
```

---

## 4.3 Observar RabbitMQ

En RabbitMQ UI:

1. Ir a la cola `orders`
2. Observar cómo los mensajes pueden acumularse
3. Revisar número de consumers

También ver logs:

```bash
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/virtual-customer -n $NS --tail=100
```

---

## 4.4 Explicación

¿Qué está pasando?

```text
order-service sigue aceptando órdenes
RabbitMQ guarda mensajes
makeline-service no consume
MongoDB no recibe nuevas órdenes procesadas
```

Mensaje clave:

> El sistema sigue parcialmente disponible, pero el estado final se retrasa.

---

## 4.5 Discusión CAP

Pregunta:

> ¿Este comportamiento favorece disponibilidad o consistencia fuerte?

Respuesta:

- Favorece disponibilidad
- Acepta inconsistencia temporal
- Es un enfoque más cercano a AP - “AP” viene del teorema CAP y significa Availability + Partition Tolerance. En este enfoque, el sistema prioriza seguir funcionando y responder peticiones incluso si algunos componentes están caídos o desincronizados temporalmente. En el laboratorio, aunque un consumidor o microservicio falle, el resto del sistema continúa operando y los mensajes pueden procesarse después, aceptando consistencia eventual en lugar de detener toda la plataforma.
- No bloquea todo el sistema por caída de un consumidor

---

## 4.6 Restaurar makeline-service

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=makeline-service -n $NS --timeout=180s
```

Ver logs:

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
```

Observar RabbitMQ:

- Mensajes consumidos
- Cola reducida
- Flujo recuperado

---

## 4.7 Mensaje instructor

> Este es el valor de un broker: desacopla temporalmente productor y consumidor. Pero no elimina la complejidad: ahora hay que gestionar backlog, reintentos, duplicados e idempotencia.

---

# 5. Fallo parcial: detener RabbitMQ

## 5.1 Escenario

Ahora detenemos el broker.

Esto simula una falla más crítica:

```text
order-service quiere publicar evento
RabbitMQ no está disponible
mensaje no puede encolarse
```

---

## 5.2 Escalar RabbitMQ a cero

RabbitMQ está desplegado como StatefulSet.

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get statefulset rabbitmq -n $NS
```

---

## 5.3 Revisar comportamiento

Ver logs de `order-service`:

```bash
kubectl logs deploy/order-service -n $NS --tail=100
```

Ver eventos de Kubernetes:

```bash
kubectl get events -n $NS --sort-by=.lastTimestamp | tail -30
```

---

## 5.4 Pregunta para estudiantes

> Si el broker está caído, ¿qué debe hacer `order-service`?

Opciones:

### Opción A: Rechazar la orden

Ventaja:

- No se pierde intención de negocio
- Consistencia más fuerte

Desventaja:

- Menor disponibilidad
- Peor experiencia del cliente

### Opción B: Aceptar y guardar localmente

Ventaja:

- Mayor disponibilidad
- Mejor experiencia

Desventaja:

- Necesita Outbox Pattern
- Necesita reintentos
- Necesita reconciliación

---

## 5.5 Explicación: Outbox Pattern

Problema:

```text
Guardar orden en DB y publicar evento en RabbitMQ no es una transacción atómica.
```

Solución conceptual:

```text
1. Guardar orden en DB
2. Guardar evento en tabla outbox
3. Worker lee outbox
4. Publica a RabbitMQ
5. Marca evento como publicado
```

Diagrama:

```text
Order Service
   |
   +--> Orders DB
   |
   +--> Outbox Table
              |
              v
       Outbox Publisher
              |
              v
          RabbitMQ
```

Mensaje clave:

> El Outbox Pattern evita perder eventos cuando el broker está temporalmente caído.

---

## 5.6 Restaurar RabbitMQ

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $NS --timeout=180s
```

Validar:

```bash
kubectl get pods -n $NS
```

---

# 6. Fallo parcial: detener MongoDB

## 6.1 Escenario

MongoDB es el almacenamiento persistente del demo.

Detenerlo permite explicar:

- dependencia de persistencia
- degradación funcional
- diferencia entre stateless y stateful
- necesidad de resiliencia en capa de datos

---

## 6.2 Escalar MongoDB a cero

MongoDB también está como StatefulSet.

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get statefulset mongodb -n $NS
```

---

## 6.3 Probar aplicación

Intentar:

- abrir `store-front`
- abrir `store-admin`
- listar productos
- observar errores

Ver logs:

```bash
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/store-admin -n $NS --tail=100
```

---

## 6.4 Discusión

Pregunta:

> ¿Qué operaciones deberían seguir funcionando si MongoDB cae?

Posibles respuestas:

- Frontend puede cargar parcialmente
- Catálogo puede fallar si depende de DB
- Nuevas órdenes pueden fallar si procesamiento requiere persistencia
- Si hay cache, algunas lecturas podrían sobrevivir

---

## 6.5 Conexión con resiliencia

Conceptos:

- Circuit breaker
- Timeout
- Retry con backoff
- Fallback
- Cache como degradación
- Read models

Mensaje clave:

> Cuando la base de datos cae, la pregunta no es solo técnica. Es de negocio: ¿qué experiencia degradada aceptamos?

---

## 6.6 Restaurar MongoDB

```bash
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=mongodb -n $NS --timeout=180s
```

Validar:

```bash
kubectl get pods -n $NS
```

---

# 7. Redis Cache Lab

## 7.1 Objetivo

Agregar Redis para demostrar:

- Cache distribuido
- TTL
- Cache Aside
- Stale data
- Invalidación manual

---

## 7.2 Desplegar Redis

```bash
cat <<'EOF' > manifests/redis-lab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: aks-store-state-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 5m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: aks-store-state-lab
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
EOF
```

Si usaste otro namespace, reemplaza `aks-store-state-lab`:

```bash
sed -i "s/namespace: aks-store-state-lab/namespace: $NS/g" manifests/redis-lab.yaml
```

Aplicar:

```bash
kubectl apply -f manifests/redis-lab.yaml -n $NS
kubectl wait --for=condition=Ready pod -l app=redis -n $NS --timeout=120s
```

---

## 7.3 Entrar a Redis

```bash
REDIS_POD=$(kubectl get pod -n $NS -l app=redis -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $REDIS_POD -n $NS -- redis-cli
```

---

## 7.4 Crear producto en cache

Dentro de Redis:

```bash
SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":25.00,"stock":100}'
GET product:dog-food
TTL product:dog-food
```

Agregar TTL:

```bash
EXPIRE product:dog-food 300
TTL product:dog-food
```

Salir:

```bash
exit
```

---

## 7.5 Explicar Cache Aside

Patrón:

```text
1. Cliente pide producto
2. Servicio consulta Redis
3. Si existe: devuelve desde cache
4. Si no existe: consulta DB
5. Guarda resultado en Redis
6. Devuelve respuesta
```

Diagrama:

```text
Client
  |
  v
Product Service
  |
  +--> Redis
       |
       +-- HIT --> return cached product
       |
       +-- MISS --> MongoDB --> Redis --> return product
```

---

## 7.6 Simular stale data

Ver valor actual:

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli GET product:dog-food
```

Ahora simular que el precio real cambió a 20.00, pero Redis sigue con 25.00.

Discusión:

> ¿Qué pasa si el cliente compra con precio viejo? ¿El precio del catálogo puede ser eventualmente consistente? ¿El precio de checkout puede ser cacheado?

---

## 7.7 Invalidar cache

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli DEL product:dog-food
kubectl exec -it $REDIS_POD -n $NS -- redis-cli GET product:dog-food
```

Resultado esperado:

```text
(nil)
```

---

## 7.8 Recrear cache con valor actualizado

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":20.00,"stock":100}'
kubectl exec -it $REDIS_POD -n $NS -- redis-cli EXPIRE product:dog-food 300
kubectl exec -it $REDIS_POD -n $NS -- redis-cli GET product:dog-food
```

---

## 7.9 Discusión de estrategias

| Estrategia | Descripción | Ventaja | Riesgo |
|---|---|---|---|
| Cache Aside | App lee cache, si miss lee DB | Simple | Stale data |
| Write Through | Escribe cache y DB juntos | Cache actualizado | Escrituras lentas |
| Write Behind | Escribe cache primero, DB después | Rápido | Riesgo de pérdida |
| TTL Only | Expira por tiempo | Fácil | Ventana de inconsistencia |

---

# 8. CAP Theorem aplicado al laboratorio

## 8.1 Caso 1: makeline-service caído

```text
RabbitMQ disponible
order-service disponible
makeline-service caído
```

Decisión:

- Se aceptan órdenes
- Se acumulan mensajes
- Se procesa después

Interpretación:

- Mayor disponibilidad
- Consistencia eventual

---

## 8.2 Caso 2: RabbitMQ caído

```text
order-service disponible
RabbitMQ caído
```

Decisión posible:

- Rechazar órdenes
- O guardar intención localmente con Outbox

Interpretación:

- Si rechazo: más CP
- Si acepto con Outbox: más AP, pero más complejidad

---

## 8.3 Caso 3: MongoDB caído

```text
product-service / makeline-service dependen de MongoDB
```

Decisión posible:

- Fallar rápido
- Responder desde cache
- Activar modo degradado

Interpretación:

- Cache mejora disponibilidad
- Pero puede entregar datos stale

---

# 9. Saga conceptual sobre AKS Store Demo

AKS Store Demo no implementa una Saga completa de e-commerce con pagos e inventario separados, pero sí permite explicar el concepto usando su flujo real.

## 9.1 Flujo actual

```text
Order Created
   |
   v
Order Queued
   |
   v
Order Processed
```

## 9.2 Flujo Saga extendido conceptual

```text
1. Create Order
2. Reserve Inventory
3. Process Payment
4. Create Shipment
5. Confirm Order
```

## 9.3 Compensaciones

| Acción | Compensación |
|---|---|
| Crear orden | Cancelar orden |
| Reservar inventario | Liberar inventario |
| Procesar pago | Reembolsar pago |
| Crear envío | Cancelar envío |

## 9.4 Mensaje clave

> En microservicios, el rollback no siempre es técnico. Muchas veces es una nueva acción de negocio que compensa el efecto anterior.

---

# 10. Observabilidad básica

## 10.1 Ver logs por servicio

```bash
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/store-front -n $NS --tail=100
```

---

## 10.2 Seguir logs en vivo

```bash
kubectl logs deploy/order-service -n $NS -f
```

En otra terminal:

```bash
kubectl logs deploy/makeline-service -n $NS -f
```

---

## 10.3 Ver eventos de Kubernetes

```bash
kubectl get events -n $NS --sort-by=.lastTimestamp
```

---

## 10.4 Ver estado de pods

```bash
kubectl get pods -n $NS -o wide
kubectl describe pod -n $NS <POD_NAME>
```

---

## 10.5 Mensaje clave

> Sin observabilidad, un sistema distribuido es una caja negra. Ver pods corriendo no significa entender el estado de negocio.

---

# 11. Ejercicio para estudiantes

## Escenario

El negocio pide que la tienda siga aceptando órdenes aunque `makeline-service` esté caído durante 10 minutos.

## Preguntas

1. ¿Qué componente absorbe el fallo?
2. ¿Qué métrica deberíamos monitorear?
3. ¿Cuándo deberíamos dejar de aceptar órdenes?
4. ¿Qué pasa si RabbitMQ también cae?
5. ¿Necesitamos Outbox Pattern?
6. ¿Qué operación debe ser idempotente?
7. ¿Dónde habría que poner correlationId?

---

## Respuesta esperada

1. RabbitMQ absorbe temporalmente el fallo.
2. Longitud de la cola, tasa de consumo, edad del mensaje más viejo.
3. Cuando el backlog supera un umbral de negocio.
4. El sistema debe rechazar o usar Outbox.
5. Sí, si queremos aceptar órdenes aunque el broker esté caído.
6. Procesamiento de orden y publicación de eventos.
7. Desde la entrada en `store-front/order-service` hasta el evento en RabbitMQ y el procesamiento en `makeline-service`.

---

# 12. Checklist arquitectónico

Antes de diseñar persistencia distribuida, validar:

- ¿Quién es dueño del dato?
- ¿Existe shared database?
- ¿Hay comunicación síncrona innecesaria?
- ¿Qué pasa si el consumidor cae?
- ¿Qué pasa si el broker cae?
- ¿Qué pasa si la DB cae?
- ¿Hay idempotencia?
- ¿Hay compensación?
- ¿Hay timeout?
- ¿Hay retry con backoff?
- ¿Hay correlationId?
- ¿Qué inconsistencia tolera el negocio?
- ¿Qué datos se pueden cachear?
- ¿Cómo se invalida cache?
- ¿Qué métricas indican degradación?

---

# 13. Limpieza del laboratorio

## 13.1 Eliminar Redis

```bash
kubectl delete -f manifests/redis-lab.yaml -n $NS
```

## 13.2 Eliminar AKS Store Demo

```bash
kubectl delete -f aks-store-all-in-one.yaml -n $NS
```

## 13.3 Eliminar namespace

```bash
kubectl delete namespace $NS
```

Validar:

```bash
kubectl get ns
```

---

## 13.4 Eliminar recursos de Azure creados desde cero

> Ejecutar solo si quieres borrar completamente el ambiente del laboratorio.

```bash
az group delete   --name $RG_NAME   --yes   --no-wait
```

Validar:

```bash
az group exists --name $RG_NAME
```

Si devuelve `false`, el grupo ya no existe.

---

# 14. Cierre para estudiantes

En este laboratorio vimos una aplicación real en AKS y la usamos para estudiar problemas de arquitectura distribuida.

Conceptos clave:

- La persistencia ya no es solo “qué base de datos usar”.
- La pregunta real es quién es dueño del estado.
- RabbitMQ desacopla, pero introduce consistencia eventual.
- MongoDB centraliza persistencia en este demo, pero en un diseño más avanzado cada bounded context tendría su propia base.
- Redis mejora performance, pero puede devolver datos stale.
- Cuando un servicio cae, el sistema debe decidir entre consistencia y disponibilidad.
- Sagas no eliminan fallos: los hacen manejables con compensaciones.
- Sin observabilidad, no hay operación real de microservicios.

Mensaje final:

> En microservicios, diseñar persistencia significa diseñar comportamiento bajo fallo.

---

## Referencias oficiales

- Azure-Samples AKS Store Demo: https://github.com/Azure-Samples/aks-store-demo
- AKS Store Demo en Microsoft Learn: https://learn.microsoft.com/en-us/samples/azure-samples/aks-store-demo/aks-store-demo/
- Integrar AKS con Azure Container Registry: https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration
- Azure CLI `az aks`: https://learn.microsoft.com/en-us/cli/azure/aks
- AKS Labs: https://azure-samples.github.io/aks-labs/
