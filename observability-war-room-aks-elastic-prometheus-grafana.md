# Observability War Room en AKS
## Elastic Cloud + Azure Managed Prometheus + Azure Managed Grafana

> **Curso:** Microservicios Avanzados – Ericsson  
> **Duración:** 2 días × 6 horas = 12 horas  
> **Ambiente:** Azure Cloud Shell + AKS + AKS Store Demo + RabbitMQ + MongoDB + Redis + Istio  
> **Repositorio base:** https://github.com/nestorreveron/Ericsson_microservicios_labs

---

## 0. Propósito del laboratorio

Este laboratorio convierte el ambiente de microservicios que ya construimos en un escenario de **operación real bajo fallo**.

La idea no es instalar herramientas por instalar. La idea es que los estudiantes aprendan a responder preguntas operativas reales:

- ¿Qué servicio falló?
- ¿Dónde se está acumulando el backlog?
- ¿Qué pod reinició?
- ¿Qué componente responde lento?
- ¿Qué logs explican el error?
- ¿Qué evidencia tengo en métricas?
- ¿Qué evidencia tengo en logs?
- ¿Qué impacto tiene el fallo en el negocio?
- ¿Cómo diferencio una caída real de un bloqueo de seguridad esperado?
- ¿Cómo diagnostico un incidente distribuido?

---

## 1. Arquitectura objetivo

```text
AKS Store Demo en AKS
   |
   +--> store-front
   +--> store-admin
   +--> order-service
   +--> product-service
   +--> makeline-service
   +--> virtual-customer
   +--> virtual-worker
   +--> RabbitMQ
   +--> MongoDB
   +--> Redis
   +--> Istio / Envoy
   |
   +--> Elastic Agent
   |       |
   |       v
   |   Elastic Cloud / Kibana
   |
   +--> Azure Monitor Agent / Managed Prometheus
           |
           v
      Azure Monitor Workspace
           |
           v
      Azure Managed Grafana
```

---

## 2. Herramientas del laboratorio

| Herramienta | Uso principal en el laboratorio |
|---|---|
| Elastic Cloud | Plataforma SaaS para logs, métricas y observabilidad |
| Kibana | Búsqueda, exploración y dashboards de logs |
| Elastic Agent | Agente para recolectar logs/métricas desde Kubernetes |
| Fleet | Gestión centralizada de Elastic Agents |
| Azure Managed Prometheus | Métricas Prometheus administradas por Azure |
| Azure Managed Grafana | Dashboards, visualización y análisis |
| AKS | Plataforma Kubernetes donde corren los microservicios |
| RabbitMQ | Broker de mensajes |
| MongoDB | Persistencia del demo |
| Redis | Cache distribuido |
| Istio/Envoy | Service mesh, mTLS, JWT y seguridad Zero Trust |

---

## 3. Distribución de los 2 días

### Día 1 – Elastic Cloud + Logs + Incidentes con Kubernetes

| Bloque | Duración | Tema |
|---|---:|---|
| 1 | 30 min | Repaso de arquitectura y objetivos |
| 2 | 45 min | Preparación del repo y validación de AKS |
| 3 | 60 min | Crear trial de Elastic Cloud y configurar Kibana/Fleet |
| 4 | 75 min | Instalar Elastic Agent en AKS |
| 5 | 60 min | Buscar logs de microservicios en Kibana |
| 6 | 60 min | Incidente 1: MongoDB caído |
| 7 | 45 min | Incidente 2: makeline-service caído |
| 8 | 45 min | Discusión: logs, evidencia y runbook |

### Día 2 – Azure Managed Prometheus + Grafana + War Room

| Bloque | Duración | Tema |
|---|---:|---|
| 1 | 45 min | Habilitar Azure Managed Prometheus |
| 2 | 60 min | Crear/conectar Azure Managed Grafana |
| 3 | 60 min | Dashboards operacionales para AKS |
| 4 | 45 min | Métricas de RabbitMQ, MongoDB, Redis e Istio |
| 5 | 75 min | Incidentes guiados: RabbitMQ, MongoDB, Redis, Istio |
| 6 | 60 min | Reto final por equipos: diagnóstico en 15 minutos |
| 7 | 45 min | Diseño de alertas, SLO/SLI y buenas prácticas |
| 8 | 30 min | Limpieza y cierre |

