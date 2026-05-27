# Observability War Room en AKS — Módulo 3
## Distributed Tracing con OpenTelemetry + Jaeger

> **Curso:** Microservicios Avanzados – Ericsson  
> **Módulo:** Observabilidad Distribuida y SRE en AKS  
> **Ambiente:** Azure Cloud Shell + AKS + AKS Store Demo + RabbitMQ + MongoDB + Redis + Istio/Envoy  
> **Stack anterior:** Elastic Cloud + Azure Managed Prometheus + Azure Managed Grafana  
> **Nuevo stack:** OpenTelemetry Collector + Jaeger + integración opcional con Istio, Prometheus y Grafana  
> **Objetivo:** Agregar trazabilidad distribuida real al laboratorio de observabilidad existente.

---

## 0. Propósito del módulo

Este módulo extiende el laboratorio anterior de observabilidad en AKS. El laboratorio anterior ya cubría:

- Logs centralizados con Elastic Cloud / Kibana.
- Métricas con Azure Managed Prometheus.
- Dashboards operacionales con Azure Managed Grafana.
- Diagnóstico de incidentes con RabbitMQ, MongoDB, Redis e Istio.

Este nuevo módulo agrega la tercera dimensión crítica de la observabilidad moderna:

> **Distributed Tracing**.

La idea no es instalar Jaeger por instalarlo. La idea es construir una arquitectura de trazabilidad distribuida donde **OpenTelemetry actúa como capa central de instrumentación** y **Jaeger funciona como backend principal para visualizar traces**.

El objetivo operativo es que los alumnos puedan responder preguntas como:

- ¿Dónde se demoró una request?
- ¿Qué servicio introdujo latencia?
- ¿Qué dependencia falló?
- ¿Dónde se cortó el flujo distribuido?
- ¿Qué spans aparecen y cuáles faltan?
- ¿Qué relación existe entre logs, métricas y traces?
- ¿Qué diferencia hay entre un error técnico, una degradación y un fallo de negocio?
- ¿Cómo cambia el diagnóstico cuando el sistema es síncrono, asíncrono o event-driven?

---

## 1. Objetivo arquitectónico

El objetivo arquitectónico es agregar una capa de trazabilidad distribuida a una arquitectura de microservicios sobre AKS.

La arquitectura objetivo queda así:

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
   |       |
   |       v
   |   Azure Monitor Workspace
   |       |
   |       v
   |   Azure Managed Grafana
   |
   +--> OpenTelemetry Collector
           |
           +--> Traces OTLP
           |       |
           |       v
           |   Jaeger
           |
           +--> Metrics opcionales
                   |
                   v
               Prometheus / Grafana
```

---

## 2. Alcance real del laboratorio

Este laboratorio cubre tres niveles de trazabilidad.

### Nivel 1 — Trazabilidad sin tocar código usando Istio/Envoy

Si los microservicios ya están dentro de Istio, Envoy puede generar spans de tráfico HTTP entre servicios.

Esto permite observar:

- llamadas HTTP entre servicios;
- latencia entre workloads;
- errores 4xx/5xx;
- rutas de tráfico dentro del mesh;
- impacto de timeouts, delays y retries configurados en Istio.

Este enfoque es ideal para explicar **Service Mesh Observability**.

### Nivel 2 — Trazabilidad centralizada con OpenTelemetry Collector

OpenTelemetry Collector será el componente central que recibe telemetry por OTLP y la exporta a Jaeger.

Esto permite desacoplar:

- aplicaciones;
- service mesh;
- backends de observabilidad;
- pipelines de traces, métricas y logs.

Este enfoque es el recomendado en arquitecturas enterprise porque evita que cada microservicio dependa directamente de Jaeger, Elastic o Prometheus.

### Nivel 3 — Instrumentación de aplicación

Para observar operaciones internas como:

- publish a RabbitMQ;
- consumo de mensajes;
- query a MongoDB;
- consulta a Redis;
- operaciones de negocio;

se requiere instrumentación de aplicación con OpenTelemetry SDK o auto-instrumentation.

Este laboratorio deja claro este punto:

> **Istio puede observar tráfico de red, pero no siempre puede ver operaciones internas de negocio o llamadas a librerías como RabbitMQ, MongoDB o Redis. Para eso se requiere instrumentación de aplicación.**

---

## 3. Relación entre OpenTelemetry, Jaeger, Elastic y Prometheus

| Componente | Rol en la arquitectura | Pregunta que ayuda a responder |
|---|---|---|
| OpenTelemetry | Estándar de instrumentación y pipeline de telemetry | ¿Cómo capturo y envío traces, métricas y logs de forma portable? |
| OpenTelemetry Collector | Gateway central de telemetry | ¿Cómo desacoplo aplicaciones de los backends de observabilidad? |
| Jaeger | Backend y UI de distributed tracing | ¿Dónde se demoró o falló una request distribuida? |
| Prometheus | Métricas time-series | ¿Qué está pasando y cuál es la tendencia? |
| Grafana | Visualización y dashboards | ¿El sistema está sano, degradado o caído? |
| Elastic/Kibana | Logs centralizados y búsqueda | ¿Qué ocurrió exactamente y qué evidencia textual existe? |
| Istio/Envoy | Service mesh y telemetry de red | ¿Cómo se comporta el tráfico east-west entre servicios? |

Mensaje clave para estudiantes:

> **Prometheus muestra síntomas, Elastic explica eventos, Jaeger muestra el camino de una request y OpenTelemetry conecta todos esos mundos.**

---

## 4. Patrones y anti-patrones del módulo

### Patrones que se aplican

| Patrón | Uso en el laboratorio |
|---|---|
| Distributed Tracing | Reconstruir requests end-to-end |
| Context Propagation | Mantener TraceId entre servicios |
| Sidecar Pattern | Envoy genera telemetry sin modificar app |
| Telemetry Gateway | OpenTelemetry Collector centraliza telemetry |
| Service Mesh Observability | Istio observa tráfico east-west |
| Event-Driven Observability | Diagnóstico de flujos RabbitMQ |
| Backpressure Detection | Identificar backlog y consumers lentos |
| Circuit Breaker | Evitar cascadas ante dependencias caídas |
| Retry with Backoff | Evitar retry storms |
| Root Cause Analysis | Correlacionar spans, logs y métricas |
| SLO-driven Troubleshooting | Diagnosticar desde impacto real |

### Anti-patrones que se discuten

| Anti-patrón | Riesgo |
|---|---|
| Enviar traces directamente desde cada app a Jaeger | Acoplamiento fuerte al backend |
| No propagar TraceId | Trazabilidad incompleta |
| Solo usar logs | Troubleshooting lento en sistemas distribuidos |
| Solo usar métricas | Se detecta el síntoma, pero no el camino del fallo |
| Tracing parcial | Falsa sensación de observabilidad |
| Sampling mal configurado | Pérdida de evidencia durante incidentes |
| No instrumentar colas | Se pierde visibilidad async |
| Retries sin control | Cascading failures |
| Exponer Jaeger públicamente sin restricciones | Riesgo de seguridad |

---

## 5. Prerrequisitos

### 5.1 Herramientas necesarias

En Azure Cloud Shell normalmente ya están disponibles:

```bash
az version
kubectl version --client
helm version
```

Si usas una máquina local, debes tener:

- Azure CLI
- kubectl
- Helm
- istioctl, si vas a modificar configuración de Istio
- acceso al AKS
- permisos para crear Services tipo LoadBalancer

---

## 6. Variables de ambiente

Ajusta estos valores según tu entorno.

```bash
export RG_NAME="<TU_RESOURCE_GROUP>"
export AKS_NAME="<TU_AKS_NAME>"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"

