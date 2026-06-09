# Advanced Lab: State Management and Persistence in Microservices on AKS Store Demo

## Lab Title

**State, Persistence, Failure Behavior, and Resilience Patterns in Microservices on Azure Kubernetes Service**

---

## 1. Lab Purpose

This lab uses the **AKS Store Demo** as the reference application to study real distributed-system behavior on Azure Kubernetes Service (AKS).

This is not just a deployment lab. The main goal is to understand what happens when a microservices application has state, queues, databases, caches, partial failures, and asynchronous processing.

During Days 4, 5, and 6, we worked with:

- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Jenkins
- Docker images
- Kubernetes Deployments and Services
- A polyglot microservices application
- RabbitMQ
- MongoDB
- Istio and Zero Trust concepts
- Service-to-service communication
- mTLS and authorization policies
- Distributed tracing and observability concepts

This lab connects those topics with one of the most difficult areas in microservices architecture:

```text
How do we manage state safely in a distributed system?
```

---

## 2. Target Audience

This lab is designed for students who already understand the basics of containers, Kubernetes, AKS, Deployments, Services, basic microservices architecture, and event-driven architecture.

The lab is intentionally advanced. It includes failure simulation, state inspection, persistence validation, RabbitMQ queue behavior, Redis cache behavior, and architecture discussions.

---

## 3. Estimated Duration

Recommended duration: **5 to 6 hours**

| Block | Topic | Estimated Time |
|---|---|---:|
| 0 | Environment preparation and safety checks | 25 min |
| 1 | Deploy AKS Store Demo in an isolated namespace | 35 min |
| 2 | Architecture and state ownership review | 35 min |
| 3 | Kubernetes persistence fundamentals with PVCs | 45 min |
| 4 | RabbitMQ and queue-based processing | 45 min |
| 5 | Failure scenario 1: `makeline-service` is down | 45 min |
| 6 | Failure scenario 2: RabbitMQ is down | 40 min |
| 7 | Failure scenario 3: MongoDB is down | 40 min |
| 8 | Redis cache lab: cache-aside, TTL, stale data | 45 min |
| 9 | Sagas, Outbox, CAP, and consistency discussion | 35 min |
| 10 | Observability, troubleshooting, cleanup | 30 min |

---

## 4. Official Application Used

Repository:

```text
https://github.com/Azure-Samples/aks-store-demo
```

Full Kubernetes manifest:

```text
https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml
```

The application is a realistic AKS sample with a polyglot architecture, event-driven communication, RabbitMQ, MongoDB, and multiple application services.

---

## 5. Application Components

| Component | Type | Responsibility | Stateful or Stateless |
|---|---|---|---|
| `store-front` | Frontend | Customer-facing web application | Stateless |
| `store-admin` | Frontend | Admin web application | Stateless |
| `order-service` | API service | Receives and places orders | Stateless compute, depends on queue |
| `product-service` | API service | Product operations | Stateless in this demo |
| `makeline-service` | Worker/API service | Processes orders from RabbitMQ and writes to MongoDB | Stateless compute, depends on queue and DB |
| `virtual-customer` | Simulator | Generates orders automatically | Stateless |
| `virtual-worker` | Simulator | Simulates workers completing orders | Stateless |
| `rabbitmq` | Broker | Queue for order messages | Stateful infrastructure |
| `mongodb` | Database | Order persistence | Stateful infrastructure |
| `redis` | Cache | Added by this lab for cache demonstrations | Stateful/cache infrastructure |

---

## 6. Architecture Narrative

The core order flow is:

```text
virtual-customer / store-front
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

Key message:

```text
The order is not completed in one single transaction.
The system accepts the order, publishes work to a queue, processes it asynchronously, and eventually persists the final state.
```

Instructor message:

> In a monolith, persistence often looks simple because one database transaction can update several tables at once.
>
> In microservices, each service should own its own state. Once we split the system, we also split transactions, consistency boundaries, failure modes, and ownership.
>
> This means we must design for partial failure, eventual consistency, message duplication, delayed processing, retries, cache staleness, and observability.

---

# 0. Environment Preparation

## 0.1 Recommended Execution Environment

Use **Azure Cloud Shell - Bash** or a Linux terminal with Azure CLI, kubectl, access to the AKS cluster, and Internet access to pull container images.

This lab assumes Bash syntax.

---

## 0.2 Set Variables

Adjust these values to your environment.

```bash
export RG_NAME="user20"
export AKS_NAME="myAKSCluster0325720758"
export LOCATION="centralus"
export NS="aks-store-persistence-lab"
export STUDENT_ID="user20"
```

Validate variables:

```bash
echo $RG_NAME
echo $AKS_NAME
echo $LOCATION
echo $NS
```

---

## 0.3 Connect to AKS

```bash
az aks get-credentials \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --overwrite-existing
```

Validate connection:

```bash
kubectl get nodes
kubectl get ns
```

Expected result:

```text
The AKS nodes should be visible and in Ready state.
```

---

## 0.4 Important Day 6 / Istio Safety Check

If you completed the Istio Zero Trust lab on Day 6, make sure this new lab runs in a **clean namespace**.

Check existing Istio labels:

```bash
kubectl get ns --show-labels | grep -E "istio|aks-store|persistence" || true
```

This persistence lab does **not** require Istio.

Recommended approach:

```text
Use a new namespace without Istio injection.
Do not reuse the namespace where default-deny AuthorizationPolicy was applied.
```

Create the namespace:

```bash
kubectl create namespace $NS --dry-run=client -o yaml | kubectl apply -f -
kubectl config set-context --current --namespace=$NS
```

Validate:

```bash
kubectl config view --minify | grep namespace
```

---

## 0.5 Check Storage Classes

```bash
kubectl get storageclass
```

Expected result in AKS might include:

```text
default
managed-csi
managed-csi-premium
azurefile-csi
```

Exact names can vary. This lab does not hardcode a StorageClass in the default PVC so the cluster default can be used.

---

# 1. Deploy AKS Store Demo

## 1.1 Download the Official Manifest

```bash
curl -fsSL \
  https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml \
  -o aks-store-all-in-one.yaml
```

Validate file:

```bash
ls -lh aks-store-all-in-one.yaml
head -n 5 aks-store-all-in-one.yaml
```

---

## 1.2 Avoid Public LoadBalancers for the Lab

The official manifest exposes `store-front` and `store-admin` as `LoadBalancer`.

For a controlled classroom environment, use `ClusterIP` and access the applications with `kubectl port-forward`.

```bash
grep -n "type: LoadBalancer" aks-store-all-in-one.yaml || true
sed -i 's/type: LoadBalancer/type: ClusterIP/g' aks-store-all-in-one.yaml
grep -n "type:" aks-store-all-in-one.yaml | head -20
```

Instructor explanation:

> In production, you would normally expose applications through an ingress controller, API gateway, Azure Application Gateway, or Istio Ingress Gateway.
>
> In this lab, we use `ClusterIP` plus `port-forward` to keep the environment safe, simple, and low-cost.

---

## 1.3 Deploy the Application

```bash
kubectl apply -f aks-store-all-in-one.yaml -n $NS
kubectl get all -n $NS
kubectl wait --for=condition=Ready pod --all -n $NS --timeout=300s
```

If some pods are still starting:

```bash
kubectl get pods -n $NS -w
```

Press `Ctrl+C` when all pods are ready.

---

## 1.4 Validate Core Resources

```bash
kubectl get deploy -n $NS
kubectl get statefulset -n $NS
kubectl get svc -n $NS
```

Expected Deployments:

```text
order-service
product-service
makeline-service
store-front
store-admin
virtual-customer
virtual-worker
```

Expected StatefulSets:

```text
mongodb
rabbitmq
```

---

## 1.5 Access `store-front`

```bash
kubectl port-forward svc/store-front 8080:80 -n $NS
```

Open:

```text
http://localhost:8080
```

If using Azure Cloud Shell, use **Web Preview** on port `8080`.

---

## 1.6 Access `store-admin`

```bash
kubectl port-forward svc/store-admin 8081:80 -n $NS
```

Open:

```text
http://localhost:8081
```

---

## 1.7 Initial Logs

```bash
kubectl logs deploy/order-service -n $NS --tail=50
kubectl logs deploy/makeline-service -n $NS --tail=50
kubectl logs deploy/virtual-customer -n $NS --tail=50
kubectl logs deploy/virtual-worker -n $NS --tail=50
```

Instructor explanation:

> In a distributed application, logs are not optional. Logs are how we reconstruct what happened when a business process crosses multiple services.

---

# 2. Explore Architecture and State Ownership

## 2.1 Identify Stateless and Stateful Components

```bash
kubectl get pods -n $NS -o wide
```

Discussion:

| Component | Why it is stateless or stateful |
|---|---|
| `store-front` | UI, can be recreated without data loss |
| `store-admin` | UI, can be recreated without data loss |
| `order-service` | API compute, but depends on RabbitMQ |
| `makeline-service` | Worker compute, but depends on RabbitMQ and MongoDB |
| `rabbitmq` | Broker with queue state |
| `mongodb` | Database with persisted business data |
| `redis` | Cache state, not system of record |

Instructor explanation:

> A stateless pod can be destroyed and recreated without losing business data.
>
> A stateful component owns or temporarily holds information that matters to the business process.

---

## 2.2 Inspect Service Communication

```bash
kubectl get svc -n $NS
```

Expected internal communication:

```text
store-front       -> product-service
store-front       -> order-service
order-service     -> rabbitmq
makeline-service  -> rabbitmq
makeline-service  -> mongodb
store-admin       -> makeline-service / order data
```

Instructor explanation:

> Kubernetes Services provide stable internal DNS names.
>
> For example, `order-service` can call `rabbitmq` by using the service name `rabbitmq` instead of knowing the pod IP address.

---

## 2.3 Inspect Environment Variables

```bash
kubectl describe deploy order-service -n $NS | grep -A20 -i "Environment"
kubectl describe deploy makeline-service -n $NS | grep -A30 -i "Environment"
kubectl describe deploy virtual-customer -n $NS | grep -A20 -i "Environment"
```

Expected ideas:

```text
ORDER_QUEUE_HOSTNAME=rabbitmq
ORDER_QUEUE_PORT=5672
ORDER_QUEUE_NAME=orders
ORDER_DB_URI=mongodb://mongodb:27017
```

---

# 3. Kubernetes Persistence Fundamentals

## 3.1 Inspect Existing StatefulSets

```bash
kubectl get statefulset -n $NS
kubectl describe statefulset mongodb -n $NS
kubectl describe statefulset rabbitmq -n $NS
```

Check whether the official demo created PVCs:

```bash
kubectl get pvc -n $NS
```

Possible result:

```text
No resources found
```

Instructor explanation:

> A Kubernetes StatefulSet gives stable pod identity, but durable storage normally requires a PersistentVolumeClaim.
>
> A workload can be stateful from a Kubernetes identity perspective but still not be durable if no persistent volume is attached.

---

## 3.2 Create a Safe Persistence Probe

This StatefulSet demonstrates persistent storage without modifying the AKS Store application.

```bash
cat <<EOF > persistence-probe.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: persistence-probe
  namespace: ${NS}