---

# Día 1 – Elastic Cloud + Logs + Incidentes

---

## 4. Preparación inicial del ambiente

### 4.1 Abrir Azure Cloud Shell

Usar **Bash** en Azure Cloud Shell.

Validar sesión:

```bash
az account show -o table
```

---

### 4.2 Definir variables

Ajusta estos valores a tu ambiente real:

```bash
export RG_NAME="<TU_RESOURCE_GROUP>"
export AKS_NAME="<TU_AKS_NAME>"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"
```

Ejemplo:

```bash
export RG_NAME="rg-microservices-lab"
export AKS_NAME="aks-microservices-lab"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"
```

Validar:

```bash
echo $RG_NAME
echo $AKS_NAME
echo $NS
echo $LOCATION
```

---

### 4.3 Conectarse al AKS

```bash
az aks get-credentials   --resource-group $RG_NAME   --name $AKS_NAME   --overwrite-existing
```

Validar:

```bash
kubectl get nodes
kubectl get ns
```

---

## 5. Preparar repositorio para GitHub

### 5.1 Clonar o entrar al repo

```bash
cd ~/clouddrive
git clone https://github.com/nestorreveron/Ericsson_microservicios_labs.git
cd Ericsson_microservicios_labs
```

Si ya existe:

```bash
cd ~/clouddrive/Ericsson_microservicios_labs
git pull
```

---

### 5.2 Crear estructura del laboratorio

```bash
mkdir -p observability-war-room
cd observability-war-room

mkdir -p elastic
mkdir -p azure-prometheus-grafana
mkdir -p incidents
mkdir -p scripts
mkdir -p screenshots
mkdir -p notes
```

Estructura esperada:

```text
observability-war-room/
  elastic/
  azure-prometheus-grafana/
  incidents/
  scripts/
  screenshots/
  notes/
```

---

### 5.3 Crear archivo de variables

```bash
cat <<EOF > scripts/env.sh
export RG_NAME="$RG_NAME"
export AKS_NAME="$AKS_NAME"
export NS="$NS"
export LOCATION="$LOCATION"
EOF
```

Usarlo cuando abras una nueva terminal:

```bash
source scripts/env.sh
```

---

## 6. Validar AKS Store Demo

### 6.1 Ver pods

```bash
kubectl get pods -n $NS -o wide
```

### 6.2 Ver servicios

```bash
kubectl get svc -n $NS
```

### 6.3 Ver deployments y statefulsets

```bash
kubectl get deploy -n $NS
kubectl get statefulset -n $NS
```

### 6.4 Ver componentes esperados

```bash
kubectl get pods -n $NS | grep -E "store|order|product|makeline|rabbitmq|mongodb|redis"
```

---

## 7. Ver logs manualmente antes de Elastic

Antes de instalar Elastic, mostrar el problema del enfoque manual:

```bash
kubectl logs deploy/order-service -n $NS --tail=50
kubectl logs deploy/makeline-service -n $NS --tail=50
kubectl logs deploy/product-service -n $NS --tail=50
```

### Pregunta para estudiantes

> ¿Qué pasa si tengo 50 microservicios, 300 pods y un incidente distribuido?

Mensaje clave:

> `kubectl logs` sirve para diagnóstico básico, pero no escala como estrategia operacional.

---

# 8. Crear Elastic Cloud Trial

> Nota: Elastic Cloud ofrece actualmente trial gratuito de **14 días**. El objetivo aquí es usarlo como plataforma SaaS temporal para el cierre del curso.

### 8.1 Crear cuenta