export TRACE_NS="observability-tracing"
export OTEL_COLLECTOR_NAME="otel-collector"
export JAEGER_NAME="jaeger"
```

Ejemplo:

```bash
export RG_NAME="rg-microservices-lab"
export AKS_NAME="aks-microservices-lab"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"

export TRACE_NS="observability-tracing"
export OTEL_COLLECTOR_NAME="otel-collector"
export JAEGER_NAME="jaeger"
```

Conectarse al AKS:

```bash
az aks get-credentials \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --overwrite-existing
```

Validar:

```bash
kubectl get nodes
kubectl get ns
kubectl get pods -n $NS
```

---

## 7. Preparar estructura del repositorio

Entrar al repositorio del curso:

```bash
cd ~/clouddrive
cd Ericsson_microservicios_labs
```

Crear carpeta para el nuevo módulo:

```bash
mkdir -p observability-war-room/tracing-jaeger-otel
cd observability-war-room/tracing-jaeger-otel

mkdir -p manifests
mkdir -p istio
mkdir -p incidents
mkdir -p notes
mkdir -p screenshots
mkdir -p scripts
```

Crear archivo de variables:

```bash
cat <<EOF > scripts/env.sh
export RG_NAME="$RG_NAME"
export AKS_NAME="$AKS_NAME"
export NS="$NS"
export LOCATION="$LOCATION"
export TRACE_NS="$TRACE_NS"
export OTEL_COLLECTOR_NAME="$OTEL_COLLECTOR_NAME"
export JAEGER_NAME="$JAEGER_NAME"
EOF
```

Cargarlo cuando abras una nueva terminal:

```bash
source scripts/env.sh
```

---

# Parte 1 — Instalar Jaeger en AKS

## 8. Crear namespace de tracing

```bash
kubectl create namespace $TRACE_NS --dry-run=client -o yaml | kubectl apply -f -
kubectl get ns $TRACE_NS
```

---

## 9. Instalar Jaeger All-in-One para laboratorio

Para este laboratorio usaremos Jaeger All-in-One porque es simple y suficiente para training.

> **Importante:** Jaeger All-in-One usa almacenamiento en memoria. Es adecuado para laboratorios, demos y troubleshooting temporal. No es un diseño de producción.

Crear manifiesto:

```bash
cat <<EOF > manifests/jaeger-all-in-one.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability-tracing
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        ports:
        - name: query
          containerPort: 16686
        - name: otlp-grpc
          containerPort: 4317
        - name: otlp-http
          containerPort: 4318
        - name: jaeger-grpc
          containerPort: 14250
        - name: jaeger-http
          containerPort: 14268
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: observability-tracing
  labels:
    app: jaeger
spec:
  type: ClusterIP
  selector:
    app: jaeger
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
  - name: jaeger-grpc
    port: 14250
    targetPort: 14250
  - name: jaeger-http
    port: 14268
    targetPort: 14268
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query-lb
  namespace: observability-tracing
  labels:
    app: jaeger