spec:
  serviceName: persistence-probe
  replicas: 1
  selector:
    matchLabels:
      app: persistence-probe
  template:
    metadata:
      labels:
        app: persistence-probe
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: busybox
        image: busybox:1.37.0
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            cpu: 5m
            memory: 16Mi
          limits:
            cpu: 50m
            memory: 64Mi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: persistence-probe
  namespace: ${NS}
spec:
  clusterIP: None
  selector:
    app: persistence-probe
  ports:
  - name: dummy
    port: 80
    targetPort: 80
EOF
```

Apply and validate:

```bash
kubectl apply -f persistence-probe.yaml
kubectl rollout status statefulset/persistence-probe -n $NS
kubectl get pvc -n $NS
```

Expected:

```text
data-persistence-probe-0   Bound   pvc-...
```

---

## 3.3 Write Data to the Persistent Volume

```bash
kubectl exec -n $NS persistence-probe-0 -- \
  sh -c 'echo "Data written at $(date)" > /data/state.txt'

kubectl exec -n $NS persistence-probe-0 -- \
  cat /data/state.txt
```

---

## 3.4 Delete the Pod and Validate Data Survives

```bash
kubectl delete pod persistence-probe-0 -n $NS
kubectl wait --for=condition=Ready pod/persistence-probe-0 -n $NS --timeout=180s
kubectl exec -n $NS persistence-probe-0 -- cat /data/state.txt
```

Expected result:

```text
The file still exists.
```

Instructor explanation:

> This is the difference between container filesystem and persistent volume.
>
> The container can disappear. The pod can be recreated. The node can change. But the PVC gives the workload stable storage.

---

## 3.5 Check the Underlying PVC

```bash
kubectl get pvc data-persistence-probe-0 -n $NS -o wide
kubectl describe pvc data-persistence-probe-0 -n $NS
```

Instructor explanation:

> In AKS, a PVC is usually dynamically provisioned by a CSI driver and backed by Azure storage.
>
> The application does not need to know the Azure disk details. It only asks Kubernetes for storage.

---

# 4. RabbitMQ and Event-Driven Processing

## 4.1 Access RabbitMQ Management UI

```bash
kubectl port-forward svc/rabbitmq 15672:15672 -n $NS
```

Open:

```text
http://localhost:15672
```

Decode credentials:

```bash
echo "RabbitMQ username:"
kubectl get secret rabbitmq-secrets -n $NS -o jsonpath='{.data.RABBITMQ_DEFAULT_USER}' | base64 -d
echo