1. Ir a: https://www.elastic.co/cloud/cloud-trial-overview
2. Crear cuenta o iniciar sesión.
3. Crear un deployment de Elastic Cloud.
4. Elegir región cercana o disponible.
5. Esperar a que el deployment quede listo.
6. Abrir **Kibana**.

---

## 9. Configurar Fleet y Elastic Agent

### 9.1 Abrir Fleet

En Kibana:

```text
Management
  → Fleet
```

Si Fleet solicita configuración inicial, aceptarla.

---

### 9.2 Crear Agent Policy

Crear una política llamada:

```text
aks-store-observability-policy
```

Sugerencia de descripción:

```text
Elastic Agent policy for AKS Store Demo observability lab.
```

---

### 9.3 Agregar integración Kubernetes

En Kibana:

```text
Integrations
  → Search: Kubernetes
  → Add Kubernetes
```

Configurar la integración para recolectar:

- Kubernetes container logs
- Kubernetes pod metrics
- Kubernetes node metrics
- Kubernetes events
- Kubernetes metadata

Guardar la integración dentro de la política:

```text
aks-store-observability-policy
```

---

### 9.4 Agregar Elastic Agent en Kubernetes

En Fleet:

```text
Agents
  → Add agent
  → Run Elastic Agent on Kubernetes
```

Seleccionar la política:

```text
aks-store-observability-policy
```

Kibana/Fleet generará un manifiesto o comandos con:

- Fleet URL
- Enrollment token
- Elastic Agent image
- Kubernetes DaemonSet
- RBAC necesario

Guardar el manifiesto generado como:

```bash
elastic/elastic-agent-managed-kubernetes.yaml
```

---

### 9.5 Aplicar manifiesto de Elastic Agent

Desde Cloud Shell:

```bash
kubectl apply -f elastic/elastic-agent-managed-kubernetes.yaml
```

Validar:

```bash
kubectl get pods -n kube-system | grep elastic
kubectl get daemonset -n kube-system | grep elastic
```

Comando alternativo:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=elastic-agent
```

---

### 9.6 Validar en Kibana

En Kibana:

```text
Management
  → Fleet
  → Agents
```

Resultado esperado:

```text
Elastic Agent: Healthy
```

---

## 10. Explorar logs en Kibana

### 10.1 Ir a Discover

En Kibana:

```text
Analytics
  → Discover
```

Buscar data views similares a:

```text
logs-*
metrics-*
```

---

### 10.2 Consultas útiles en Kibana

Buscar logs del namespace:

```text
kubernetes.namespace : "aks-store-state-lab"
```

Buscar logs de makeline-service:

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "makeline-service"
```

Buscar logs de MongoDB:

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "mongodb"
```

Buscar errores:

```text
kubernetes.namespace : "aks-store-state-lab" and message : "*error*"
```

Buscar connection refused:

```text
message : "*connection refused*"
```

Buscar timeouts:

```text
message : "*timeout*" or message : "*deadline exceeded*"
```

---

## 11. Incidente 1 – MongoDB caído

### 11.1 Escenario

MongoDB es la persistencia principal del demo. Si MongoDB cae:

- product-service puede fallar al leer productos
- makeline-service puede fallar al guardar órdenes
- store-admin puede fallar al listar o administrar datos
- algunos pods pueden seguir Running aunque el negocio esté degradado

---

### 11.2 Provocar fallo

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get statefulset mongodb -n $NS
```

---