spec:
  type: LoadBalancer
  selector:
    app: jaeger
  ports:
  - name: query
    port: 16686
    targetPort: 16686
EOF
```

Aplicar:

```bash
kubectl apply -f manifests/jaeger-all-in-one.yaml
```

Validar:

```bash
kubectl get pods -n $TRACE_NS
kubectl get svc -n $TRACE_NS
kubectl describe svc jaeger-query-lb -n $TRACE_NS
```

Esperar IP pública:

```bash
kubectl get svc jaeger-query-lb -n $TRACE_NS -w
```

Obtener URL de Jaeger:

```bash
export JAEGER_IP=$(kubectl get svc jaeger-query-lb -n $TRACE_NS -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://$JAEGER_IP:16686"
```

Abrir en el navegador:

```text
http://<JAEGER_IP>:16686
```

---

## 10. Validación técnica importante

El usuario pidió validar con:

```bash
kubectl get traces
```

Pero esto no es un comando nativo de Kubernetes.

Los traces no son objetos Kubernetes. Son datos de telemetry almacenados en Jaeger. Por eso la validación correcta es:

```bash
kubectl get pods -n $TRACE_NS
kubectl get svc -n $TRACE_NS
curl http://$JAEGER_IP:16686/api/services
```

Si todavía no hay tráfico instrumentado, el endpoint puede responder sin servicios o con una lista vacía.

Mensaje clave para estudiantes:

> **Kubernetes administra workloads; Jaeger administra telemetry. No todo lo que observamos en una plataforma cloud-native existe como objeto Kubernetes.**

---

# Parte 2 — Instalar OpenTelemetry Collector

## 11. Arquitectura del Collector

OpenTelemetry Collector actuará como gateway central.

```text
Microservicios / Istio / SDKs
        |
        | OTLP gRPC / OTLP HTTP
        v
OpenTelemetry Collector
        |
        +--> traces --> Jaeger
        |
        +--> metrics opcionales --> Prometheus/Grafana
        |
        +--> logs opcionales --> Elastic u otros backends
```

Beneficios:

- desacopla aplicaciones de Jaeger;
- permite cambiar backend sin tocar microservicios;
- centraliza sampling, batching y transformación;
- permite pipelines distintos para traces, metrics y logs;
- reduce acoplamiento operacional.

---

## 12. Crear OpenTelemetry Collector

Crear ConfigMap y Deployment:

```bash
cat <<EOF > manifests/otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability-tracing
data:
  collector.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      memory_limiter:
        check_interval: 5s
        limit_mib: 400
        spike_limit_mib: 100
      batch:
        timeout: 5s
        send_batch_size: 1024

    exporters:
      otlp/jaeger:
        endpoint: jaeger-collector.observability-tracing.svc.cluster.local:4317
        tls:
          insecure: true
      debug:
        verbosity: basic
      prometheus:
        endpoint: 0.0.0.0:8889

    service:
      telemetry:
        metrics:
          address: 0.0.0.0:8888
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger, debug]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus, debug]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability-tracing
  labels:
    app: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.152.0
        imagePullPolicy: IfNotPresent
        args:
        - "--config=/conf/collector.yaml"
        ports:
        - name: otlp-grpc
          containerPort: 4317
        - name: otlp-http
          containerPort: 4318
        - name: metrics
          containerPort: 8888
        - name: prom-exporter
          containerPort: 8889
        volumeMounts:
        - name: config
          mountPath: /conf
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability-tracing
  labels:
    app: otel-collector
spec:
  type: ClusterIP
  selector:
    app: otel-collector
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
  - name: metrics
    port: 8888
    targetPort: 8888
  - name: prom-exporter
    port: 8889
    targetPort: 8889
EOF
```

Aplicar:

```bash
kubectl apply -f manifests/otel-collector.yaml
```

Validar:

```bash
kubectl get pods -n $TRACE_NS
kubectl logs deploy/otel-collector -n $TRACE_NS --tail=100
kubectl get svc otel-collector -n $TRACE_NS
```

Probar puertos:

```bash
kubectl port-forward svc/otel-collector 8888:8888 -n $TRACE_NS
```

En otra terminal:

```bash
curl http://localhost:8888/metrics | head
```

---

# Parte 3 — Integración con Istio

## 13. Objetivo de la integración

La integración con Istio permite que Envoy genere spans automáticamente para tráfico HTTP dentro del service mesh.

Esto es útil porque:

- no requiere modificar código;
- muestra llamadas entre servicios;
- permite visualizar latencia east-west;
- permite observar errores HTTP;
- permite entender rutas de tráfico y retries.

Limitación:

> Istio no ve automáticamente operaciones internas como queries MongoDB o publish a RabbitMQ si esas operaciones ocurren dentro del proceso de aplicación y no están instrumentadas por OpenTelemetry SDK.

---

## 14. Validar si Istio está instalado

```bash
kubectl get ns istio-system
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

Validar sidecars en el namespace del laboratorio:

```bash
kubectl get pods -n $NS -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{.spec.containers[*].name}{"\n"}{end}'
```

Si ves containers como `istio-proxy`, los pods tienen sidecar Envoy.

---

## 15. Configurar Istio para enviar traces al OpenTelemetry Collector