echo "RabbitMQ password:"
kubectl get secret rabbitmq-secrets -n $NS -o jsonpath='{.data.RABBITMQ_DEFAULT_PASS}' | base64 -d
echo
```

Expected:

```text
username
password
```

---

## 4.2 Inspect RabbitMQ Queues from CLI

```bash
export RABBITMQ_POD=$(kubectl get pod -n $NS -l app=rabbitmq -o jsonpath='{.items[0].metadata.name}')
echo $RABBITMQ_POD

kubectl exec -n $NS $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

If the `orders` queue is not visible yet, wait a minute for traffic or generate traffic by opening the store.

---

## 4.3 Increase Order Generation Temporarily

```bash
kubectl set env deployment/virtual-customer ORDERS_PER_HOUR=600 -n $NS
kubectl rollout restart deployment/virtual-customer -n $NS
kubectl rollout status deployment/virtual-customer -n $NS
```

Observe logs:

```bash
kubectl logs deploy/virtual-customer -n $NS --tail=100
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
```

Inspect queue:

```bash
kubectl exec -n $NS $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

Instructor explanation:

> RabbitMQ decouples order intake from order processing.
>
> This is a queue-based load leveling pattern. The producer can publish orders while the consumer processes them at its own pace.

---

## 4.4 Verify Orders in MongoDB

```bash
export MONGO_POD=$(kubectl get pod -n $NS -l app=mongodb -o jsonpath='{.items[0].metadata.name}')
echo $MONGO_POD

kubectl exec -n $NS $MONGO_POD -- \
  mongo orderdb --quiet --eval 'db.orders.count()'

kubectl exec -n $NS $MONGO_POD -- \
  mongo orderdb --quiet --eval 'db.orders.findOne()'
```

Instructor explanation:

> RabbitMQ is not the system of record.
>
> In this demo, RabbitMQ transports work. MongoDB stores the processed order state.

---

# 5. Failure Scenario 1: `makeline-service` Is Down

## 5.1 Scenario

We will stop the consumer that processes orders from RabbitMQ.

This simulates:

- Worker outage
- Consumer crash
- Slow downstream processing
- Backlog growth
- Eventual consistency

---

## 5.2 Scale `makeline-service` to Zero

```bash
kubectl scale deployment makeline-service --replicas=0 -n $NS
kubectl get deploy makeline-service -n $NS
kubectl get pods -n $NS | grep makeline || true
```

---

## 5.3 Observe RabbitMQ Backlog

Run several times:

```bash
kubectl exec -n $NS $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

Expected behavior:

```text
orders queue messages may increase.
consumers may drop to 0.
```

Observe logs:

```bash
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/virtual-customer -n $NS --tail=100
```

Instructor explanation:

> The system is partially available.
>
> `order-service` can still receive orders and publish messages.
>
> RabbitMQ can store the work.
>
> But `makeline-service` is unavailable, so final order processing is delayed.

---

## 5.4 Architecture Discussion: Availability vs Consistency

Ask students:

```text
Is the order accepted?
Is the order fully processed?
Is the system consistent right now?
```

Guided answer:

```text
The order was accepted as a message.
The final processing is delayed.
The system is temporarily inconsistent.
This is eventual consistency.
```

---

## 5.5 Restore `makeline-service`

```bash
kubectl scale deployment makeline-service --replicas=1 -n $NS
kubectl rollout status deployment/makeline-service -n $NS
kubectl logs deploy/makeline-service -n $NS --tail=100
```

Watch the queue:

```bash
kubectl exec -n $NS $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

Expected behavior:

```text
The consumer returns.
The queue starts draining.
Orders eventually reach MongoDB.
```

Check MongoDB again:

```bash
kubectl exec -n $NS $MONGO_POD -- \
  mongo orderdb --quiet --eval 'db.orders.count()'