### 11.3 Observar logs con kubectl

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/store-admin -n $NS --tail=100
```

Posibles errores esperados:

```text
connection refused
server selection timeout
Failed to insert order
Failed to save orders to database
context deadline exceeded
```

---

### 11.4 Buscar en Kibana

Usar consultas:

```text
message : "*connection refused*" and kubernetes.namespace : "aks-store-state-lab"
```

```text
message : "*Failed to insert order*"
```

```text
message : "*mongodb*"
```

---

### 11.5 Discusión

Preguntas para alumnos:

1. ¿La aplicación está completamente caída?
2. ¿Qué pods siguen Running?
3. ¿Qué operaciones fallan?
4. ¿Qué operaciones podrían seguir funcionando?
5. ¿Qué evidencia aparece en logs?
6. ¿Qué métrica sería útil para alertar?
7. ¿Qué patrón ayudaría a degradar mejor? Cache, circuit breaker, fallback, retry?

Mensaje clave:

> En microservicios, “pod Running” no significa “negocio saludable”.

---

### 11.6 Restaurar MongoDB

```bash
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=mongodb -n $NS --timeout=180s
kubectl get pods -n $NS
```

Si MongoDB queda en CrashLoopBackOff:

```bash
kubectl logs mongodb-0 -n $NS --previous
kubectl describe pod mongodb-0 -n $NS
kubectl delete pod mongodb-0 -n $NS
```

---

## 12. Incidente 2 – makeline-service caído

### 12.1 Escenario

`makeline-service` consume órdenes desde RabbitMQ y las procesa. Al apagarlo:

- order-service puede seguir generando mensajes
- RabbitMQ acumula mensajes
- el sistema sigue parcialmente disponible
- el procesamiento se retrasa

---

### 12.2 Provocar fallo

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get deploy makeline-service -n $NS
```

---

### 12.3 Observar RabbitMQ UI

Si RabbitMQ está como LoadBalancer:

```bash
kubectl get svc rabbitmq -n $NS
```

Si no está expuesto:

```bash
kubectl port-forward svc/rabbitmq 15672:15672 -n $NS
```

Abrir:

```text
http://localhost:15672
```

Credenciales típicas del lab:

```text
username: username
password: password
```

Ver:

```text
Queues and Streams → orders
```

Observar:

- Ready
- Unacked
- Total
- Consumers

---

### 12.4 Buscar logs en Kibana

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "order-service"
```

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "virtual-customer"
```

---

### 12.5 Discusión CAP

Pregunta:

> ¿Esto favorece disponibilidad o consistencia fuerte?

Respuesta esperada:

- Favorece disponibilidad.
- Acepta inconsistencia temporal.
- Es un comportamiento más cercano a AP.
- El sistema no bloquea toda la plataforma porque un consumer esté caído.

Mensaje clave:

> RabbitMQ permite desacoplamiento temporal, pero introduce backlog, latencia eventual e idempotencia como preocupaciones reales.

---

### 12.6 Restaurar makeline-service

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=makeline-service -n $NS --timeout=180s
```

Ver logs:

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
```

---

## 13. Cierre del Día 1

### 13.1 Evidencias que deben guardar los alumnos

Crear archivo:

```bash
cat <<EOF > notes/day1-evidence.md
# Day 1 Evidence – Elastic + Kubernetes Logs

## MongoDB Down
- Symptoms:
- Kibana query used:
- Log evidence:
- Business impact:
- Recovery action:

## makeline-service Down
- Symptoms:
- RabbitMQ evidence:
- Kibana query used:
- Business impact:
- Recovery action:

## Lessons Learned
-
-
-
EOF
```

---

# Día 2 – Azure Managed Prometheus + Azure Managed Grafana

---

## 14. Objetivo del Día 2

El Día 1 vimos logs centralizados con Elastic.

El Día 2 vamos a complementar con:

- métricas
- dashboards
- visualización operacional
- alertas conceptuales
- diagnóstico por síntomas
- war room final por equipos

Mensaje clave:

> Logs explican qué ocurrió. Métricas muestran el estado y la tendencia del sistema.

---

## 15. Habilitar Azure Managed Prometheus

### 15.1 Registrar proveedores si es necesario

```bash
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.Dashboard
az provider register --namespace Microsoft.Insights
```

Validar:

```bash
az provider show --namespace Microsoft.Monitor --query registrationState -o tsv
az provider show --namespace Microsoft.Dashboard --query registrationState -o tsv
az provider show --namespace Microsoft.Insights --query registrationState -o tsv
```

---

### 15.2 Crear Azure Monitor Workspace

```bash
export AMW_NAME="amw-ericsson-observability"
```