> Este paso depende de cómo se instaló Istio. Si Istio fue instalado con `istioctl`, puedes aplicar configuración con IstioOperator. Si fue instalado con Helm o addon administrado, ajusta el método según tu instalación.

Crear archivo de configuración:

```bash
cat <<EOF > istio/istio-otel-extension-provider.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    extensionProviders:
    - name: otel-tracing
      opentelemetry:
        service: otel-collector.observability-tracing.svc.cluster.local
        port: 4317
EOF
```

Aplicar con `istioctl`:

```bash
istioctl install -f istio/istio-otel-extension-provider.yaml -y
```

Validar:

```bash
kubectl get cm istio -n istio-system -o yaml | grep -A20 extensionProviders
```

---

## 16. Crear Telemetry API para activar tracing

```bash
cat <<EOF > istio/telemetry-tracing.yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default-tracing
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: otel-tracing
    randomSamplingPercentage: 100
EOF
```

Aplicar:

```bash
kubectl apply -f istio/telemetry-tracing.yaml
```

Validar:

```bash
kubectl get telemetry -A
kubectl describe telemetry mesh-default-tracing -n istio-system
```

Reiniciar workloads del laboratorio para garantizar que los sidecars tomen la configuración:

```bash
kubectl rollout restart deployment -n $NS
kubectl get pods -n $NS -w
```

---

## 17. Generar tráfico para crear traces

Identificar endpoint de entrada.

Si usas Istio Ingress Gateway:

```bash
export GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

Generar tráfico:

```bash
for i in {1..50}; do
  curl -s -o /dev/null http://$GATEWAY_IP
  sleep 1
