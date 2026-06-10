# Advanced Lab – OpenTelemetry Collector + Jaeger for Distributed Tracing on AKS

**Course:** Advanced Microservices, Kubernetes, Service Mesh and Observability  
**Environment:** Azure Kubernetes Service (AKS) + Istio + AKS Store Demo / E-commerce microservices  
**Level:** Advanced – Senior Engineer / Architect / SRE  
**Estimated Duration:** 4 to 6 hours  
**Primary Focus:** OpenTelemetry Collector, Jaeger, Zipkin-compatible ingestion, Istio tracing, context propagation, sampling and troubleshooting

---

## 1. Lab Purpose

During the previous observability lab, students validated centralized logs, metrics, dashboards, basic tracing and failure diagnosis.

This complementary lab goes deeper into **distributed tracing**.

The objective is not only to "install Jaeger" or "enable OpenTelemetry."  
The real objective is to answer operational questions such as:

- Where did a distributed request spend most of its time?
- Which service introduced latency?
- Did the request reach the backend services?
- Did context propagation work across service boundaries?
- Is the trace complete or fragmented?
- Is the OpenTelemetry Collector receiving, processing and exporting telemetry correctly?
- What happens when sampling is changed?
- How would this design change in production?

---

## 2. Key Message for Students

Do not start with the tool.

The wrong question is:

> Do we need OpenTelemetry, Jaeger or Zipkin?

The better question is:

> What problem are we trying to solve?

In a monolith, one log file may be enough.

In a distributed microservices architecture, one user request may cross:

```text
Browser
  |
  v
Istio Ingress Gateway
  |
  v
Frontend Service
  |
  v
Product Service
  |
  v
Order Service
  |
  v
RabbitMQ
  |
  v
Makeline Service
  |
  v
MongoDB / Redis
```

Without distributed tracing, troubleshooting becomes guesswork.

With distributed tracing, we can see the complete request path, latency per hop, dependencies, failed calls and broken context propagation.

---

## 3. What Students Will Build

In this lab, students will deploy a dedicated tracing stack inside AKS:

```text
Application Pods with Istio Sidecars
        |
        | Envoy-generated traces
        v
OpenTelemetry Collector
        |
        | OTLP
        v
Jaeger All-in-One
        |
        v
Jaeger UI
```

The collector will also support Zipkin-compatible input:

```text
Manual Zipkin Span
        |
        | Zipkin API
        v
OpenTelemetry Collector
        |
        | OTLP
        v
Jaeger
```

This shows that OpenTelemetry can act as a neutral telemetry pipeline.

---

## 4. What This Lab Covers

### Technical Topics

- OpenTelemetry architecture
- OpenTelemetry Collector
- Receivers
- Processors
- Exporters
- Pipelines
- OTLP
- Zipkin-compatible ingestion
- Jaeger backend integration
- Istio Telemetry API
- Distributed tracing concepts
- Trace context
- Baggage
- Head sampling
- Tail sampling
- Troubleshooting incomplete traces
- Production considerations

### Operational Topics

- Service-to-service trace visibility
- Latency analysis
- Error analysis
- Trace correlation
- Root cause analysis
- Sampling strategy
- Resilience and observability tradeoffs

---

## 5. Prerequisites

This lab assumes the existing course environment already has:

- AKS cluster
- `kubectl` connected to the cluster
- Istio installed
- Istio ingress gateway configured
- Application microservices already deployed
- Application namespace either:
  - `default`, or
  - `aks-store-state-lab`
- Existing components such as:
  - frontend
  - product-service
  - order-service
  - makeline-service
  - RabbitMQ
  - MongoDB
  - Redis

Optional but useful:

- Existing Jaeger from the previous lab
- Existing Telemetry API configuration from the previous lab
- Existing valid JWT token file named `valid-token.txt`

---

## 6. Lab Conventions

This lab uses a separate namespace for the advanced tracing stack:

```text
otel-advanced
```

The application remains in its existing namespace.

Run all commands from **Azure Cloud Shell Bash** or a local terminal with:

```bash
az login
az account set --subscription "<your-subscription-id>"
kubectl get nodes
```

---

## 7. Prepare Variables

Create a working folder:

```bash
mkdir -p otel-jaeger-advanced
cd otel-jaeger-advanced
```

Set variables:

```bash
export OTEL_NS="otel-advanced"

# Auto-detect the application namespace used in this workshop.
if kubectl get ns aks-store-state-lab >/dev/null 2>&1; then
  export APP_NS="aks-store-state-lab"
else
  export APP_NS="default"
fi

echo "OpenTelemetry namespace: $OTEL_NS"
echo "Application namespace:   $APP_NS"
```

Validate the application namespace:

```bash
kubectl get pods -n $APP_NS
kubectl get svc -n $APP_NS
```

Expected result:

You should see the application services and pods, for example:

```text
frontend
product-service
order-service
makeline-service
rabbitmq
mongodb
redis
```

If nothing appears, set the namespace manually:

```bash
export APP_NS="default"
# or
export APP_NS="aks-store-state-lab"
```

---

## 8. Pre-Check Istio

Validate Istio system pods:

```bash
kubectl get pods -n istio-system
```

Validate ingress gateway:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Get gateway IP:

```bash
export GATEWAY_IP=$(kubectl get svc istio-ingressgateway \
  -n istio-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo $GATEWAY_IP
```

