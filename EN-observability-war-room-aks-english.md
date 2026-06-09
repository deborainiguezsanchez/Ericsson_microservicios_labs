# Observability War Room on AKS
## Elastic Cloud + Azure Managed Prometheus + Azure Managed Grafana

> **Course:** Advanced Microservices – Ericsson  
> **Duration:** 2 days × 6 hours = 12 hours  
> **Environment:** Azure Cloud Shell + AKS + AKS Store Demo + RabbitMQ + MongoDB + Redis + Istio  
> **Base repository:** https://github.com/nestorreveron/Ericsson_microservicios_labs

---

## 0. Lab Purpose

This lab turns the microservices environment we have already built into a **real operations scenario under failure**.

The goal is not to install tools just for the sake of installing tools. The goal is for students to learn how to answer real operational questions:

- Which service failed?
- Where is the backlog accumulating?
- Which pod restarted?
- Which component is responding slowly?
- Which logs explain the error?
- What evidence do I have in metrics?
- What evidence do I have in logs?
- What business impact does the failure have?
- How do I differentiate a real outage from an expected security block?
- How do I diagnose a distributed incident?

---

## 1. Target Architecture

```text
AKS Store Demo on AKS
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

## 2. Lab Tools

| Tool | Main Use in the Lab |
|---|---|
| Elastic Cloud | SaaS platform for logs, metrics, and observability |
| Kibana | Log search, exploration, and dashboards |
| Elastic Agent | Agent used to collect logs and metrics from Kubernetes |
| Fleet | Centralized management for Elastic Agents |
| Azure Managed Prometheus | Azure-managed Prometheus metrics |
| Azure Managed Grafana | Dashboards, visualization, and analysis |
| AKS | Kubernetes platform where the microservices run |
| RabbitMQ | Message broker |
| MongoDB | Demo persistence layer |
| Redis | Distributed cache |
| Istio/Envoy | Service mesh, mTLS, JWT, and Zero Trust security |

---

## 3. Two-Day Distribution

### Day 1 – Elastic Cloud + Logs + Kubernetes Incidents

| Block | Duration | Topic |
|---|---:|---|
| 1 | 30 min | Architecture and objectives review |
| 2 | 45 min | Repository preparation and AKS validation |
| 3 | 60 min | Create Elastic Cloud trial and configure Kibana/Fleet |
| 4 | 75 min | Install Elastic Agent on AKS |
| 5 | 60 min | Search microservices logs in Kibana |
| 6 | 60 min | Incident 1: MongoDB down |
| 7 | 45 min | Incident 2: makeline-service down |
| 8 | 45 min | Discussion: logs, evidence, and runbook |

### Day 2 – Azure Managed Prometheus + Grafana + War Room

| Block | Duration | Topic |
|---|---:|---|
| 1 | 45 min | Enable Azure Managed Prometheus |
| 2 | 60 min | Create/connect Azure Managed Grafana |
| 3 | 60 min | Operational dashboards for AKS |
| 4 | 45 min | Metrics for RabbitMQ, MongoDB, Redis, and Istio |
| 5 | 75 min | Guided incidents: RabbitMQ, MongoDB, Redis, Istio |
| 6 | 60 min | Final team challenge: diagnosis in 15 minutes |
| 7 | 45 min | Alert design, SLO/SLI, and best practices |
| 8 | 30 min | Cleanup and closing |

---

# Day 1 – Elastic Cloud + Logs + Incidents

---

## 4. Initial Environment Preparation

### 4.1 Open Azure Cloud Shell

Use **Bash** in Azure Cloud Shell.

Validate the session:

```bash
az account show -o table
```

---

### 4.2 Define Variables

Adjust these values to your real environment:

```bash
export RG_NAME="<YOUR_RESOURCE_GROUP>"
export AKS_NAME="<YOUR_AKS_NAME>"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"
```

Example:

```bash
export RG_NAME="rg-microservices-lab"
export AKS_NAME="aks-microservices-lab"
export NS="aks-store-state-lab"
export LOCATION="$(az group show -n $RG_NAME --query location -o tsv)"
```

Validate:

```bash
echo $RG_NAME
echo $AKS_NAME
echo $NS
echo $LOCATION
```

---

### 4.3 Connect to AKS

```bash
az aks get-credentials   --resource-group $RG_NAME   --name $AKS_NAME   --overwrite-existing
```

Validate:

```bash
kubectl get nodes
kubectl get ns
```

---

## 5. Prepare the Repository for GitHub

### 5.1 Clone or Enter the Repository

```bash
cd ~/clouddrive
git clone https://github.com/nestorreveron/Ericsson_microservicios_labs.git
cd Ericsson_microservicios_labs
```

If it already exists:

```bash
cd ~/clouddrive/Ericsson_microservicios_labs
git pull
```

---

### 5.2 Create the Lab Structure

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

Expected structure:

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

### 5.3 Create a Variables File

```bash
cat <<EOF > scripts/env.sh
export RG_NAME="$RG_NAME"
export AKS_NAME="$AKS_NAME"
export NS="$NS"
export LOCATION="$LOCATION"
EOF
```

Use it when opening a new terminal:

```bash
source scripts/env.sh
```

---

## 6. Validate AKS Store Demo

### 6.1 View Pods

```bash
kubectl get pods -n $NS -o wide
```

### 6.2 View Services

```bash
kubectl get svc -n $NS
```

### 6.3 View Deployments and StatefulSets

```bash
kubectl get deploy -n $NS
kubectl get statefulset -n $NS
```

### 6.4 View Expected Components

```bash
kubectl get pods -n $NS | grep -E "store|order|product|makeline|rabbitmq|mongodb|redis"
```

---

## 7. View Logs Manually Before Elastic

Before installing Elastic, show the limitation of the manual approach:

```bash
kubectl logs deploy/order-service -n $NS --tail=50
kubectl logs deploy/makeline-service -n $NS --tail=50
kubectl logs deploy/product-service -n $NS --tail=50
```

### Question for Students

> What happens if I have 50 microservices, 300 pods, and a distributed incident?

Key message:

> `kubectl logs` is useful for basic diagnosis, but it does not scale as an operational strategy.

---

# 8. Create Elastic Cloud Trial

> Note: Elastic Cloud currently offers a **14-day free trial**. The goal here is to use it as a temporary SaaS platform for the final part of the course.

### 8.1 Create an Account

1. Go to: https://www.elastic.co/cloud/cloud-trial-overview
2. Create an account or sign in.
3. Create an Elastic Cloud deployment.
4. Choose a nearby or available region.
5. Wait until the deployment is ready.
6. Open **Kibana**.

---

## 9. Configure Fleet and Elastic Agent

### 9.1 Open Fleet

In Kibana:

```text
Management
  → Fleet