```

Instructor message:

> A broker helps absorb failure, but it does not remove complexity.
>
> You still need to monitor backlog, message age, retries, duplicate handling, and idempotency.

---

# 6. Failure Scenario 2: RabbitMQ Is Down

## 6.1 Scenario

Now we will stop the broker.

This is more serious:

```text
order-service wants to publish an order
RabbitMQ is unavailable
the message cannot be queued
```

---

## 6.2 Scale RabbitMQ to Zero

```bash
kubectl scale statefulset rabbitmq --replicas=0 -n $NS
kubectl get statefulset rabbitmq -n $NS
kubectl get pods -n $NS | grep rabbitmq || true
```

---

## 6.3 Observe Application Behavior

```bash
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/virtual-customer -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl get events -n $NS --sort-by=.lastTimestamp | tail -30
```

Instructor explanation:

> When the consumer is down, the broker can buffer messages.
>
> When the broker itself is down, the producer cannot publish reliably unless the application has another durability mechanism.

---

## 6.4 Architecture Discussion: What Should `order-service` Do?

Ask:

```text
If RabbitMQ is down, should order-service accept the order?
```

Option A: Reject the order.

Pros:

```text
No hidden data loss.
Clear feedback to the customer.
Simpler architecture.
```

Cons:

```text
Lower availability.
Poorer user experience.
```

Option B: Accept the order and persist the intention locally.

Pros:

```text
Higher availability.
The customer can continue.
```

Cons:

```text
Requires Outbox Pattern.
Requires retry logic.
Requires reconciliation.
Requires idempotent publishing.
```

---

## 6.5 Explain the Outbox Pattern

Problem:

```text
Saving business data and publishing an event are two different operations.
They are not one atomic distributed transaction.
```

Outbox solution:

```text
1. Save the order in the service database.
2. Save an outbox record in the same local transaction.
3. A background publisher reads the outbox.
4. It publishes the message to RabbitMQ.
5. It marks the outbox record as published.
```

Diagram:

```text
Order Service
   |
   +--> Orders Table
   |
   +--> Outbox Table
              |
              v
       Outbox Publisher
              |
              v
          RabbitMQ
```

Instructor message:

> The Outbox Pattern does not make distributed systems simple.
>
> It makes failure explicit and recoverable.

---

## 6.6 Restore RabbitMQ

```bash
kubectl scale statefulset rabbitmq --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=rabbitmq -n $NS --timeout=300s

export RABBITMQ_POD=$(kubectl get pod -n $NS -l app=rabbitmq -o jsonpath='{.items[0].metadata.name}')
echo $RABBITMQ_POD

kubectl exec -n $NS $RABBITMQ_POD -- rabbitmq-diagnostics ping
```

If some services do not reconnect automatically:

```bash
kubectl rollout restart deployment/order-service -n $NS
kubectl rollout restart deployment/makeline-service -n $NS
kubectl rollout status deployment/order-service -n $NS
kubectl rollout status deployment/makeline-service -n $NS
```

---

# 7. Failure Scenario 3: MongoDB Is Down

## 7.1 Scenario

Now we stop the database.

This allows us to discuss persistence dependency, database as system of record, read/write degradation, worker failure, retry behavior, and cache as degraded read path.

---

## 7.2 Scale MongoDB to Zero

```bash
kubectl scale statefulset mongodb --replicas=0 -n $NS
kubectl get statefulset mongodb -n $NS
kubectl get pods -n $NS | grep mongodb || true
```

---

## 7.3 Observe Behavior

```bash
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/store-admin -n $NS --tail=100
kubectl logs deploy/order-service -n $NS --tail=100
kubectl get events -n $NS --sort-by=.lastTimestamp | tail -30
```

Instructor explanation:

> A database failure is different from a worker failure.
>
> A queue can buffer work, but if the system of record is unavailable, the system must decide whether to fail, retry, queue, or degrade functionality.

---

## 7.4 Discussion: What Should Still Work?

Ask students:

```text
If MongoDB is down, what operations should still work?
```

Possible answers:

```text
The UI might still load.
Some static product data may still display.
Order finalization may fail.
The worker may retry or fail.
The system may need a degraded mode.
```

Important nuance:

```text
Not every operation has the same consistency requirement.
```

Examples:

| Operation | Can tolerate stale data? | Comment |
|---|---|---|
| Product description | Often yes | Catalog data can be cached |
| Product price at checkout | Usually no | Business-critical |
| Payment authorization | No | Financial correctness |
| Order status | Maybe | Depends on UX and business rules |
| Inventory reservation | Usually no | Overselling risk |

---

## 7.5 Restore MongoDB

```bash
kubectl scale statefulset mongodb --replicas=1 -n $NS
kubectl wait --for=condition=Ready pod -l app=mongodb -n $NS --timeout=300s

export MONGO_POD=$(kubectl get pod -n $NS -l app=mongodb -o jsonpath='{.items[0].metadata.name}')
echo $MONGO_POD

kubectl exec -n $NS $MONGO_POD -- \
  mongo --quiet --eval 'db.runCommand({ ping: 1 })'
```

Restart dependent service if needed:

```bash
kubectl rollout restart deployment/makeline-service -n $NS
kubectl rollout status deployment/makeline-service -n $NS
```

---

# 8. Redis Cache Lab

## 8.1 Goal

We will add Redis to demonstrate distributed cache, TTL, cache-aside, cache hit, cache miss, stale data, manual invalidation, and why cache is not the same as system of record.

---

## 8.2 Deploy Redis

```bash
cat <<EOF > redis-cache-lab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ${NS}
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
  namespace: ${NS}