Test the application:

```bash
curl -i http://$GATEWAY_IP
```

If your security lab enabled JWT and you get `403`, that is not necessarily a problem.  
It may simply mean the Istio authorization policy is working.

If you have a valid token:

```bash
export VALID_TOKEN=$(cat ../valid-token.txt 2>/dev/null || cat valid-token.txt 2>/dev/null || true)

if [ -n "$VALID_TOKEN" ]; then
  curl -i -H "Authorization: Bearer $VALID_TOKEN" http://$GATEWAY_IP
else
  echo "No valid-token.txt found. Testing without token."
  curl -i http://$GATEWAY_IP
fi
```

---

## 9. What Is Happening?

Before deploying anything, clarify the architecture.

### OpenTelemetry SDK

The SDK lives inside the application process.

It can create spans such as:

```text
GET /checkout
DB query
RabbitMQ publish
Redis lookup
```

This requires application code instrumentation or auto-instrumentation.

### Istio / Envoy Tracing

Istio sidecars can generate network-level spans without changing application code.

They can show:

```text
frontend -> product-service
frontend -> order-service
order-service -> rabbitmq
```

However, they may not show internal code operations such as:

```text
validate business rule
calculate tax
serialize message
execute MongoDB query
```

For those internal spans, you usually need OpenTelemetry SDK instrumentation.

### OpenTelemetry Collector

The Collector is a telemetry pipeline.

It receives telemetry, processes telemetry and exports telemetry.

```text
Receivers -> Processors -> Exporters
```

### Jaeger

Jaeger stores and visualizes distributed traces.

In this lab, Jaeger is the backend used by students to inspect traces visually.

---

## 10. Deploy Jaeger Advanced Instance

This lab deploys a new Jaeger instance dedicated to this exercise.

Create namespace:

```bash
kubectl create namespace $OTEL_NS --dry-run=client -o yaml | kubectl apply -f -
```

Create Jaeger manifest:

```bash
cat <<'EOF' > jaeger-advanced.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-advanced
  namespace: otel-advanced
  labels:
    app: jaeger-advanced
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger-advanced
  template:
    metadata:
      labels:
        app: jaeger-advanced
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.76.0
        imagePullPolicy: IfNotPresent
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        - name: COLLECTOR_ZIPKIN_HOST_PORT
          value: ":9411"
        ports:
        - containerPort: 16686
          name: query
        - containerPort: 4317
          name: otlp-grpc
        - containerPort: 4318
          name: otlp-http
        - containerPort: 14250
          name: jaeger-grpc
        - containerPort: 14268
          name: jaeger-http
        - containerPort: 9411
          name: zipkin
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-advanced
  namespace: otel-advanced
  labels:
    app: jaeger-advanced
spec:
  selector:
    app: jaeger-advanced
  ports:
  - name: query
    port: 16686
    targetPort: 16686
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
  - name: zipkin
    port: 9411
    targetPort: 9411
EOF
```

Apply:

```bash
kubectl apply -f jaeger-advanced.yaml
```

Validate:

```bash
kubectl get pods -n $OTEL_NS
kubectl get svc -n $OTEL_NS
kubectl rollout status deployment/jaeger-advanced -n $OTEL_NS --timeout=180s
```

Open Jaeger UI:

```bash
kubectl port-forward svc/jaeger-advanced -n $OTEL_NS 16686:16686
```

Open in browser:

```text
http://localhost:16686
```

### What Is Happening?

Jaeger all-in-one contains multiple components in one pod:

```text
Jaeger Collector
Jaeger Query
Jaeger UI
In-memory storage
```

This is appropriate for a lab.

### Production Reality

In production, do not usually use all-in-one.

A production Jaeger architecture normally separates:

```text
Collectors
Query services
Storage backend
Retention policy
Ingress / authentication
Horizontal scaling
```

Storage may use backends such as Elasticsearch, Cassandra, OpenSearch or a vendor-managed tracing platform.

---

## 11. Deploy OpenTelemetry Collector

This collector will receive telemetry from:

- Istio Envoy sidecars using OTLP
- Manual Zipkin test spans
- Optional Jaeger-format clients

It will process telemetry using:

- memory limiter
- Kubernetes attributes
- resource enrichment
- tail sampling
- batching

Then it will export traces to Jaeger over OTLP.

---

## 12. Create RBAC for Kubernetes Metadata Enrichment

The `k8sattributes` processor needs permission to read Kubernetes metadata.

Create RBAC:

```bash
cat <<'EOF' > otel-collector-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
  namespace: otel-advanced
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-k8sattributes
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["replicasets", "deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-k8sattributes
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-k8sattributes
subjects:
- kind: ServiceAccount
  name: otel-collector
  namespace: otel-advanced
EOF
```

Apply:

```bash
kubectl apply -f otel-collector-rbac.yaml
```

Validate:

```bash
kubectl get sa otel-collector -n $OTEL_NS
kubectl get clusterrole otel-collector-k8sattributes
kubectl get clusterrolebinding otel-collector-k8sattributes
```

---

## 13. Create OpenTelemetry Collector Configuration

Create the configuration:

```bash
cat <<'EOF' > otel-collector-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: otel-advanced
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      zipkin:
        endpoint: 0.0.0.0:9411

      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_http:
            endpoint: 0.0.0.0:14268

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 100

      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        extract:
          metadata:
          - k8s.namespace.name
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.deployment.name
          - k8s.node.name

      resource:
        attributes:
        - key: deployment.environment
          value: ericsson-training
          action: upsert
        - key: cloud.provider
          value: azure
          action: upsert
        - key: k8s.cluster.name
          value: aks-ericsson-observability
          action: upsert

      tail_sampling:
        decision_wait: 10s
        num_traces: 50000
        expected_new_traces_per_sec: 100
        policies:
        - name: errors-always-sample
          type: status_code
          status_code:
            status_codes: [ERROR]
        - name: slow-traces-over-500ms
          type: latency
          latency:
            threshold_ms: 500
        - name: probabilistic-baseline
          type: probabilistic
          probabilistic:
            sampling_percentage: 20

      batch:
        timeout: 5s
        send_batch_size: 512
        send_batch_max_size: 1024

    exporters:
      otlp/jaeger:
        endpoint: jaeger-advanced.otel-advanced.svc.cluster.local:4317
        tls:
          insecure: true

      debug:
        verbosity: basic

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      zpages:
        endpoint: 0.0.0.0:55679

    service:
      extensions: [health_check, zpages]
      telemetry:
        logs:
          level: info
      pipelines:
        traces:
          receivers: [otlp, zipkin, jaeger]
          processors: [memory_limiter, k8sattributes, resource, tail_sampling, batch]
          exporters: [otlp/jaeger, debug]
EOF
```

Apply:

```bash
kubectl apply -f otel-collector-config.yaml
```

Validate:

```bash
kubectl get configmap otel-collector-config -n $OTEL_NS
```

---

## 14. Deploy OpenTelemetry Collector

Create collector deployment and service:

```bash
cat <<'EOF' > otel-collector-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: otel-advanced
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
      serviceAccountName: otel-collector
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.153.0
        imagePullPolicy: IfNotPresent
        args:
        - "--config=/conf/otel-collector-config.yaml"
        ports:
        - containerPort: 4317
          name: otlp-grpc
        - containerPort: 4318
          name: otlp-http
        - containerPort: 9411
          name: zipkin
        - containerPort: 14250
          name: jaeger-grpc
        - containerPort: 14268
          name: jaeger-http
        - containerPort: 13133
          name: health
        - containerPort: 55679
          name: zpages
        - containerPort: 8888
          name: metrics
        readinessProbe:
          httpGet:
            path: /
            port: 13133
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 13133
          initialDelaySeconds: 20
          periodSeconds: 20
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: otel-collector-config-vol
          mountPath: /conf
      volumes:
      - name: otel-collector-config-vol
        configMap:
          name: otel-collector-config
          items:
          - key: otel-collector-config.yaml
            path: otel-collector-config.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: otel-advanced
  labels:
    app: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
  - name: zipkin
    port: 9411
    targetPort: 9411
  - name: jaeger-grpc
    port: 14250
    targetPort: 14250
  - name: jaeger-http
    port: 14268
    targetPort: 14268
  - name: health
    port: 13133
    targetPort: 13133
  - name: zpages
    port: 55679
    targetPort: 55679
  - name: metrics
    port: 8888
    targetPort: 8888
EOF
```

Apply:

```bash
kubectl apply -f otel-collector-deployment.yaml
```

Validate rollout:

```bash
kubectl rollout status deployment/otel-collector -n $OTEL_NS --timeout=180s
kubectl get pods -n $OTEL_NS
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=100
```

Expected logs should show the collector starting successfully and receivers listening.

---

## 15. Validate Collector Health

Port-forward the health endpoint:

```bash
kubectl port-forward svc/otel-collector -n $OTEL_NS 13133:13133
```

In another terminal:

```bash
curl http://localhost:13133
```

Expected output:

```text
Server available
```

Stop the port-forward with `Ctrl+C`.

Validate collector self-metrics:

```bash
kubectl port-forward svc/otel-collector -n $OTEL_NS 8888:8888
```

In another terminal:

```bash
curl -s http://localhost:8888/metrics | head
```

Expected result:

You should see OpenTelemetry Collector internal metrics.

Stop the port-forward with `Ctrl+C`.

### What Is Happening?

The Collector is now ready to receive telemetry on multiple protocols:

```text
OTLP gRPC: 4317
OTLP HTTP: 4318
Zipkin:    9411
Jaeger:    14250 / 14268
```

This is useful in real environments because not all services are instrumented the same way.

Some services may use OpenTelemetry SDKs.  
Some legacy services may still send Jaeger or Zipkin traces.  
The Collector normalizes and forwards telemetry to the backend.

---

## 16. Test Pipeline with a Manual Zipkin Span

Before integrating Istio, validate the collector-to-Jaeger pipeline directly.

Open port-forward to the collector Zipkin receiver:

```bash
kubectl port-forward svc/otel-collector -n $OTEL_NS 9411:9411
```

In another terminal, send a manual Zipkin span:

```bash
export TS=$(($(date +%s%N)/1000))

curl -s -X POST "http://localhost:9411/api/v2/spans" \
  -H "Content-Type: application/json" \
  -d "[
    {
      \"traceId\": \"11111111111111111111111111111111\",
      \"id\": \"2222222222222222\",
      \"name\": \"manual-zipkin-to-otel-to-jaeger\",
      \"timestamp\": ${TS},
      \"duration\": 250000,
      \"localEndpoint\": {
        \"serviceName\": \"manual-zipkin-client\",
        \"ipv4\": \"127.0.0.1\"
      },
      \"tags\": {
        \"lab\": \"otel-jaeger-advanced\",
        \"source\": \"zipkin-receiver\"
      }
    }
  ]"
```