```

If Fleet asks for initial setup, accept it.

---

### 9.2 Create an Agent Policy

Create a policy named:

```text
aks-store-observability-policy
```

Suggested description:

```text
Elastic Agent policy for AKS Store Demo observability lab.
```

---

### 9.3 Add Kubernetes Integration

In Kibana:

```text
Integrations
  → Search: Kubernetes
  → Add Kubernetes
```

Configure the integration to collect:

- Kubernetes container logs
- Kubernetes pod metrics
- Kubernetes node metrics
- Kubernetes events
- Kubernetes metadata

Save the integration inside the policy:

```text
aks-store-observability-policy
```

---

### 9.4 Add Elastic Agent in Kubernetes

In Fleet:

```text
Agents
  → Add agent
  → Run Elastic Agent on Kubernetes
```

Select the policy:

```text
aks-store-observability-policy
```

Kibana/Fleet will generate a manifest or commands with:

- Fleet URL
- Enrollment token
- Elastic Agent image
- Kubernetes DaemonSet
- Required RBAC

Save the generated manifest as:

```bash
elastic/elastic-agent-managed-kubernetes.yaml
```

---

### 9.5 Apply the Elastic Agent Manifest

From Cloud Shell:

```bash
kubectl apply -f elastic/elastic-agent-managed-kubernetes.yaml
```

Validate:

```bash
kubectl get pods -n kube-system | grep elastic
kubectl get daemonset -n kube-system | grep elastic
```

Alternative command:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=elastic-agent
```

---

### 9.6 Validate in Kibana

In Kibana:

```text
Management
  → Fleet
  → Agents
```

Expected result:

```text
Elastic Agent: Healthy
```

---

## 10. Explore Logs in Kibana