spec:
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  type: ClusterIP
EOF
```

Apply:

```bash
kubectl apply -f redis-cache-lab.yaml
kubectl rollout status deployment/redis -n $NS
kubectl get pods -n $NS -l app=redis
kubectl get svc redis -n $NS
```

---

## 8.3 Use Redis CLI

```bash
export REDIS_POD=$(kubectl get pod -n $NS -l app=redis -o jsonpath='{.items[0].metadata.name}')
echo $REDIS_POD
kubectl exec -n $NS $REDIS_POD -- redis-cli PING
```

Expected:

```text
PONG
```

---

## 8.4 Create Cached Product Data

```bash
kubectl exec -n $NS $REDIS_POD -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":25.00,"stock":100}'
kubectl exec -n $NS $REDIS_POD -- redis-cli GET product:dog-food
kubectl exec -n $NS $REDIS_POD -- redis-cli EXPIRE product:dog-food 300
kubectl exec -n $NS $REDIS_POD -- redis-cli TTL product:dog-food
```

Expected:

```text
TTL should be a positive number.
```

Instructor explanation:

> TTL means the cache entry is temporary.
>
> This is useful because cache is not the source of truth. Eventually, cached data must expire or be invalidated.

---

## 8.5 Explain Cache-Aside

Flow:

```text
Client
  |
  v
Product Service
  |
  +--> Redis
       |
       +-- HIT  --> return cached product
       |
       +-- MISS --> query MongoDB or source system
                    write result to Redis
                    return result
```

Key message:

```text
Cache-aside improves read performance, but the application must handle cache misses and stale data.
```

---

## 8.6 Simulate Stale Data

```bash
kubectl exec -n $NS $REDIS_POD -- redis-cli GET product:dog-food
```

Imagine the real price changed to `20.00`, but Redis still returns `25.00`.

Discussion questions:

```text
Can product catalog data be stale?
Can checkout price be stale?
Can inventory availability be stale?
```

Guided answer:

```text
Catalog description may tolerate staleness.
Checkout price usually should not.
Inventory availability depends on business rules.
```

---

## 8.7 Invalidate Cache

```bash
kubectl exec -n $NS $REDIS_POD -- redis-cli DEL product:dog-food
kubectl exec -n $NS $REDIS_POD -- redis-cli GET product:dog-food
```

Expected:

```text
(nil)
```

Recreate with updated value:

```bash
kubectl exec -n $NS $REDIS_POD -- redis-cli SET product:dog-food '{"productId":"dog-food","name":"Dog Food","price":20.00,"stock":100}'
kubectl exec -n $NS $REDIS_POD -- redis-cli EXPIRE product:dog-food 300
kubectl exec -n $NS $REDIS_POD -- redis-cli GET product:dog-food
```

---

## 8.8 Cache Strategy Discussion

| Strategy | Description | Benefit | Risk |
|---|---|---|---|
| Cache-Aside | App loads cache on demand | Simple and common | Stale data |
| Write-Through | Write DB and cache together | Fresh cache | Slower writes |
| Write-Behind | Write cache first, DB later | Fast writes | Risk of data loss |
| TTL Only | Expire data after time | Easy to operate | Inconsistency window |
| Manual Invalidation | Delete/update cache on change | More precise | More logic required |

Instructor message:

> Cache is not only a performance tool. Cache is a consistency decision.
>
> Every cached value answers a business question: how stale is acceptable?

---

# 9. CAP, Sagas, and Consistency

## 9.1 CAP Discussion Using the Lab

### Case 1: `makeline-service` down

```text
RabbitMQ is available.
order-service is available.
makeline-service is down.
```

Behavior:

```text
Orders can still be accepted.
Messages accumulate.
Processing happens later.
```

Interpretation:

```text
The system favors availability and accepts eventual consistency.
```

---

### Case 2: RabbitMQ down

```text
order-service is available.
RabbitMQ is down.
```

Possible behavior:

```text
Reject orders.
Or accept orders only if Outbox Pattern exists.
```

Interpretation:

```text
Without Outbox, accepting orders can risk losing business intent.
With Outbox, availability improves but complexity increases.
```

---

### Case 3: MongoDB down

```text
The system of record is unavailable.
```

Possible behavior:

```text
Fail writes.
Retry processing.
Serve some reads from cache.
Enter degraded mode.
```

Interpretation:

```text
Cache improves availability but can return stale data.
```

---

## 9.2 Saga Concept Applied to AKS Store Demo

The demo does not implement a full e-commerce saga with payment, inventory, and shipping services.

However, we can use it to explain the idea.

Current simplified flow:

```text
Order Created
   |
   v
Order Queued
   |
   v
Order Processed
   |
   v