```bash
az monitor account create   --name $AMW_NAME   --resource-group $RG_NAME   --location $LOCATION
```

Obtener ID:

```bash
export AMW_ID=$(az monitor account show   --name $AMW_NAME   --resource-group $RG_NAME   --query id -o tsv)

echo $AMW_ID
```

---

### 15.3 Crear Azure Managed Grafana

El nombre debe ser único globalmente dentro de Azure.

```bash
export GRAFANA_NAME="grafana-ericsson-$RANDOM"
```

```bash
az grafana create   --name $GRAFANA_NAME   --resource-group $RG_NAME   --location $LOCATION
```

Obtener ID:

```bash
export GRAFANA_ID=$(az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query id -o tsv)

echo $GRAFANA_ID
```

Obtener endpoint:

```bash
az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query properties.endpoint -o tsv
```

---

### 15.4 Asignar permisos de acceso a Grafana

Obtener usuario actual:

```bash
export USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
echo $USER_OBJECT_ID
```

Asignar rol Grafana Admin:

```bash
az role assignment create   --assignee $USER_OBJECT_ID   --role "Grafana Admin"   --scope $GRAFANA_ID
```

Si este comando falla por permisos del tenant, hacerlo desde Azure Portal:

```text
Azure Managed Grafana
  → Access control (IAM)
  → Add role assignment
  → Grafana Admin
  → Select user
```

---

### 15.5 Habilitar Prometheus en AKS y vincular Grafana

```bash
az aks update   --name $AKS_NAME   --resource-group $RG_NAME   --enable-azure-monitor-metrics   --azure-monitor-workspace-resource-id $AMW_ID   --grafana-resource-id $GRAFANA_ID
```

Validar:

```bash
kubectl get pods -n kube-system | grep -E "ama-metrics|azuremonitor|prometheus"
```

También revisar:

```bash
kubectl get pods -n kube-system
```

---

## 16. Explorar métricas en Azure Managed Grafana

### 16.1 Abrir Grafana

```bash
az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query properties.endpoint -o tsv
```

Abrir el endpoint en navegador.

---

### 16.2 Confirmar data source

En Grafana:

```text
Connections
  → Data sources
```

Buscar un data source similar a:

```text
Azure Monitor managed service for Prometheus
```

o:

```text
Prometheus_<Azure Monitor workspace endpoint>
```

---

### 16.3 Revisar dashboards predefinidos

Buscar dashboards relacionados con:

- Kubernetes
- AKS
- Node
- Pod
- Namespace
- Workload

Crear o duplicar dashboard para el laboratorio.

---

## 17. PromQL básico para clase

> Las métricas exactas pueden variar según configuración. Usar estas consultas como guía y ajustarlas en Grafana/Prometheus Explorer.

### 17.1 Pods por namespace

```promql
sum by (namespace) (kube_pod_status_phase)
```

### 17.2 Pods del namespace del laboratorio

```promql
sum by (pod, phase) (kube_pod_status_phase{namespace="aks-store-state-lab"})
```

### 17.3 Restarts por pod

```promql
sum by (pod) (kube_pod_container_status_restarts_total{namespace="aks-store-state-lab"})
```

### 17.4 Pods no disponibles

```promql
sum by (deployment) (kube_deployment_status_replicas_unavailable{namespace="aks-store-state-lab"})
```

### 17.5 CPU por container

```promql
sum by (pod, container) (
  rate(container_cpu_usage_seconds_total{namespace="aks-store-state-lab"}[5m])
)
```

### 17.6 Memoria por container

```promql
sum by (pod, container) (
  container_memory_working_set_bytes{namespace="aks-store-state-lab"}
)
```

### 17.7 Pods reiniciando

```promql
increase(kube_pod_container_status_restarts_total{namespace="aks-store-state-lab"}[15m])
```

---

## 18. Crear dashboard mínimo en Grafana

Crear un dashboard llamado:

```text
AKS Store Demo – Observability War Room
```

Paneles mínimos:

| Panel | Consulta o fuente |
|---|---|
| Pods por estado | `kube_pod_status_phase` |
| Restarts por pod | `kube_pod_container_status_restarts_total` |
| CPU por pod | `container_cpu_usage_seconds_total` |
| Memoria por pod | `container_memory_working_set_bytes` |
| Deployments no disponibles | `kube_deployment_status_replicas_unavailable` |
| RabbitMQ backlog | RabbitMQ UI o métricas Prometheus si se habilita plugin |
| Incidentes recientes | Kibana/Elastic |
| Seguridad 401/403 | Logs Istio/Envoy si están recolectados |

---

## 19. Opcional avanzado – Exponer métricas Prometheus de RabbitMQ

> Esta sección depende de la imagen y configuración de RabbitMQ. El plugin Prometheus de RabbitMQ expone métricas normalmente por el puerto `15692` en el endpoint `/metrics`.

### 19.1 Entrar al pod RabbitMQ

```bash
RABBIT_POD=$(kubectl get pod -n $NS -l app=rabbitmq -o jsonpath='{.items[0].metadata.name}')
echo $RABBIT_POD
```

```bash
kubectl exec -it $RABBIT_POD -n $NS -- bash
```

Dentro del pod:

```bash
rabbitmq-plugins enable rabbitmq_prometheus
rabbitmq-diagnostics -s listeners
exit
```

---

### 19.2 Crear servicio para métricas RabbitMQ

```bash
cat <<EOF > azure-prometheus-grafana/rabbitmq-prometheus-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-prometheus
  namespace: $NS
  labels:
    app: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
  - name: prometheus
    port: 15692
    targetPort: 15692
EOF
```

Aplicar:

```bash
kubectl apply -f azure-prometheus-grafana/rabbitmq-prometheus-svc.yaml
```

Validar localmente:

```bash
kubectl port-forward svc/rabbitmq-prometheus 15692:15692 -n $NS
```

En otra terminal:

```bash
curl http://localhost:15692/metrics | head
```

---

### 19.3 Discusión

Si se ven métricas:

- RabbitMQ está exponiendo métricas Prometheus.
- Podemos integrar estas métricas en un scrape custom.
- Podemos crear alertas por queue depth, unacked messages y consumers.

Si no se ven métricas:

- Continuar usando RabbitMQ UI para backlog.
- Usar Elastic/Kibana para logs.
- Usar Prometheus/Grafana para salud general de Kubernetes.

---

## 20. Incidente 3 – RabbitMQ caído

### 20.1 Provocar fallo

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
kubectl get statefulset rabbitmq -n $NS
```

---

### 20.2 Observar en Grafana

Buscar:

- pods no disponibles
- restarts
- cambios en deployments
- errores indirectos en workloads

---

### 20.3 Observar en Kibana

Buscar:

```text
rabbitmq
```

```text
connection refused
```

```text
failed
```

```text
order-service
```

---

### 20.4 Preguntas para alumnos

1. ¿order-service debería rechazar órdenes?
2. ¿Debería guardarlas localmente?
3. ¿Existe Outbox Pattern en esta app?
4. ¿Qué evidencia tengo de que RabbitMQ está caído?
5. ¿Qué métrica alertaría antes que el usuario reporte el problema?

---

### 20.5 Restaurar RabbitMQ

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $NS --timeout=180s
```

---

## 21. Incidente 4 – Redis stale data

### 21.1 Validar Redis

```bash
REDIS_POD=$(kubectl get pod -n $NS -l app=redis -o jsonpath='{.items[0].metadata.name}')
echo $REDIS_POD
```

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli GET product:dog-food
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

---

### 21.2 Crear stale data

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":25.00,"stock":100}'
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

Resultado típico:

```text
-1
```

Significa:

```text
La key no expira automáticamente.
```

---

### 21.3 Agregar TTL

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli EXPIRE product:dog-food 300
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

---

### 21.4 Preguntas para alumnos