Expected result:

The command may return an empty response or HTTP 202/200 depending on the receiver behavior.

Open Jaeger UI:

```bash
kubectl port-forward svc/jaeger-advanced -n $OTEL_NS 16686:16686
```

Open:

```text
http://localhost:16686
```

In Jaeger:

1. Select service: `manual-zipkin-client`
2. Click **Find Traces**
3. Open the trace

Expected result:

You should see the manually generated span.

### What Is Happening?

This validates:

```text
Zipkin Span
   |
   v
OpenTelemetry Collector Zipkin Receiver
   |
   v
Tail Sampling Processor
   |
   v
OTLP Exporter
   |
   v
Jaeger
```

### Architect Discussion

Why is this useful?

Because enterprises often have mixed telemetry sources:

```text
New services        -> OpenTelemetry OTLP
Legacy services     -> Jaeger format
Older libraries     -> Zipkin format
Security tools      -> Different formats
Commercial agents   -> Vendor-specific integrations
```

The Collector provides a control plane for telemetry routing.

---

## 17. Configure Istio to Send Traces to OpenTelemetry Collector

Istio can send Envoy-generated traces to an OpenTelemetry provider.

First, check whether Istio already has an OpenTelemetry provider configured:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep -A20 "extensionProviders" || true
```

If you already see a provider named `otel-tracing`, review the service and port.

Expected provider for this lab:

```text
name: otel-tracing
service: otel-collector.otel-advanced.svc.cluster.local
port: 4317
```

---

## 18. Backup Current Istio Mesh Config

Before changing Istio configuration, create a backup:

```bash
kubectl get configmap istio -n istio-system -o yaml > backup-istio-configmap-before-otel-advanced.yaml
```

Validate backup exists:

```bash
ls -l backup-istio-configmap-before-otel-advanced.yaml
```

---

## 19. Add OpenTelemetry Extension Provider to Istio

### Recommended Method: Using istioctl

Validate `istioctl`:

```bash
istioctl version
```

Create Istio operator overlay:

```bash
cat <<'EOF' > istio-otel-extension-provider.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    extensionProviders:
    - name: otel-tracing
      opentelemetry:
        service: otel-collector.otel-advanced.svc.cluster.local
        port: 4317
EOF
```

Apply:

```bash
istioctl install -y -f istio-otel-extension-provider.yaml
```

Validate:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep -A20 "extensionProviders"
```

Expected result:

```yaml
extensionProviders:
- name: otel-tracing
  opentelemetry:
    service: otel-collector.otel-advanced.svc.cluster.local
    port: 4317
```

### If istioctl Is Not Available

Use this lab as a conceptual step and continue with the collector and Zipkin test.  
For production or controlled training environments, installing or patching Istio mesh config should be done with the same method used to install Istio originally.

Avoid blindly overwriting the Istio ConfigMap in production.

---

## 20. Configure Istio Telemetry API

Create mesh-wide tracing policy:

```bash
cat <<'EOF' > istio-telemetry-otel-tracing.yaml
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

Apply:

```bash
kubectl apply -f istio-telemetry-otel-tracing.yaml
```

Validate:

```bash
kubectl get telemetry -A
kubectl describe telemetry mesh-default-tracing -n istio-system
```

### What Is Happening?

The `Telemetry` resource tells Istio:

```text
Use provider: otel-tracing
Sample: 100% of requests
Send spans to: OpenTelemetry Collector
```

For a lab, 100% sampling is useful because students need to see traces quickly.

### Production Reality

100% tracing is usually too expensive at high volume.

Production systems often use:

- Lower head sampling
- Tail sampling for errors and slow requests
- Different sampling by service
- Different retention by business criticality

---

## 21. Restart Application Workloads

To make sure Envoy sidecars reload the updated tracing configuration, restart application deployments.

List deployments:

```bash
kubectl get deploy -n $APP_NS
```

Restart deployments:

```bash
kubectl rollout restart deployment -n $APP_NS
kubectl rollout status deployment -n $APP_NS --timeout=180s
```

Validate sidecars:

```bash
kubectl get pods -n $APP_NS
```

Optional check for Istio sidecar:

```bash
kubectl get pods -n $APP_NS -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

Expected result:

Pods that participate in the mesh should include:

```text
istio-proxy
```

If there is no `istio-proxy`, the namespace or workload may not have Istio sidecar injection enabled.

---

## 22. Generate Application Traffic

Generate normal traffic:

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w "Request $i -> HTTP %{http_code}\n" http://$GATEWAY_IP
  sleep 1
done
```

If JWT is required:

```bash
export VALID_TOKEN=$(cat ../valid-token.txt 2>/dev/null || cat valid-token.txt 2>/dev/null || true)

if [ -n "$VALID_TOKEN" ]; then
  for i in $(seq 1 30); do
    curl -s -o /dev/null -w "Request $i -> HTTP %{http_code}\n" \
      -H "Authorization: Bearer $VALID_TOKEN" \
      http://$GATEWAY_IP
    sleep 1
  done