### 10.1 Go to Discover

In Kibana:

```text
Analytics
  → Discover
```

Look for data views similar to:

```text
logs-*
metrics-*
```

---

### 10.2 Useful Queries in Kibana

Search logs for the namespace:

```text
kubernetes.namespace : "aks-store-state-lab"
```

Search logs for makeline-service:

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "makeline-service"
```

Search logs for MongoDB:

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "mongodb"
```

Search for errors:

```text
kubernetes.namespace : "aks-store-state-lab" and message : "*error*"
```

Search for connection refused:

```text
message : "*connection refused*"
```

Search for timeouts:

```text
message : "*timeout*" or message : "*deadline exceeded*"
```

---

## 11. Incident 1 – MongoDB Down

### 11.1 Scenario

MongoDB is the main persistence layer for the demo. If MongoDB goes down:

- product-service may fail when reading products
- makeline-service may fail when saving orders
- store-admin may fail when listing or managing data
- some pods may remain Running even though the business is degraded

---

### 11.2 Trigger the Failure

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
```

Validate:

```bash
kubectl get pods -n $NS
kubectl get statefulset mongodb -n $NS
```

---

### 11.3 Observe Logs with kubectl

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/store-admin -n $NS --tail=100
```

Possible expected errors:

```text
connection refused
server selection timeout
Failed to insert order
Failed to save orders to database
context deadline exceeded
```

---

### 11.4 Search in Kibana

Use queries:

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

### 11.5 Discussion

Questions for students:

1. Is the application completely down?
2. Which pods are still Running?
3. Which operations are failing?
4. Which operations could still work?
5. What evidence appears in logs?
6. Which metric would be useful for alerting?
7. Which pattern would help the system degrade better? Cache, circuit breaker, fallback, retry?

Key message:

> In microservices, “pod Running” does not mean “business healthy”.

---

### 11.6 Restore MongoDB

```bash
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=mongodb -n $NS --timeout=180s
kubectl get pods -n $NS
```

If MongoDB remains in CrashLoopBackOff:

```bash
kubectl logs mongodb-0 -n $NS --previous
kubectl describe pod mongodb-0 -n $NS
kubectl delete pod mongodb-0 -n $NS
```

---

## 12. Incident 2 – makeline-service Down

### 12.1 Scenario

`makeline-service` consumes orders from RabbitMQ and processes them. When it is stopped:

- order-service can keep generating messages
- RabbitMQ accumulates messages
- the system remains partially available
- processing is delayed

---

### 12.2 Trigger the Failure

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
```

Validate:

```bash
kubectl get pods -n $NS
kubectl get deploy makeline-service -n $NS
```

---

### 12.3 Observe RabbitMQ UI

If RabbitMQ is exposed as LoadBalancer:

```bash
kubectl get svc rabbitmq -n $NS
```

If it is not exposed:

```bash
kubectl port-forward svc/rabbitmq 15672:15672 -n $NS
```

Open:

```text
http://localhost:15672
```

Typical lab credentials:

```text
username: username
password: password
```

Go to:

```text
Queues and Streams → orders
```

Observe:

- Ready
- Unacked
- Total
- Consumers

---

### 12.4 Search Logs in Kibana

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "order-service"
```

```text
kubernetes.namespace : "aks-store-state-lab" and kubernetes.container.name : "virtual-customer"
```

---

### 12.5 CAP Discussion

Question:

> Does this favor availability or strong consistency?

Expected answer:

- It favors availability.
- It accepts temporary inconsistency.
- It is closer to AP behavior.
- The system does not block the entire platform because one consumer is down.

Key message:

> RabbitMQ enables temporal decoupling, but introduces backlog, eventual latency, and idempotency as real concerns.

---

### 12.6 Restore makeline-service

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=makeline-service -n $NS --timeout=180s
```

View logs:

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
```

---

## 13. Day 1 Closing

### 13.1 Evidence Students Must Save

Create file:

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

# Day 2 – Azure Managed Prometheus + Azure Managed Grafana

---

## 14. Day 2 Objective

On Day 1, we saw centralized logs with Elastic.

On Day 2, we will complement that with:

- metrics
- dashboards
- operational visualization
- conceptual alerts
- symptom-based diagnosis
- final team war room