1. ¿El sistema está fallando técnicamente?
2. ¿Puede estar fallando para negocio?
3. ¿Prometheus detecta stale data automáticamente?
4. ¿Qué métrica de negocio necesitaríamos?
5. ¿El checkout debería confiar en cache?

Mensaje clave:

> No todo problema observable es técnico. Algunos problemas son inconsistencias de negocio.

---

## 22. Incidente 5 – Istio/JWT 403 esperado

### 22.1 Probar sin token

```bash
GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

```bash
curl -s -o /dev/null -w "%{http_code}
" http://$GATEWAY_IP
```

Resultado esperado:

```text
403
```

---

### 22.2 Preguntas para alumnos

1. ¿La app está caída?
2. ¿O la política de seguridad está funcionando?
3. ¿Qué logs buscaríamos?
4. ¿Cómo diferenciar un 403 esperado de un incidente?
5. ¿Qué dashboard debería mostrar tráfico denegado?

Mensaje clave:

> Un 403 no siempre significa error. Puede ser evidencia de que Zero Trust está funcionando.

---

## 23. Reto final – Observability War Room

### 23.1 Dinámica

Dividir la clase en equipos.

El instructor provoca un fallo sin decir cuál.

Opciones:

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
```

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
```

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
```

```bash
kubectl delete pod -n $NS -l app=product-service
```

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":999.00,"stock":100}'
```

---

### 23.2 Cada equipo debe entregar diagnóstico

Crear archivo:

```bash
cat <<EOF > notes/final-war-room-diagnosis.md
# Final War Room Diagnosis

## Team Name
-

## Incident Detected
-

## Symptoms
-

## Evidence from Kubernetes
-

## Evidence from Elastic/Kibana
-

## Evidence from Grafana/Prometheus
-

## Business Impact
-

## CAP / Resilience Interpretation
-

## Recovery Action
-

## Preventive Alert
-

## Recommended Architecture Improvement
-
EOF
```

---

### 23.3 Preguntas guía

1. ¿Qué componente falló?
2. ¿El sistema está totalmente caído o degradado?
3. ¿Qué métrica cambió?
4. ¿Qué logs explican el problema?
5. ¿Qué comando Kubernetes confirma el diagnóstico?
6. ¿Cuál es el impacto de negocio?
7. ¿Qué alerta debería existir?
8. ¿Qué patrón arquitectónico ayudaría?
9. ¿Qué runbook deberíamos documentar?
10. ¿Cómo evitamos que el incidente se repita?

---

## 24. Runbook mínimo de recuperación

Crear archivo:

```bash
cat <<EOF > notes/recovery-runbook.md
# Recovery Runbook – AKS Store Demo Observability Lab

## makeline-service down

Detect:
- RabbitMQ queue depth increasing
- No makeline-service pods
- No processing logs

Confirm:
kubectl get deploy makeline-service -n \$NS
kubectl get pods -n \$NS

Recover:
kubectl scale deployment makeline-service --replicas=1 -n \$NS

Validate:
kubectl logs deploy/makeline-service -n \$NS --tail=100

---

## RabbitMQ down

Detect:
- RabbitMQ pod missing
- order-service publish errors
- RabbitMQ UI unavailable

Confirm:
kubectl get statefulset rabbitmq -n \$NS
kubectl get pods -n \$NS

Recover:
kubectl scale statefulset rabbitmq --replicas=1 -n \$NS

Validate:
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n \$NS --timeout=180s

---

## MongoDB down

Detect:
- connection refused
- server selection timeout
- product/order persistence errors

Confirm:
kubectl get statefulset mongodb -n \$NS
kubectl logs deploy/makeline-service -n \$NS --tail=100

Recover:
kubectl scale statefulset mongodb --replicas=1 -n \$NS

Validate:
kubectl wait --for=condition=Ready pod -l app=mongodb -n \$NS --timeout=180s

---

## Redis stale data

Detect:
- Fast responses but incorrect business value
- TTL missing or too long

Confirm:
kubectl exec -it \$REDIS_POD -n \$NS -- redis-cli GET product:dog-food
kubectl exec -it \$REDIS_POD -n \$NS -- redis-cli TTL product:dog-food