else
  echo "No valid token found. Requests may return 403 if JWT policies are enabled."
fi
```

Validate collector logs:

```bash
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=100
```

Open Jaeger UI:

```bash
kubectl port-forward svc/jaeger-advanced -n $OTEL_NS 16686:16686
```

Open:

```text
http://localhost:16686
```

In Jaeger, look for services such as:

```text
istio-ingressgateway
frontend
product-service
order-service
makeline-service
```

The exact names depend on your Istio and workload metadata.

---

## 23. Analyze a Trace

In Jaeger:

1. Select a service.
2. Click **Find Traces**.
3. Open one trace.
4. Identify:
   - total duration
   - root span
   - child spans
   - service-to-service calls
   - errors
   - tags
   - process metadata

Discussion questions:

1. Which service received the request first?
2. Which service spent the most time?
3. Is this a complete trace or a partial trace?
4. Are all expected microservices visible?
5. Do you see HTTP status codes?
6. Do you see Kubernetes metadata?
7. Do you see any evidence of Istio sidecars?

### What Is Happening?

A trace is the full journey of a request.

A span is one operation inside that journey.

Example:

```text
Trace: checkout request
  |
  +-- Span: ingress gateway receives request
  |
  +-- Span: frontend calls product-service
  |
  +-- Span: frontend calls order-service
  |
  +-- Span: order-service publishes to RabbitMQ
```

### Important Nuance

Istio can generate proxy-level spans, but application-level spans require application instrumentation.

Without SDK instrumentation, you may not see:

```text
MongoDB query duration
Redis GET duration
Business validation time
Message serialization time
```

---

## 24. Context Propagation Experiment

Generate a manual `traceparent` header.

```bash
export TRACE_ID="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
export PARENT_SPAN_ID="bbbbbbbbbbbbbbbb"
export TRACEPARENT="00-${TRACE_ID}-${PARENT_SPAN_ID}-01"

echo $TRACEPARENT
```

Send traffic with the header:

```bash
curl -i \
  -H "traceparent: $TRACEPARENT" \
  http://$GATEWAY_IP
```

If JWT is required:

```bash
if [ -n "$VALID_TOKEN" ]; then
  curl -i \
    -H "Authorization: Bearer $VALID_TOKEN" \
    -H "traceparent: $TRACEPARENT" \
    http://$GATEWAY_IP
fi
```

Check Jaeger.

Discussion questions:

1. Can you find the trace?
2. Did the trace ID remain the same?
3. Did the request create a new trace instead?
4. Are all services connected in a single trace?
5. What could break context propagation?

### Common Causes of Broken Context Propagation

- Application does not forward tracing headers.
- Gateway strips headers.
- Service mesh and application SDK use incompatible propagators.
- Asynchronous messaging loses context.
- Custom HTTP clients do not propagate headers.
- Sampling decisions are inconsistent across services.

---

## 25. Baggage Conceptual Demo

`baggage` carries contextual key-value pairs across services.

Example header:

```text
baggage: tenant.id=ericsson,customer.tier=gold
```

Send request:

```bash
curl -i \
  -H "baggage: tenant.id=ericsson,customer.tier=gold" \
  http://$GATEWAY_IP
```

If JWT is required:

```bash
if [ -n "$VALID_TOKEN" ]; then
  curl -i \
    -H "Authorization: Bearer $VALID_TOKEN" \
    -H "baggage: tenant.id=ericsson,customer.tier=gold" \
    http://$GATEWAY_IP
fi
```

### Discussion

Baggage is powerful but dangerous.

It can be useful for:

- tenant correlation
- region
- feature flag
- test scenario
- business context

It should not carry:

- passwords
- tokens
- personal data
- credit card data
- sensitive business secrets

### Production Reality

Baggage can increase payload size and may expose sensitive metadata if misused.

Governance is required.

---

## 26. Incident Lab 1 – Payment or Checkout Latency Simulation

If your application has a service that can be scaled down or stressed, use it.  
If not, simulate partial degradation by scaling one backend deployment to zero.

Example using `product-service`:

```bash
kubectl scale deployment product-service --replicas=0 -n $APP_NS
```

Generate traffic:

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "Request $i -> HTTP %{http_code}\n" http://$GATEWAY_IP
  sleep 1
done
```

Observe in Jaeger:

- Do traces still appear?
- Which service shows errors?
- Is the error at ingress, frontend or backend?
- Are traces shorter or longer?
- Are spans missing?

Restore:

```bash
kubectl scale deployment product-service --replicas=1 -n $APP_NS
kubectl rollout status deployment/product-service -n $APP_NS --timeout=180s
```

### What Is Happening?

When a downstream service is unavailable, the frontend or gateway may return:

```text
503
500
timeout
connection refused
no healthy upstream
```

Distributed tracing helps identify where the request failed.

---

## 27. Incident Lab 2 – RabbitMQ Down and Async Tracing Limitation

Scale RabbitMQ down:

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $APP_NS
```

Generate order-related traffic if your frontend supports it.  
Otherwise, generate normal traffic and inspect logs:

```bash
kubectl logs deployment/order-service -n $APP_NS --tail=100
```

Observe traces in Jaeger.

Discussion questions:

1. Do traces clearly show RabbitMQ failure?
2. Are asynchronous operations represented as child spans?
3. Does the trace continue after a message is published?
4. What would be needed to trace async workflows correctly?

Restore RabbitMQ:

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $APP_NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $APP_NS --timeout=180s
```