done
```

Si usas `store-front` como LoadBalancer directo:

```bash
kubectl get svc -n $NS
export STORE_IP=$(kubectl get svc store-front -n $NS -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $STORE_IP
```

```bash
for i in {1..50}; do
  curl -s -o /dev/null http://$STORE_IP
  sleep 1
done
```

---

## 18. Ver traces en Jaeger

Abrir Jaeger:

```bash
echo "http://$JAEGER_IP:16686"
```

En la UI:

```text
Search
  → Service
  → seleccionar servicio disponible
  → Find Traces
```

Si no aparecen servicios:

1. Validar OTel Collector:

```bash
kubectl logs deploy/otel-collector -n $TRACE_NS --tail=100
```

2. Validar Jaeger:

```bash
kubectl logs deploy/jaeger -n $TRACE_NS --tail=100
```

3. Validar Telemetry API:

```bash
kubectl get telemetry -A
```

4. Validar configuración de Istio:

```bash
kubectl get cm istio -n istio-system -o yaml | grep -A30 extensionProviders
```

---

# Parte 4 — Instrumentación de aplicación con OpenTelemetry

## 19. Cuándo necesitas instrumentación de aplicación

La instrumentación de Istio es útil, pero no suficiente para todo.

Necesitas instrumentación de aplicación cuando quieres observar:

- publish a RabbitMQ;
- consume de RabbitMQ;
- query a MongoDB;
- operación Redis;
- validación de orden;
- reglas de negocio;
- retries internos;
- errores capturados por código;
- spans de negocio.

---

## 20. Variables estándar de OpenTelemetry

Estas variables funcionan como referencia general para aplicaciones que ya tienen SDK o auto-instrumentation compatible.

```bash
OTEL_SERVICE_NAME=order-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability-tracing.svc.cluster.local:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_PROPAGATORS=tracecontext,baggage
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=lab,service.namespace=aks-store
```

Aplicar variables a un deployment:

```bash
kubectl set env deployment/order-service -n $NS \
  OTEL_SERVICE_NAME=order-service \
  OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability-tracing.svc.cluster.local:4318 \
  OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf \
  OTEL_TRACES_EXPORTER=otlp \
  OTEL_METRICS_EXPORTER=otlp \
  OTEL_PROPAGATORS=tracecontext,baggage \
  OTEL_RESOURCE_ATTRIBUTES=deployment.environment=lab,service.namespace=aks-store
```

Repetir para otros servicios, ajustando `OTEL_SERVICE_NAME`:

```bash
kubectl set env deployment/store-front -n $NS OTEL_SERVICE_NAME=store-front
kubectl set env deployment/product-service -n $NS OTEL_SERVICE_NAME=product-service
kubectl set env deployment/makeline-service -n $NS OTEL_SERVICE_NAME=makeline-service
```

> **Nota importante:** Estas variables solo generan telemetry si la aplicación tiene OpenTelemetry SDK, auto-instrumentation o agente compatible. Si la imagen no está instrumentada, no aparecerán spans de aplicación solo por agregar variables.

---

## 21. Diseño recomendado para instrumentación real

En producción o en un lab avanzado con código fuente disponible, el diseño recomendado es:

```text
Application Code
   |
   +--> OpenTelemetry SDK
   |       |
   |       +--> HTTP spans
   |       +--> RabbitMQ spans
   |       +--> MongoDB spans
   |       +--> Redis spans
   |       +--> Business spans
   |
   v
OpenTelemetry Collector
   |
   v
Jaeger / Elastic / Prometheus / Grafana
```

Spans sugeridos para el dominio del laboratorio:

| Servicio | Span recomendado |
|---|---|
| store-front | `http.request.checkout` |
| order-service | `order.create` |
| order-service | `rabbitmq.publish.orders` |
| makeline-service | `rabbitmq.consume.orders` |
| makeline-service | `redis.get.product` |
| makeline-service | `mongodb.insert.order` |
| product-service | `mongodb.find.products` |
| store-admin | `mongodb.update.product` |

Buenas prácticas:

- propagar `traceparent`;
- incluir `correlationId` de negocio;
- no registrar datos sensibles;
- usar nombres consistentes;
- agregar atributos de negocio no sensibles;
- mantener sampling controlado;
- usar Collector como gateway central.

---

# Parte 5 — Integración opcional con Prometheus y Grafana

## 22. Qué se puede integrar realmente

OpenTelemetry Collector ya expone métricas propias en:

```text
http://otel-collector.observability-tracing.svc.cluster.local:8888/metrics
```

Y puede exponer métricas recibidas por OTLP en:

```text
http://otel-collector.observability-tracing.svc.cluster.local:8889/metrics
```

Esto se puede consumir desde Prometheus si tienes:

- kube-prometheus-stack;
- Prometheus propio;
- Prometheus compatible con scrape custom;
- Azure Managed Prometheus con configuración de scraping custom habilitada.

Limitación real:

> En Azure Managed Prometheus, el scraping de métricas custom puede requerir configuración adicional del addon de Azure Monitor. Por eso en este módulo se deja como integración opcional.

---

## 23. Validar endpoint Prometheus del Collector

Port-forward:

```bash
kubectl port-forward svc/otel-collector 8889:8889 -n $TRACE_NS
```

Probar:

```bash
curl http://localhost:8889/metrics | head
```

Si no aparecen métricas de aplicación, es normal si las apps no están enviando métricas OTLP.

---

## 24. ServiceMonitor opcional si usas kube-prometheus-stack

Si tienes Prometheus Operator instalado:

```bash
cat <<EOF > manifests/otel-collector-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector
  namespace: observability-tracing
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: otel-collector
  namespaceSelector:
    matchNames:
    - observability-tracing
  endpoints:
  - port: metrics
    interval: 30s
  - port: prom-exporter
    interval: 30s
EOF
```

Aplicar solo si existe el CRD:

```bash
kubectl get crd servicemonitors.monitoring.coreos.com
kubectl apply -f manifests/otel-collector-servicemonitor.yaml
```

---

## 25. Dashboards sugeridos en Grafana

Crear o extender dashboard:

```text
AKS Store Demo – Tracing + SRE
```

Paneles sugeridos:

| Panel | Fuente |
|---|---|
| Traces por servicio | Jaeger UI |
| p95/p99 latency por servicio | Prometheus / Istio metrics |
| HTTP 5xx por servicio | Prometheus / Istio metrics |
| OTel Collector accepted spans | Prometheus exporter |
| OTel Collector dropped spans | Prometheus exporter |
| RabbitMQ backlog | RabbitMQ metrics |
| Pods restarts | Azure Managed Prometheus |
| Incidentes recientes | Elastic/Kibana |
| TraceId lookup | Kibana + Jaeger |

---

# Parte 6 — Escenarios de fallo con Jaeger

## 26. Escenario 1 — Latencia alta con Istio Fault Injection

### Objetivo

Simular latencia en un servicio para verla reflejada en traces.

### Crear VirtualService con delay

```bash
cat <<EOF > incidents/order-service-delay.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service-delay
  namespace: aks-store-state-lab
spec:
  hosts:
  - order-service
  http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 3s
    route:
    - destination:
        host: order-service
EOF
```

Aplicar:

```bash
kubectl apply -f incidents/order-service-delay.yaml
```

Generar tráfico:

```bash
for i in {1..20}; do
  curl -s -o /dev/null http://$GATEWAY_IP
  sleep 1
done
```

Observar en Jaeger:

- spans más largos;
- duración total mayor;
- servicio con latencia artificial.

Discusión:

- ¿El servicio está caído o degradado?
- ¿Qué diferencia hay entre availability y latency?
- ¿El SLO debería incluir p95/p99?
- ¿Qué pasaría si los clients tienen timeouts agresivos?

Eliminar delay:

```bash
kubectl delete -f incidents/order-service-delay.yaml
```

---

## 27. Escenario 2 — makeline-service apagado

### Objetivo

Observar cómo un consumer caído afecta un flujo event-driven.

Apagar:

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
```

Validar:

```bash
kubectl get deploy makeline-service -n $NS
kubectl get pods -n $NS
```

Generar órdenes:

```bash
for i in {1..30}; do
  curl -s -o /dev/null http://$GATEWAY_IP
  sleep 1
done
```

Evidencia esperada:

- RabbitMQ backlog aumenta.
- No hay spans de consumo de makeline-service si está apagado.
- Order-service puede seguir aceptando órdenes si el diseño es async.
- El sistema queda parcialmente disponible, pero el procesamiento queda degradado.

Validar RabbitMQ:

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

Credenciales típicas:

```text
username: username
password: password
```

Restaurar:

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=makeline-service -n $NS --timeout=180s
```

Discusión:

- ¿Por qué los traces pueden quedar incompletos?
- ¿Cómo se observa un fallo asíncrono?
- ¿Qué diferencia hay entre `TraceId` técnico y `CorrelationId` de negocio?
- ¿Qué patrón ayudaría? DLQ, idempotencia, outbox, retry con backoff.

---

## 28. Escenario 3 — RabbitMQ caído

### Objetivo

Observar qué ocurre cuando falla el broker de mensajes.

Apagar RabbitMQ:

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
```

Validar:

```bash
kubectl get statefulset rabbitmq -n $NS
kubectl get pods -n $NS
```

Generar tráfico:

```bash
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
  sleep 1
done
```

Buscar evidencia:

```bash
kubectl logs deploy/order-service -n $NS --tail=100
```

En Elastic/Kibana buscar:

```text
rabbitmq
connection refused
failed
order-service
```

En Jaeger observar:

- spans HTTP hacia order-service;
- posible error interno si la app devuelve error;
- ausencia de spans AMQP si la app no está instrumentada a nivel de librería.

Restaurar:

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $NS --timeout=180s
```

Discusión:

- ¿Debe order-service aceptar órdenes si RabbitMQ está caído?
- ¿Existe Outbox Pattern?
- ¿Qué haría un diseño enterprise?
- ¿Qué métrica alertaría antes que el usuario?
- ¿Qué span faltaría si no hay instrumentación de aplicación?

---

## 29. Escenario 4 — MongoDB caído

### Objetivo

Diagnosticar degradación de persistencia.

Apagar MongoDB:

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
```

Validar:

```bash
kubectl get statefulset mongodb -n $NS
kubectl get pods -n $NS
```

Generar tráfico:

```bash
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
  sleep 1
done
```

Buscar logs:

```bash
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
```

Buscar en Kibana:

```text
mongodb
connection refused
server selection timeout
deadline exceeded
```

Observar en Jaeger:

- spans HTTP exitosos o fallidos;
- latencia elevada si hay retries;
- errores asociados si hay instrumentación;
- ausencia de spans MongoDB si no hay SDK instrumentation.

Restaurar:

```bash
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=mongodb -n $NS --timeout=180s
```

Discusión:

- ¿Por qué “pod Running” no significa “negocio saludable”?
- ¿Qué operaciones siguen funcionando?
- ¿Qué operaciones fallan?
- ¿Qué patrón ayuda? Cache aside, fallback, circuit breaker, read model.

---

## 30. Escenario 5 — OOMKilled en makeline-service

### Objetivo

Observar cómo un problema de recursos genera impacto distribuido.

Reducir memoria de makeline-service:

```bash
kubectl set resources deployment makeline-service -n $NS \
  --requests=memory=32Mi \
  --limits=memory=64Mi
```

Reiniciar:

```bash
kubectl rollout restart deployment makeline-service -n $NS
kubectl get pods -n $NS -w
```

Validar eventos:

```bash
kubectl describe pod -n $NS -l app=makeline-service
```

Buscar OOMKilled:

```bash
kubectl get pods -n $NS
kubectl describe pod -n $NS -l app=makeline-service | grep -i oom -C 3
```

Evidencia esperada:

- restarts;
- posible CrashLoopBackOff;
- backlog en RabbitMQ;
- traces incompletos;
- logs interrumpidos.

Restaurar recursos:

```bash
kubectl set resources deployment makeline-service -n $NS \
  --requests=memory=256Mi \
  --limits=memory=512Mi

kubectl rollout restart deployment makeline-service -n $NS
```

Discusión:

- ¿Cómo se relaciona OOMKilled con backlog?
- ¿Por qué Kubernetes reinicia, pero no resuelve root cause?
- ¿Qué falta? memory profiling, HPA/VPA, límites correctos, carga realista.

---

## 31. Escenario 6 — Retries visibles con Istio

### Objetivo

Mostrar cómo retries pueden mejorar resiliencia o empeorar una cascada.

Crear VirtualService con retries:

```bash
cat <<EOF > incidents/order-service-retries.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: order-service-retries
  namespace: aks-store-state-lab
spec:
  hosts:
  - order-service
  http:
  - retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx,connect-failure,refused-stream
    route:
    - destination:
        host: order-service
EOF
```

Aplicar:

```bash
kubectl apply -f incidents/order-service-retries.yaml
```

Generar tráfico:

```bash
for i in {1..30}; do
  curl -s -o /dev/null http://$GATEWAY_IP
  sleep 1
done
```

Observar:

- métricas de Istio;
- latencia;
- posibles spans más largos;
- comportamiento bajo fallo.

Eliminar:

```bash
kubectl delete -f incidents/order-service-retries.yaml
```

Discusión:

- ¿Cuándo un retry ayuda?
- ¿Cuándo un retry se vuelve retry storm?
- ¿Por qué se necesita retry budget?
- ¿Qué relación hay con circuit breaker y backpressure?

---

# Parte 7 — Correlación entre métricas, logs y traces

## 32. Modelo de diagnóstico SRE

Durante un incidente, usar este orden:

```text
1. Métricas
   ¿Qué cambió?
   ¿Cuándo empezó?
   ¿Cuánto impacto tiene?

2. Logs
   ¿Qué error exacto ocurrió?
   ¿Qué componente lo reportó?

3. Traces
   ¿Dónde se demoró o rompió la request?
   ¿Qué dependencia aparece lenta o ausente?

4. Kubernetes
   ¿Qué objeto confirma el estado?
   ¿Deployment, pod, statefulset, service, events?

5. Arquitectura
   ¿Qué patrón falló?
   ¿Qué anti-patrón permitió la degradación?
```

---

## 33. Plantilla de diagnóstico

Crear archivo:

```bash
cat <<EOF > notes/tracing-incident-diagnosis.md
# Distributed Tracing Incident Diagnosis

## Incident Name
-

## Start Time
-

## Business Impact
-

## Metrics Evidence
-

## Logs Evidence
-

## Trace Evidence
-

## Kubernetes Evidence
-

## Suspected Root Cause
-

## Confirmed Root Cause
-

## Recovery Action
-

## Architecture Pattern Involved
-

## Anti-Pattern Detected
-

## Preventive Improvement
-

## SLO / SLI Recommendation
-
EOF
```

---

## 34. Ejemplo de correlación

| Señal | Evidencia | Interpretación |
|---|---|---|
| Prometheus | p95 latency sube | Degradación visible |
| Grafana | error rate aumenta | Impacto operacional |
| Elastic | `connection refused` | Error técnico |
| Jaeger | span largo en order-service | Cuello de botella |
| Kubernetes | pod restart | Problema de runtime |
| RabbitMQ | backlog crece | Consumer no procesa |
| Arquitectura | no DLQ | Riesgo de pérdida o reprocesamiento |

---

# Parte 8 — Diseño enterprise/telco

## 35. Diseño recomendado para producción

Para un entorno enterprise/telco, no usar Jaeger All-in-One.

Diseño recomendado:

```text
Apps / Envoy / SDKs
   |
   v
OpenTelemetry Collector Agent / Gateway
   |
   +--> Jaeger Collector
   |       |
   |       v
   |   Storage backend
   |
   +--> Prometheus remote write / Azure Monitor
   |
   +--> Elastic / SIEM
```

Componentes recomendados:

- OpenTelemetry Collector como gateway escalable.
- Jaeger Collector separado de Jaeger Query.
- Storage persistente para Jaeger.
- Sampling controlado.
- Retention definida.
- RBAC y network policies.
- TLS/mTLS entre componentes.
- Dashboards basados en SLO.
- Alertas por síntomas, no por ruido.

---

## 36. Consideraciones de almacenamiento de Jaeger

| Modo | Uso recomendado |
|---|---|
| All-in-One memory | Laboratorio y demos |
| Badger local | Demo avanzada o pruebas pequeñas |
| Elasticsearch | Producción o alto volumen |
| Cassandra | Producción distribuida de alto volumen |
| OpenSearch | Producción compatible con Elastic ecosystem |

Mensaje clave:

> **El backend de tracing puede crecer rápidamente. Sin sampling, retention e ILM, tracing puede volverse costoso.**

---

## 37. Seguridad

Exponer Jaeger con LoadBalancer es práctico para el laboratorio, pero no es recomendable en producción sin controles.

Buenas prácticas:

- restringir IPs permitidas;
- usar Ingress con autenticación;
- proteger con Entra ID / OAuth2 proxy;
- usar Private Link o red interna;
- aplicar NetworkPolicy;
- usar TLS;
- no exponer telemetry sensible;
- evitar datos personales en span attributes.

Anti-patrón:

> Jaeger UI pública en internet sin autenticación.

---

## 38. Buenas prácticas de sampling

Para laboratorio:

```text
randomSamplingPercentage: 100
```

Para producción:

- no usar 100% por defecto en tráfico alto;
- usar probabilistic sampling;
- usar tail-based sampling para errores o latencia alta;
- muestrear más errores que éxitos;
- mantener trazas críticas de negocio.

Ejemplos:

| Estrategia | Uso |
|---|---|
| Head sampling | Simple y barato |
| Tail sampling | Mejor para errores y latencia |
| Probabilistic sampling | Control de volumen |
| AlwaysOn | Solo laboratorio o bajo tráfico |
| AlwaysOff | Para servicios no críticos o pruebas |

---

## 39. Patrones arquitectónicos recomendados

### 39.1 Trace Context Propagation

Propagar:

```text
traceparent
tracestate
baggage
```

Aplica a:

- HTTP;
- gRPC;
- eventos;
- mensajes RabbitMQ/Kafka.

### 39.2 Correlation ID de negocio

Usar `correlationId` para:

- orden;
- cliente;
- transacción;
- workflow;
- saga.

Diferencia clave:

| ID | Propósito |
|---|---|
| TraceId | Ejecución técnica distribuida |
| CorrelationId | Transacción o proceso de negocio |

### 39.3 Outbox Pattern

Muy importante cuando RabbitMQ es crítico.

Evita perder mensajes si:

- order-service guarda en DB pero no publica evento;
- RabbitMQ cae;
- hay retry parcial.

### 39.4 Idempotent Consumer

Makeline-service debe procesar eventos repetidos sin duplicar efectos.

### 39.5 Backpressure

Si la cola crece, el sistema debe:

- escalar consumers;
- limitar producers;
- aplicar circuit breaker;
- alertar antes de saturación.

### 39.6 Circuit Breaker

Si MongoDB o RabbitMQ fallan, evitar retries infinitos.

### 39.7 Bulkhead

Separar recursos por dominio:

- thread pools;
- connection pools;
- queues;
- deployments;
- node pools si aplica.

---

# Parte 9 — Runbook de troubleshooting

## 40. Si no aparecen traces en Jaeger

Validar Jaeger:

```bash
kubectl get pods -n $TRACE_NS
kubectl logs deploy/jaeger -n $TRACE_NS --tail=100
```

Validar Collector:

```bash
kubectl get pods -n $TRACE_NS
kubectl logs deploy/otel-collector -n $TRACE_NS --tail=100
```

Validar servicio:

```bash
kubectl get svc -n $TRACE_NS
```

Validar Istio:

```bash
kubectl get telemetry -A
kubectl get cm istio -n istio-system -o yaml | grep -A30 extensionProviders
```

Validar sidecars:

```bash
kubectl get pods -n $NS -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{.spec.containers[*].name}{"\n"}{end}'
```

Validar tráfico:

```bash
for i in {1..20}; do curl -s -o /dev/null http://$GATEWAY_IP; done
```

---

## 41. Problemas comunes

| Problema | Posible causa | Solución |
|---|---|---|
| Jaeger UI abre, pero no hay servicios | No hay traces enviados | Validar OTel Collector e Istio |
| OTel Collector no inicia | Error en config YAML | Revisar logs del collector |
| No hay spans de RabbitMQ | Falta instrumentación app-level | Agregar OTel SDK |
| No hay spans de MongoDB | Istio no ve llamadas internas de librería | Instrumentar driver |
| Spans incompletos | No se propaga contexto | Usar W3C Trace Context |
| Demasiados traces | Sampling 100% | Ajustar sampling |
| Jaeger pierde traces | All-in-One memory | Usar storage persistente |
| LoadBalancer no obtiene IP | Restricción cloud/network | Revisar Azure LB y permisos |
| 403 en app | Puede ser Istio/JWT esperado | Validar policy antes de declarar incidente |

---

# Parte 10 — Limpieza

## 42. Eliminar incidentes

```bash
kubectl delete -f incidents/order-service-delay.yaml --ignore-not-found=true
kubectl delete -f incidents/order-service-retries.yaml --ignore-not-found=true
```

Restaurar componentes:

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
```

Restaurar recursos de makeline-service si los cambiaste:

```bash
kubectl set resources deployment makeline-service -n $NS \
  --requests=memory=256Mi \
  --limits=memory=512Mi
```

---

## 43. Eliminar OpenTelemetry y Jaeger

```bash
kubectl delete -f manifests/otel-collector.yaml --ignore-not-found=true
kubectl delete -f manifests/jaeger-all-in-one.yaml --ignore-not-found=true
```

Eliminar namespace:

```bash
kubectl delete ns $TRACE_NS
```

Eliminar Telemetry API:

```bash
kubectl delete -f istio/telemetry-tracing.yaml --ignore-not-found=true
```

> Si modificaste Istio con `istioctl install`, revisa tu configuración original antes de revertir. En un entorno compartido, no elimines Istio ni modifiques MeshConfig sin validarlo con el equipo.

---

# Parte 11 — Cierre del módulo

## 44. Mensaje final para estudiantes

En microservicios, la pregunta no es solo:

```text
¿El pod está Running?
```

La pregunta real es:

```text
¿La operación de negocio completó correctamente a través de todos los servicios y dependencias?
```

Para responder eso necesitas:

- métricas para ver síntomas;
- logs para ver eventos;
- traces para ver el recorrido;
- Kubernetes para confirmar estado;
- arquitectura para interpretar el fallo.

---

## 45. Conclusiones clave

- OpenTelemetry es el estándar moderno para capturar telemetry.
- Jaeger permite visualizar el recorrido de requests distribuidas.
- Istio puede generar trazas de red sin tocar código.
- Para RabbitMQ, MongoDB y Redis se necesita instrumentación app-level.
- Prometheus y Grafana muestran síntomas y tendencias.
- Elastic/Kibana explica eventos y errores.
- Jaeger muestra dónde se demora o rompe el flujo.
- Un buen diseño enterprise usa Collector como gateway central.
- Tracing sin sampling, retention y seguridad puede volverse costoso y riesgoso.
- En arquitecturas telco-grade, observabilidad es parte del diseño, no un agregado posterior.

---

## 46. Preguntas de reflexión

1. ¿Qué parte del flujo solo puede verse con traces?
2. ¿Qué parte solo puede verse con logs?
3. ¿Qué parte solo puede verse con métricas?
4. ¿Dónde termina la visibilidad de Istio y empieza la necesidad de SDK?
5. ¿Cómo propagarías contexto en RabbitMQ?
6. ¿Qué SLO definirías para order-service?
7. ¿Qué alerta detectaría backlog antes del impacto al usuario?
8. ¿Qué cambiarías para producción?
9. ¿Cómo protegerías Jaeger UI?
10. ¿Qué anti-patrón fue más evidente durante el laboratorio?

---

## 47. Referencias oficiales sugeridas

- OpenTelemetry Collector para Kubernetes.
- OpenTelemetry Collector Helm Chart.
- Jaeger official documentation.
- Jaeger Helm Charts.
- Istio Distributed Tracing.
- Istio Telemetry API.
- Kubernetes Services y LoadBalancer.
- Azure Managed Prometheus.
- Azure Managed Grafana.
- Elastic Observability.