Recover:
kubectl exec -it \$REDIS_POD -n \$NS -- redis-cli DEL product:dog-food

Validate:
kubectl exec -it \$REDIS_POD -n \$NS -- redis-cli GET product:dog-food
EOF
```

---

## 25. Buenas prácticas que debe llevarse el alumno

### 25.1 Métricas

Usar métricas para responder:

```text
¿Qué está pasando?
¿Cuándo empezó?
¿Cuánto impacto tiene?
¿Está mejorando o empeorando?
```

### 25.2 Logs

Usar logs para responder:

```text
¿Qué ocurrió exactamente?
¿Qué error devolvió la aplicación?
¿Qué request o evento falló?
```

### 25.3 Traces

Usar traces para responder:

```text
¿Dónde se demoró la request?
¿Qué dependencia fue lenta?
¿Qué servicio rompió el flujo?
```

### 25.4 Dashboards

Un dashboard debe ser accionable.

Mal dashboard:

```text
Muchos gráficos bonitos sin decisión clara.
```

Buen dashboard:

```text
Muestra si el negocio está sano, degradado o caído.
```

### 25.5 Alertas

Una alerta debe tener:

- síntoma
- impacto
- severidad
- responsable
- acción recomendada

---

## 26. Limpieza del laboratorio

### 26.1 Restaurar componentes

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
```

Validar:

```bash
kubectl get pods -n $NS
```

---

### 26.2 Eliminar Elastic Agent

Si quieres eliminar Elastic Agent del cluster:

```bash
kubectl delete -f elastic/elastic-agent-managed-kubernetes.yaml
```

Validar:

```bash
kubectl get pods -n kube-system | grep elastic
```

---

### 26.3 Eliminar recursos de Azure Monitor/Grafana

> Solo si ya no se necesitan.

```bash
az grafana delete   --name $GRAFANA_NAME   --resource-group $RG_NAME   --yes
```

```bash
az resource delete --ids $AMW_ID
```

---

## 27. Cierre del laboratorio

Mensaje final:

> En microservicios, no basta con diseñar sistemas resilientes. Debemos poder observar, medir, diagnosticar y explicar cómo se comportan cuando fallan.

Conceptos finales:

- `kubectl get pods` no es suficiente.
- Logs centralizados permiten investigar incidentes distribuidos.
- Prometheus muestra la salud y tendencia del sistema.
- Grafana convierte métricas en decisiones operativas.
- Elastic/Kibana permite buscar evidencia técnica.
- RabbitMQ desacopla, pero el backlog debe observarse.
- MongoDB caído no siempre tumba todos los pods, pero sí degrada el negocio.
- Redis mejora performance, pero puede entregar datos stale.
- Istio puede devolver 403 como comportamiento correcto.
- Observabilidad real combina métricas, logs, traces y contexto de negocio.

Frase final para estudiantes:

> Construir microservicios es solo la mitad del trabajo. La otra mitad es poder entenderlos cuando algo falla.

---

## 28. Referencias oficiales

- Elastic Cloud Trial: https://www.elastic.co/cloud/cloud-trial-overview
- Evaluate Elastic during a trial: https://www.elastic.co/docs/get-started/evaluate-elastic
- Run Elastic Agent on Azure AKS managed by Fleet: https://www.elastic.co/docs/reference/fleet/running-on-aks-managed-by-fleet
- Run Elastic Agent on Kubernetes managed by Fleet: https://www.elastic.co/docs/reference/fleet/running-on-kubernetes-managed-by-fleet
- Azure Monitor managed service for Prometheus: https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-metrics-overview
- Enable monitoring for AKS: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable
- Monitor AKS: https://learn.microsoft.com/en-us/azure/aks/monitor-aks
- Connect Grafana to Azure Monitor managed Prometheus: https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-grafana
- RabbitMQ Prometheus metrics: https://www.rabbitmq.com/docs/prometheus