Order Stored
```

Extended saga flow:

```text
1. Create Order
2. Reserve Inventory
3. Authorize Payment
4. Create Shipment
5. Confirm Order
```

Possible compensations:

| Forward Action | Compensation |
|---|---|
| Create Order | Cancel Order |
| Reserve Inventory | Release Inventory |
| Authorize Payment | Refund or Void Payment |
| Create Shipment | Cancel Shipment |
| Send Notification | Send Correction / Follow-up |

Instructor message:

> In microservices, rollback is not always a database rollback.
>
> Often, rollback is a new business action that compensates a previous committed action.

---

## 9.3 Idempotency

Ask:

```text
What happens if RabbitMQ delivers the same message twice?
```

Expected answer:

```text
The consumer must process it safely only once from a business perspective.
```

Example:

```text
Message: ProcessOrder ORD-1001
```

Idempotent behavior:

```text
First delivery: create processed order
Second delivery: detect it already exists and ignore or update safely
```

Instructor message:

> Exactly-once delivery is difficult in distributed systems.
>
> Business-level idempotency is usually more important than believing the broker will never duplicate a message.

---

## 9.4 Correlation IDs

Ask:

```text
How do we follow one order across store-front, order-service, RabbitMQ, makeline-service, and MongoDB?
```

Answer:

```text
Use a correlationId or traceId.
```

Conceptual flow:

```text
store-front
   correlationId=ORD-1001
      |
      v
order-service
   correlationId=ORD-1001
      |
      v
RabbitMQ message
   correlationId=ORD-1001
      |
      v
makeline-service
   correlationId=ORD-1001
      |
      v
MongoDB order record
   correlationId=ORD-1001
```

Instructor message:

> Without correlation, troubleshooting is guesswork.
>
> With correlation, we can reconstruct the business transaction.

---

# 10. Observability and Troubleshooting

## 10.1 Logs by Service

```bash
kubectl logs deploy/order-service -n $NS --tail=100
kubectl logs deploy/makeline-service -n $NS --tail=100
kubectl logs deploy/product-service -n $NS --tail=100
kubectl logs deploy/store-front -n $NS --tail=100
```

---

## 10.2 Live Logs

Terminal 1:

```bash
kubectl logs deploy/order-service -n $NS -f
```

Terminal 2:

```bash
kubectl logs deploy/makeline-service -n $NS -f
```

Terminal 3:

```bash
kubectl exec -n $NS $RABBITMQ_POD -- \
  sh -c 'while true; do rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers; sleep 10; done'