Key message:

> Logs explain what happened. Metrics show the state and trend of the system.

---

## 15. Enable Azure Managed Prometheus

### 15.1 Register Providers if Needed

```bash
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.Dashboard
az provider register --namespace Microsoft.Insights
```

Validate:

```bash
az provider show --namespace Microsoft.Monitor --query registrationState -o tsv
az provider show --namespace Microsoft.Dashboard --query registrationState -o tsv
az provider show --namespace Microsoft.Insights --query registrationState -o tsv
```

---

### 15.2 Create Azure Monitor Workspace

```bash
export AMW_NAME="amw-ericsson-observability"
```

```bash
az monitor account create   --name $AMW_NAME   --resource-group $RG_NAME   --location $LOCATION
```

Get ID:

```bash
export AMW_ID=$(az monitor account show   --name $AMW_NAME   --resource-group $RG_NAME   --query id -o tsv)

echo $AMW_ID
```

---

### 15.3 Create Azure Managed Grafana

The name must be globally unique within Azure.

```bash
export GRAFANA_NAME="grafana-ericsson-$RANDOM"
```

```bash
az grafana create   --name $GRAFANA_NAME   --resource-group $RG_NAME   --location $LOCATION
```

Get ID:

```bash
export GRAFANA_ID=$(az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query id -o tsv)

echo $GRAFANA_ID
```

Get endpoint:

```bash
az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query properties.endpoint -o tsv
```

---

### 15.4 Assign Grafana Access Permissions

Get current user:

```bash
export USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
echo $USER_OBJECT_ID
```

Assign Grafana Admin role:

```bash
az role assignment create   --assignee $USER_OBJECT_ID   --role "Grafana Admin"   --scope $GRAFANA_ID
```

If this command fails due to tenant permissions, do it from Azure Portal:

```text
Azure Managed Grafana
  → Access control (IAM)
  → Add role assignment
  → Grafana Admin
  → Select user
```

---

### 15.5 Enable Prometheus on AKS and Link Grafana

```bash
az aks update   --name $AKS_NAME   --resource-group $RG_NAME   --enable-azure-monitor-metrics   --azure-monitor-workspace-resource-id $AMW_ID   --grafana-resource-id $GRAFANA_ID
```

Validate:

```bash
kubectl get pods -n kube-system | grep -E "ama-metrics|azuremonitor|prometheus"
```

Also check:

```bash
kubectl get pods -n kube-system
```

---

## 16. Explore Metrics in Azure Managed Grafana

### 16.1 Open Grafana

```bash
az grafana show   --name $GRAFANA_NAME   --resource-group $RG_NAME   --query properties.endpoint -o tsv
```

Open the endpoint in a browser.

---

### 16.2 Confirm Data Source

In Grafana:

```text
Connections
  → Data sources
```

Look for a data source similar to:

```text
Azure Monitor managed service for Prometheus
```

or:

```text
Prometheus_<Azure Monitor workspace endpoint>
```

---

### 16.3 Review Predefined Dashboards

Look for dashboards related to:

- Kubernetes
- AKS
- Node
- Pod
- Namespace
- Workload

Create or duplicate a dashboard for the lab.

---

## 17. Basic PromQL for Class

> Exact metrics can vary depending on configuration. Use these queries as a guide and adjust them in Grafana/Prometheus Explorer.

### 17.1 Pods by Namespace

```promql
sum by (namespace) (kube_pod_status_phase)
```

### 17.2 Pods in the Lab Namespace

```promql
sum by (pod, phase) (kube_pod_status_phase{namespace="aks-store-state-lab"})
```

### 17.3 Restarts by Pod

```promql
sum by (pod) (kube_pod_container_status_restarts_total{namespace="aks-store-state-lab"})
```

### 17.4 Unavailable Pods

```promql
sum by (deployment) (kube_deployment_status_replicas_unavailable{namespace="aks-store-state-lab"})
```

### 17.5 CPU by Container

```promql
sum by (pod, container) (
  rate(container_cpu_usage_seconds_total{namespace="aks-store-state-lab"}[5m])
)
```

### 17.6 Memory by Container

```promql
sum by (pod, container) (
  container_memory_working_set_bytes{namespace="aks-store-state-lab"}
)
```

### 17.7 Restarting Pods