### Production Reality

Tracing asynchronous messaging requires application instrumentation.

For example:

```text
HTTP request span
  |
  +-- publish message span
       |
       +-- consumer processing span
```

To connect producer and consumer spans, the application must propagate trace context through message headers.

Service mesh alone usually cannot fully reconstruct asynchronous business workflows.

---

## 28. Sampling Deep Dive

The collector currently uses tail sampling:

```yaml
tail_sampling:
  decision_wait: 10s
  policies:
  - errors-always-sample
  - slow-traces-over-500ms
  - probabilistic-baseline
```

### Head Sampling

Decision happens at the beginning of the request.

Advantage:

```text
Low overhead
Simple
```

Risk:

```text
May drop important errors before knowing they are errors
```

### Tail Sampling

Decision happens after the trace is observed.

Advantage:

```text
Can keep errors and slow traces
```

Risk:

```text
Requires buffering
Higher memory usage
More operational complexity
```

### Probabilistic Sampling

Keeps a percentage of traces.

Example:

```text
Keep 20% of traces
Drop 80%
```

Good for high-volume services.

Bad if rare errors are dropped.

---

## 29. Experiment – Reduce Sampling

Edit collector config:

```bash
kubectl edit configmap otel-collector-config -n $OTEL_NS
```

Change:

```yaml
sampling_percentage: 20
```

To:

```yaml
sampling_percentage: 5
```

Restart collector:

```bash
kubectl rollout restart deployment/otel-collector -n $OTEL_NS
kubectl rollout status deployment/otel-collector -n $OTEL_NS --timeout=180s
```

Generate traffic:

```bash
for i in $(seq 1 50); do
  curl -s -o /dev/null http://$GATEWAY_IP
done
```

Observe Jaeger.

Discussion questions:

1. Are fewer traces visible?
2. Are errors still retained?
3. How would you choose the right sampling percentage?
4. Would checkout and payment use the same sampling as product catalog browsing?

### Architect Discussion

Not all services deserve the same sampling strategy.

Example:

```text
Product catalog browsing: lower sampling
Checkout: higher sampling
Payment: errors always sampled
Authentication: high-value security traces
Internal health checks: usually dropped
```

---

## 30. Collector Troubleshooting Commands

Check collector status:

```bash
kubectl get pods -n $OTEL_NS
kubectl describe pod -n $OTEL_NS -l app=otel-collector
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=200
```

Check Jaeger:

```bash
kubectl get pods -n $OTEL_NS -l app=jaeger-advanced
kubectl logs deployment/jaeger-advanced -n $OTEL_NS --tail=100
```

Check services:

```bash
kubectl get svc -n $OTEL_NS
```

Check collector endpoints from inside the cluster:

```bash
kubectl run otel-test-client \
  -n $OTEL_NS \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl:8.10.1 \
  -- sh
```

Inside the pod:

```bash
curl http://otel-collector:13133
curl http://jaeger-advanced:16686
exit
```

---

## 31. Istio Tracing Troubleshooting

Check Telemetry resources:

```bash
kubectl get telemetry -A
kubectl describe telemetry mesh-default-tracing -n istio-system
```

Check Istio config:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep -A20 "extensionProviders"
```

Check app sidecars:

```bash
kubectl get pods -n $APP_NS -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

Check Istio proxy logs for one pod:

```bash
export ONE_POD=$(kubectl get pod -n $APP_NS -o jsonpath='{.items[0].metadata.name}')
echo $ONE_POD

kubectl logs $ONE_POD -n $APP_NS -c istio-proxy --tail=100
```

If there is no `istio-proxy` container, the workload is not part of the sidecar mesh.

---

## 32. Common Problems and Fixes

### Problem 1: No traces in Jaeger

Check:

```bash
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=200
kubectl logs deployment/jaeger-advanced -n $OTEL_NS --tail=200
kubectl get telemetry -A
```

Possible causes:

- No traffic generated
- Istio provider not configured
- Telemetry resource references wrong provider
- Collector cannot reach Jaeger
- Sampling dropped traces
- App pods do not have sidecars

---

### Problem 2: Manual Zipkin span appears, but Istio traces do not

This means:

```text
Collector -> Jaeger works
Istio -> Collector may be broken
```

Check:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep -A20 "extensionProviders"
kubectl describe telemetry mesh-default-tracing -n istio-system
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=100
```

---

### Problem 3: Traces are fragmented

Symptoms:

```text
You see multiple separate traces for what should be one request.
```

Possible causes:

- Application does not forward trace headers
- Async messaging loses context
- Different propagators
- Proxy and SDK both create independent traces
- Sampling mismatch

---

### Problem 4: Collector pod crashes

Check:

```bash
kubectl describe pod -n $OTEL_NS -l app=otel-collector
kubectl logs deployment/otel-collector -n $OTEL_NS
```

Common causes:

- YAML indentation error
- Unsupported processor configuration
- Port conflict
- Memory limit too low

Rollback:

```bash
kubectl rollout undo deployment/otel-collector -n $OTEL_NS
```

Or re-apply the known-good files:

```bash
kubectl apply -f otel-collector-config.yaml
kubectl apply -f otel-collector-deployment.yaml
```

---

## 33. Advanced Discussion – SDK vs Service Mesh Instrumentation

### Service Mesh Instrumentation

Pros:

- No code changes
- Fast to deploy
- Good for network-level visibility
- Useful for service-to-service HTTP/gRPC traffic

Cons:

- Limited business context
- Limited internal operation visibility
- Weak for async messaging
- May not show database calls clearly

### SDK Instrumentation

Pros:

- Full business context
- Internal spans
- Database spans
- Messaging spans
- Custom attributes
- Better root cause analysis

Cons:

- Requires code changes or auto-instrumentation
- Requires language-specific knowledge
- Requires governance
- May introduce overhead if poorly configured

### Best Practice

Use both.

```text
Istio / Envoy tracing
  +