```

---

## 10.3 Kubernetes Events

```bash
kubectl get events -n $NS --sort-by=.lastTimestamp | tail -50
```

---

## 10.4 Pod Health

```bash
kubectl get pods -n $NS -o wide
kubectl describe pod -n $NS <POD_NAME>
```

---

## 10.5 Resource Usage

If Metrics Server is available:

```bash
kubectl top pods -n $NS
kubectl top nodes
```

If not available:

```text
Metrics Server is required for kubectl top.
```

---

## 10.6 Optional: Connect with Day 6 OpenTelemetry Concepts

```text
Logs tell us what each service says happened.
Metrics tell us how often and how fast it happened.
Traces tell us how one business transaction moved across services.
```

Instructor explanation:

> For a production-grade version of this lab, every order should carry a trace ID from the frontend to the queue and into the worker.
>
> This is how we answer the business question: "What happened to order ORD-1001?"

---

# 11. Student Architecture Exercise

## 11.1 Scenario

```text
The store must continue accepting orders if makeline-service is down for up to 10 minutes.
But we must not lose orders.
```

Ask students:

1. Which component absorbs the failure?
2. What metric should we monitor?
3. When should we stop accepting new orders?
4. What happens if RabbitMQ also fails?
5. Do we need the Outbox Pattern?
6. Which operation must be idempotent?
7. Where should we place the correlation ID?
8. Which data can be cached?
9. Which data must not be stale?
10. What is the degraded-mode user experience?

---

## 11.2 Expected Answers

| Question | Expected Answer |
|---|---|
| Which component absorbs the failure? | RabbitMQ |
| What metric should we monitor? | Queue length, consumer count, oldest message age, processing rate |
| When should we stop accepting orders? | When backlog exceeds a business threshold |
| What if RabbitMQ fails? | Reject orders or use Outbox |
| Do we need Outbox? | Yes, if we accept orders while broker is down |
| What must be idempotent? | Order processing and message handling |
| Where should correlation ID go? | Request, message, logs, DB record |
| What can be cached? | Catalog descriptions, low-risk read models |
| What must not be stale? | Payment, checkout price, final inventory decision |
| What is degraded UX? | Read-only catalog, delayed order confirmation, temporary checkout pause |

---

# 12. Architecture Checklist

Before designing persistence in a microservices system, ask:

```text
1. Who owns the data?
2. Is there a shared database?
3. Which data belongs to which bounded context?
4. Is the operation synchronous or asynchronous?
5. What happens if the consumer is down?
6. What happens if the broker is down?
7. What happens if the database is down?
8. Is there retry with backoff?
9. Is message processing idempotent?
10. Is there a dead-letter queue?
11. Is there a compensation strategy?
12. Is the operation eventually consistent?
13. What inconsistency can the business tolerate?
14. What data can be cached?
15. How is cache invalidated?
16. Is stale data acceptable?
17. Is there a correlation ID?
18. Are logs, metrics, and traces available?
19. What is the degraded mode?
20. What is the manual recovery process?
```

---

# 13. Cleanup

## 13.1 Restore Traffic Rate

```bash
kubectl set env deployment/virtual-customer ORDERS_PER_HOUR=100 -n $NS
kubectl rollout restart deployment/virtual-customer -n $NS
```

---

## 13.2 Delete Redis

```bash
kubectl delete -f redis-cache-lab.yaml -n $NS --ignore-not-found=true
```

---

## 13.3 Delete Persistence Probe

```bash
kubectl delete -f persistence-probe.yaml -n $NS --ignore-not-found=true
```

PVCs created by StatefulSets may remain depending on retention behavior.

```bash
kubectl get pvc -n $NS
kubectl delete pvc data-persistence-probe-0 -n $NS --ignore-not-found=true
```

---

## 13.4 Delete AKS Store Demo

```bash
kubectl delete -f aks-store-all-in-one.yaml -n $NS --ignore-not-found=true
```

---

## 13.5 Delete Namespace

```bash
kubectl delete namespace $NS
kubectl get ns
```

---

# 14. Final Instructor Wrap-Up

Use this closing explanation:

> Today we used a real AKS microservices application to study state and persistence.
>
> We saw that persistence in microservices is not just about selecting a database.
>
> It is about ownership, consistency boundaries, failure behavior, retries, queues, cache invalidation, idempotency, and observability.
>
> RabbitMQ helps decouple producers and consumers, but it introduces backlog, ordering, retries, and duplicate-processing concerns.
>
> MongoDB acts as the system of record for processed orders, but when the database is unavailable, the system must decide whether to fail, retry, queue, or degrade.
>
> Redis improves read performance and resilience for some scenarios, but it can also return stale data.
>
> Kubernetes StatefulSets provide stable identity, but durable persistence requires PersistentVolumeClaims.
>
> Finally, Sagas and compensating actions remind us that rollback in microservices is often a business process, not a simple database command.

Final message:

```text
In microservices, designing persistence means designing behavior under failure.
```

---

# 15. Optional Advanced Extensions

## 15.1 Dead Letter Queue Discussion

```text
What if makeline-service cannot process one specific message?
```

A poison message can block or repeatedly fail processing. After controlled retries, the message should be moved to a DLQ for investigation.

---

## 15.2 Retry and Backoff Discussion

Retry immediately can amplify failure. Retry with exponential backoff reduces pressure on unhealthy dependencies.

---

## 15.3 Circuit Breaker Discussion

If MongoDB is down, calling it repeatedly can make recovery slower. A circuit breaker temporarily stops calls to a failing dependency.

---

## 15.4 Connect with Day 6 Zero Trust

Security and persistence are connected. A service should not be able to read or write state unless it has a valid identity and an explicit authorization policy.

Example:

```text
store-front should not talk directly to MongoDB.
product-service should not talk directly to RabbitMQ unless required.
makeline-service should talk to RabbitMQ and MongoDB because it owns that processing responsibility.
```

---

## 15.5 Production Hardening Discussion

Ask students what would be required for production:

```text
1. Managed database instead of single MongoDB pod.
2. Managed message broker or highly available RabbitMQ.
3. Persistent volumes with backup policy.
4. Secret management with Azure Key Vault.
5. Autoscaling based on queue depth.
6. DLQ and poison message handling.
7. Outbox Pattern for reliable publication.
8. Idempotent consumers.
9. Distributed tracing.
10. Alerting on backlog, latency, error rate, and oldest message age.
```

---

# 16. References

- AKS Store Demo: https://github.com/Azure-Samples/aks-store-demo
- Kubernetes StatefulSets: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Azure Architecture Center - Queue-Based Load Leveling: https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling
- Azure Architecture Center - Cache-Aside Pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside
- Azure Architecture Center - Compensating Transaction Pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction
- Redis EXPIRE command: https://redis.io/docs/latest/commands/expire/