```promql
increase(kube_pod_container_status_restarts_total{namespace="aks-store-state-lab"}[15m])
```

---

## 18. Create a Minimal Dashboard in Grafana

Create a dashboard named:

```text
AKS Store Demo – Observability War Room
```

Minimum panels:

| Panel | Query or Source |
|---|---|
| Pods by state | `kube_pod_status_phase` |
| Restarts by pod | `kube_pod_container_status_restarts_total` |
| CPU by pod | `container_cpu_usage_seconds_total` |
| Memory by pod | `container_memory_working_set_bytes` |
| Unavailable deployments | `kube_deployment_status_replicas_unavailable` |
| RabbitMQ backlog | RabbitMQ UI or Prometheus metrics if the plugin is enabled |
| Recent incidents | Kibana/Elastic |
| Security 401/403 | Istio/Envoy logs if collected |

---

## 19. Optional Advanced – Expose RabbitMQ Prometheus Metrics

> This section depends on the RabbitMQ image and configuration. The RabbitMQ Prometheus plugin normally exposes metrics on port `15692` at the `/metrics` endpoint.

### 19.1 Enter the RabbitMQ Pod

```bash
RABBIT_POD=$(kubectl get pod -n $NS -l app=rabbitmq -o jsonpath='{.items[0].metadata.name}')
echo $RABBIT_POD
```

```bash
kubectl exec -it $RABBIT_POD -n $NS -- bash
```

Inside the pod:

```bash
rabbitmq-plugins enable rabbitmq_prometheus
rabbitmq-diagnostics -s listeners
exit
```

---

### 19.2 Create a Service for RabbitMQ Metrics

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

Apply:

```bash
kubectl apply -f azure-prometheus-grafana/rabbitmq-prometheus-svc.yaml
```

Validate locally:

```bash
kubectl port-forward svc/rabbitmq-prometheus 15692:15692 -n $NS
```

In another terminal:

```bash
curl http://localhost:15692/metrics | head
```

---

### 19.3 Discussion

If metrics are visible:

- RabbitMQ is exposing Prometheus metrics.
- We can integrate these metrics into a custom scrape configuration.
- We can create alerts for queue depth, unacked messages, and consumers.

If metrics are not visible:

- Continue using RabbitMQ UI for backlog.
- Use Elastic/Kibana for logs.
- Use Prometheus/Grafana for general Kubernetes health.

---

## 20. Incident 3 – RabbitMQ Down

### 20.1 Trigger the Failure

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
```

Validate:

```bash
kubectl get pods -n $NS
kubectl get statefulset rabbitmq -n $NS
```

---

### 20.2 Observe in Grafana

Look for:

- unavailable pods
- restarts
- deployment changes
- indirect errors in workloads

---

### 20.3 Observe in Kibana

Search:

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

### 20.4 Questions for Students

1. Should order-service reject orders?
2. Should it store them locally?
3. Does this application implement the Outbox Pattern?
4. What evidence do I have that RabbitMQ is down?
5. Which metric would alert before the user reports the problem?

---

### 20.5 Restore RabbitMQ

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $NS --timeout=180s
```

---

## 21. Incident 4 – Redis Stale Data

### 21.1 Validate Redis

```bash
REDIS_POD=$(kubectl get pod -n $NS -l app=redis -o jsonpath='{.items[0].metadata.name}')
echo $REDIS_POD
```

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli GET product:dog-food
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

---

### 21.2 Create Stale Data

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":25.00,"stock":100}'
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

Typical result:

```text
-1
```

Meaning:

```text
The key does not expire automatically.
```

---

### 21.3 Add TTL

```bash
kubectl exec -it $REDIS_POD -n $NS -- redis-cli EXPIRE product:dog-food 300
kubectl exec -it $REDIS_POD -n $NS -- redis-cli TTL product:dog-food
```

---

### 21.4 Questions for Students

1. Is the system failing technically?
2. Could it be failing from a business perspective?
3. Does Prometheus automatically detect stale data?
4. What business metric would we need?
5. Should checkout trust cache?

Key message:

> Not every observable problem is technical. Some problems are business inconsistencies.

---

## 22. Incident 5 – Expected Istio/JWT 403

### 22.1 Test Without Token