OpenTelemetry SDK instrumentation
  +
OpenTelemetry Collector
  +
Tracing backend
```

---

## 34. Recommended Trace Attributes

Useful span attributes:

```text
service.name
service.version
deployment.environment
http.method
http.route
http.status_code
db.system
db.statement
messaging.system
messaging.destination.name
enduser.id
tenant.id
cloud.provider
k8s.namespace.name
k8s.pod.name
```

Avoid sensitive attributes:

```text
password
access_token
refresh_token
credit_card_number
national_id
personal health information
private keys
connection strings
```

### Governance Discussion

Observability data can become sensitive data.

Traces may contain:

- URLs
- user identifiers
- payload fragments
- query strings
- tenant names
- IP addresses
- internal hostnames

Therefore, observability requires security, privacy and retention governance.

---

## 35. Production Architecture Options

### Option 1: Simple Lab Architecture

```text
App / Istio
   |
   v
OpenTelemetry Collector
   |
   v
Jaeger All-in-One
```

Good for:

- training
- proof of concept
- demos

Not good for:

- production
- high volume
- long retention

---

### Option 2: Production Gateway Collector

```text
Applications
   |
   v
Node-level Collectors / Agents
   |
   v
Gateway Collectors
   |
   v
Tracing Backend
```

Good for:

- centralized policy
- sampling
- enrichment
- routing
- vendor neutrality

---

### Option 3: Multi-Backend Export

```text
OpenTelemetry Collector
   |
   +--> Jaeger
   |
   +--> Azure Monitor / Application Insights
   |
   +--> Elastic / APM
   |
   +--> Vendor SIEM
```

Good for:

- enterprise integration
- migration scenarios
- platform engineering

Risk:

- cost explosion
- duplicate data
- inconsistent retention
- governance complexity

---

## 36. Production Best Practices

### Collector Deployment

Use:

- multiple replicas
- resource limits
- health checks
- horizontal scaling
- separate agent and gateway patterns
- config versioning
- GitOps

### Sampling

Use:

- tail sampling for important transactions
- error-always-sample
- latency-based sampling
- lower sampling for high-volume low-value traffic
- higher sampling for checkout, payment and authentication

### Security

Use:

- TLS for OTLP
- authentication where supported
- network policies
- restricted namespaces
- RBAC
- data redaction
- attribute filtering

### Cost Control

Control:

- sampling percentage
- retention period
- indexed attributes
- duplicate export
- high-cardinality labels
- noisy health checks

### Governance

Define:

- telemetry ownership
- retention policy
- sensitive data policy
- naming standards
- service metadata standards
- incident runbooks
- dashboard standards

---

## 37. Student Challenge – Diagnose a Broken Trace

Instructor can trigger one of the following conditions:

### Option A: Scale down a backend service

```bash
kubectl scale deployment product-service --replicas=0 -n $APP_NS
```

### Option B: Break RabbitMQ

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $APP_NS
```

### Option C: Reduce sampling

```bash
kubectl edit configmap otel-collector-config -n $OTEL_NS
kubectl rollout restart deployment/otel-collector -n $OTEL_NS
```

### Option D: Delete telemetry resource

```bash
kubectl delete telemetry mesh-default-tracing -n istio-system
```

Students must answer:

```text
Which symptom did users experience?
Which service failed?
Which traces exist?
Which traces are missing?
Which logs support the conclusion?
Which metric would detect the issue earlier?
Is this a technical failure or business degradation?
What recovery action is required?
What preventive alert should exist?
What architecture improvement would reduce recurrence?
```

---

## 38. Create Diagnosis Template

Create a notes folder:

```bash
mkdir -p notes
```

Create diagnosis template:

```bash
cat <<'EOF' > notes/otel-jaeger-diagnosis-template.md
# OpenTelemetry + Jaeger Diagnosis

## Team Name
-

## Incident Scenario
-

## User Symptom
-

## Trace Evidence
-

## Missing Spans or Broken Propagation
-

## Collector Evidence
-

## Jaeger Evidence
-

## Kubernetes Evidence
-

## Istio Evidence
-

## Logs Evidence
-

## Metrics Evidence
-

## Root Cause
-

## Business Impact
-

## Recovery Action
-

## Preventive Alert
-

## Architecture Improvement
-
EOF
```

---

## 39. Minimal Recovery Runbook

Create a recovery runbook:

```bash
cat <<'EOF' > notes/otel-jaeger-recovery-runbook.md
# Recovery Runbook – OpenTelemetry + Jaeger Advanced Lab

## Collector not receiving traces

Detect:
- No new traces in Jaeger
- Collector logs show no received spans
- Istio configured but no telemetry exported

Confirm:
kubectl logs deployment/otel-collector -n $OTEL_NS --tail=200
kubectl get telemetry -A
kubectl get configmap istio -n istio-system -o yaml | grep -A20 extensionProviders

Recover:
kubectl rollout restart deployment/otel-collector -n $OTEL_NS
kubectl rollout restart deployment -n $APP_NS

Validate:
Generate traffic and check Jaeger UI.

---

## Jaeger UI unavailable

Detect:
- Browser cannot open http://localhost:16686
- Port-forward fails

Confirm:
kubectl get pods -n $OTEL_NS -l app=jaeger-advanced
kubectl logs deployment/jaeger-advanced -n $OTEL_NS --tail=100
kubectl get svc jaeger-advanced -n $OTEL_NS

Recover:
kubectl rollout restart deployment/jaeger-advanced -n $OTEL_NS

Validate:
kubectl port-forward svc/jaeger-advanced -n $OTEL_NS 16686:16686

---

## Manual Zipkin span works but Istio traces do not

Detect:
- service manual-zipkin-client appears in Jaeger
- application services do not appear

Confirm:
kubectl get telemetry -A
kubectl get configmap istio -n istio-system -o yaml | grep -A20 extensionProviders
kubectl get pods -n $APP_NS -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'

Recover:
Confirm otel-tracing provider exists.
Confirm workloads have istio-proxy.
Restart application deployments.

Validate:
Generate traffic and check Jaeger.

---

## Traces are incomplete

Detect:
- only ingress spans visible
- backend services missing
- multiple disconnected traces

Confirm:
Check whether application forwards traceparent headers.
Check whether asynchronous messages propagate trace context.
Check Istio sidecars on all workloads.

Recover:
Use OpenTelemetry SDK instrumentation.
Enable propagation in HTTP clients and messaging clients.

Validate:
A single user transaction should produce a connected trace.
EOF
```

---

## 40. Cleanup

### Restore Application Components

```bash
kubectl scale deployment product-service --replicas=1 -n $APP_NS 2>/dev/null || true
kubectl scale statefulset rabbitmq --replicas=1 -n $APP_NS 2>/dev/null || true
kubectl scale statefulset mongodb --replicas=1 -n $APP_NS 2>/dev/null || true

kubectl get pods -n $APP_NS
```

### Delete Lab Telemetry Resource

If this lab created the telemetry resource and you want to remove it:

```bash
kubectl delete telemetry mesh-default-tracing -n istio-system --ignore-not-found
```

### Delete Advanced OTel / Jaeger Stack

```bash
kubectl delete namespace $OTEL_NS
```

### Optional: Restore Istio ConfigMap Backup

Only do this if you are sure the backup belongs to the current cluster and represents the desired previous state.

```bash
kubectl apply -f backup-istio-configmap-before-otel-advanced.yaml
kubectl rollout restart deployment/istiod -n istio-system
kubectl rollout status deployment/istiod -n istio-system --timeout=180s
```

Then restart app workloads:

```bash
kubectl rollout restart deployment -n $APP_NS
kubectl rollout status deployment -n $APP_NS --timeout=180s
```

---

## 41. Final Knowledge Check

Ask students:

1. What is the difference between a trace and a span?
2. What is the role of the OpenTelemetry Collector?
3. What is the difference between a receiver, processor and exporter?
4. Why did we use Jaeger?
5. Why did we test the Zipkin receiver?
6. What is the difference between head sampling and tail sampling?
7. Why is 100% sampling usually not recommended in production?
8. Why can service mesh tracing be insufficient for async workflows?
9. What causes broken context propagation?
10. What sensitive data could accidentally appear in traces?
11. What should be instrumented manually?
12. Which business transactions deserve higher sampling?
13. How would you design this for production?
14. How would you control cost?
15. How would you govern telemetry data?

---

## 42. Instructor Summary

By the end of this lab, students should understand that distributed tracing is not only a visualization tool.

It is an architectural capability.

It helps teams:

- reconstruct request flows
- identify slow dependencies
- detect broken service interactions
- distinguish infrastructure failures from application failures
- understand degraded business behavior
- improve incident response
- design better microservices

Final message for students:

> Building microservices is only half the work. The other half is understanding them when they fail.

---

## 43. Final Architecture Recap

```text
User
 |
 v
Istio Ingress Gateway
 |
 v
Microservices on AKS
 |
 v
Istio Envoy Sidecars
 |
 v
OpenTelemetry Collector
 |   \
 |    \__ Debug Exporter
 |
 v
Jaeger Advanced
 |
 v
Jaeger UI
```

### Main Takeaways

```text
Logs answer:    What happened?
Metrics answer: Is it healthy or degraded?
Traces answer:  Where did the request go and where did it fail?
```

Real observability requires all three:

```text
Metrics + Logs + Traces + Business Context
```

---

## 44. Recommended Next Step

As a follow-up lab, instrument one application service with the OpenTelemetry SDK.

Suggested service:

```text
order-service
```

Suggested spans:

```text
POST /orders
validate order
read product
reserve inventory
publish RabbitMQ message
write MongoDB document
```

Suggested custom attributes:

```text
order.id
customer.id_hash
cart.item_count
payment.method
business.operation
```

The key improvement would be moving from proxy-level traces to business-aware traces.