```bash
GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

```bash
curl -s -o /dev/null -w "%{http_code}
" http://$GATEWAY_IP
```

Expected result:

```text
403
```

---

### 22.2 Questions for Students

1. Is the app down?
2. Or is the security policy working?
3. Which logs would we search?
4. How do we differentiate an expected 403 from an incident?
5. Which dashboard should show denied traffic?

Key message:

> A 403 does not always mean error. It can be evidence that Zero Trust is working.

---

## 23. Final Challenge – Observability War Room

### 23.1 Dynamic

Split the class into teams.

The instructor triggers a failure without saying which one.

Options:

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

### 23.2 Each Team Must Deliver a Diagnosis

Create file:

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

### 23.3 Guiding Questions

1. Which component failed?
2. Is the system completely down or degraded?
3. Which metric changed?
4. Which logs explain the problem?
5. Which Kubernetes command confirms the diagnosis?
6. What is the business impact?
7. Which alert should exist?
8. Which architectural pattern would help?
9. Which runbook should we document?
10. How do we prevent the incident from happening again?

---

## 24. Minimal Recovery Runbook

Create file:

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

## 25. Best Practices Students Should Take Away

### 25.1 Metrics

Use metrics to answer:

```text
What is happening?
When did it start?
How much impact does it have?
Is it improving or getting worse?
```

### 25.2 Logs

Use logs to answer:

```text
What exactly happened?
Which error did the application return?
Which request or event failed?
```

### 25.3 Traces

Use traces to answer:

```text
Where was the request delayed?
Which dependency was slow?
Which service broke the flow?
```

### 25.4 Dashboards

A dashboard must be actionable.

Bad dashboard:

```text
Many nice-looking charts without a clear decision.
```

Good dashboard:

```text
Shows whether the business is healthy, degraded, or down.
```

### 25.5 Alerts

An alert must have:

- symptom
- impact
- severity
- owner
- recommended action

---

## 26. Lab Cleanup

### 26.1 Restore Components

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
```

Validate:

```bash
kubectl get pods -n $NS
```

---

### 26.2 Delete Elastic Agent

If you want to remove Elastic Agent from the cluster:

```bash
kubectl delete -f elastic/elastic-agent-managed-kubernetes.yaml
```

Validate:

```bash
kubectl get pods -n kube-system | grep elastic
```

---

### 26.3 Delete Azure Monitor/Grafana Resources

> Only if they are no longer needed.

```bash
az grafana delete   --name $GRAFANA_NAME   --resource-group $RG_NAME   --yes
```

```bash
az resource delete --ids $AMW_ID
```

---

## 27. Lab Closing

Final message:

> In microservices, designing resilient systems is not enough. We must be able to observe, measure, diagnose, and explain how they behave when they fail.

Final concepts:

- `kubectl get pods` is not enough.
- Centralized logs make it possible to investigate distributed incidents.
- Prometheus shows system health and trends.
- Grafana turns metrics into operational decisions.
- Elastic/Kibana helps us find technical evidence.
- RabbitMQ decouples services, but backlog must be observed.
- MongoDB going down does not always bring all pods down, but it does degrade the business.
- Redis improves performance, but it can serve stale data.
- Istio can return 403 as correct behavior.
- Real observability combines metrics, logs, traces, and business context.

Final sentence for students:

> Building microservices is only half the work. The other half is being able to understand them when something fails.

---

## 28. Official References

- Elastic Cloud Trial: https://www.elastic.co/cloud/cloud-trial-overview
- Evaluate Elastic during a trial: https://www.elastic.co/docs/get-started/evaluate-elastic
- Run Elastic Agent on Azure AKS managed by Fleet: https://www.elastic.co/docs/reference/fleet/running-on-aks-managed-by-fleet
- Run Elastic Agent on Kubernetes managed by Fleet: https://www.elastic.co/docs/reference/fleet/running-on-kubernetes-managed-by-fleet
- Azure Monitor managed service for Prometheus: https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-metrics-overview
- Enable monitoring for AKS: https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable
- Monitor AKS: https://learn.microsoft.com/en-us/azure/aks/monitor-aks
- Connect Grafana to Azure Monitor managed Prometheus: https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-grafana
- RabbitMQ Prometheus metrics: https://www.rabbitmq.com/docs/prometheus
