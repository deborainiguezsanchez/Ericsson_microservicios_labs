# Lab: Zero Trust Security with Istio for the AKS Store Polyglot Application

## 1. Lab Overview

In this lab, you will secure the **AKS Store polyglot application** using **Istio on Azure Kubernetes Service (AKS)**.

This lab is designed to be executed after the previous AKS Store labs, where you deployed or prepared the distributed application with these main components:

```text
store-front
product-service
order-service
rabbitmq
```

The goal is to apply **Zero Trust Security** to a real distributed application instead of using a synthetic demo.

You will implement:

- Istio sidecar injection on the application namespace.
- mTLS STRICT for service-to-service encryption.
- Default deny authorization.
- Least-privilege service-to-service access.
- Explicit validation of allowed and denied paths.
- Troubleshooting checks for sidecars, service accounts, policies, and traffic.

---

## 2. Business Scenario

The AKS Store application represents a simple e-commerce system.

The main business flow is:

```text
Customer
  ↓
store-front
  ↓
product-service       Used to display products
  ↓
order-service         Used to place orders
  ↓
rabbitmq              Used as the order queue
```

From a Zero Trust perspective, we do not want every component to call every other component.

Instead, we want a controlled access model:

```text
store-front     → product-service     Allowed
store-front     → order-service       Allowed
order-service   → rabbitmq            Allowed

store-front     → rabbitmq            Denied
product-service → order-service       Denied
product-service → rabbitmq            Denied
outside-mesh    → any mesh service     Denied
```

The key idea is:

```text
Every workload must have an identity.
Every connection must be encrypted.
Every service-to-service call must be explicitly allowed.
Everything else must be denied.
```

---

## 3. Learning Objectives

By the end of this lab, students will be able to:

1. Enable the Istio service mesh add-on on AKS.
2. Add the AKS Store application namespace to the mesh.
3. Validate Istio sidecar injection.
4. Assign explicit Kubernetes ServiceAccounts to application workloads.
5. Apply mTLS STRICT using Istio `PeerAuthentication`.
6. Apply a namespace-level default deny policy.
7. Apply least-privilege `AuthorizationPolicy` rules.
8. Test allowed service-to-service communication.
9. Test denied service-to-service communication.
10. Validate non-mesh traffic rejection.
11. Troubleshoot common Istio security issues on AKS.

---

## 4. Target Architecture

### 4.1 Application Architecture

```text
+-------------+        +-----------------+
|  Customer   | -----> |   store-front   |
+-------------+        +-----------------+
                              |
                              | HTTP
                              v
                       +-----------------+
                       | product-service |
                       +-----------------+

                              |
                              | HTTP
                              v
                       +-----------------+
                       |  order-service |
                       +-----------------+
                              |
                              | AMQP / TCP 5672
                              v
                       +-----------------+
                       |     rabbitmq    |
                       +-----------------+
```

### 4.2 Zero Trust Security Architecture

```text
Istio mTLS STRICT
+
Default Deny
+
Explicit Allow Policies
+
Workload Identity Based on Kubernetes ServiceAccounts
```

Workload identities:

```text
cluster.local/ns/<namespace>/sa/store-front-sa
cluster.local/ns/<namespace>/sa/product-service-sa
cluster.local/ns/<namespace>/sa/order-service-sa
cluster.local/ns/<namespace>/sa/rabbitmq-sa
```

---

## 5. Prerequisites

You need:

- Azure subscription access.
- Existing AKS cluster.
- Existing Azure CLI login.
- `kubectl` access to the AKS cluster.
- AKS Store application already deployed, or ready to be deployed.
- Sufficient permissions to enable the Istio add-on on AKS.
- Linux node pool.
- No conflicting self-managed Istio installation on the same AKS cluster.

> Important: The AKS Istio add-on uses revision-based sidecar injection. Do not use the classic `istio-injection=enabled` label with the AKS managed Istio add-on. Use `istio.io/rev=<revision>`.

---

## 6. Set Lab Variables

Adjust these variables for your environment.

```bash
export RESOURCE_GROUP="user20"
export CLUSTER_NAME="myAKSCluster0325720758"
export LOCATION="centralus"

# Namespace where the AKS Store application is deployed.
# Change this if your app uses a different namespace.
export APP_NS="aks-store"

# Namespace used to test traffic from outside the mesh.
export OUTSIDE_NS="outside-mesh"
```

Validate Azure access:

```bash
az account show -o table
```

Connect to AKS:

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing
```

Validate cluster access:

```bash
kubectl get nodes
```

---

## 7. Validate the AKS Store Application

Check that the application namespace exists:

```bash
kubectl get ns
```

If the namespace does not exist, create it:

```bash
kubectl create namespace $APP_NS
```

Check the application workloads:

```bash
kubectl get deploy -n $APP_NS
```

Expected deployments:

```text
store-front
product-service
order-service
rabbitmq
```

Check the services:

```bash
kubectl get svc -n $APP_NS
```

Expected services:

```text
store-front
product-service
order-service
rabbitmq
```

If your service names are different, update the commands in the lab accordingly.

---

## 8. Capture Service Ports

Because different manifests may expose different ports, capture service ports dynamically.

```bash
export STORE_FRONT_PORT=$(kubectl get svc store-front -n $APP_NS -o jsonpath='{.spec.ports[0].port}')
export PRODUCT_SERVICE_PORT=$(kubectl get svc product-service -n $APP_NS -o jsonpath='{.spec.ports[0].port}')
export ORDER_SERVICE_PORT=$(kubectl get svc order-service -n $APP_NS -o jsonpath='{.spec.ports[0].port}')
export RABBITMQ_PORT=$(kubectl get svc rabbitmq -n $APP_NS -o jsonpath='{.spec.ports[?(@.name=="amqp")].port}')

# Fallback if the RabbitMQ service port does not have the name "amqp"
if [ -z "$RABBITMQ_PORT" ]; then
  export RABBITMQ_PORT=$(kubectl get svc rabbitmq -n $APP_NS -o jsonpath='{.spec.ports[0].port}')
fi

echo "store-front port: $STORE_FRONT_PORT"
echo "product-service port: $PRODUCT_SERVICE_PORT"
echo "order-service port: $ORDER_SERVICE_PORT"
echo "rabbitmq port: $RABBITMQ_PORT"
```

Typical values:

```text
store-front: 80 or 8080
product-service: 3002
order-service: 3000
rabbitmq: 5672
```

---

## 9. Enable Istio Add-on on AKS

Check whether the Istio add-on is already enabled:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "serviceMeshProfile.mode" \
  -o tsv
```

If the output is:

```text
Istio
```

the add-on is already enabled.

If the output is empty, enable the add-on:

```bash
az aks mesh enable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

Verify:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "serviceMeshProfile.mode" \
  -o tsv
```

Expected output:

```text
Istio
```

---

## 10. Verify Istio Control Plane

Refresh AKS credentials:

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing
```

Check Istio control plane pods:

```bash
kubectl get pods -n aks-istio-system
```

Expected result:

```text
NAME                               READY   STATUS    RESTARTS   AGE
istiod-asm-x-xx-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
```

---

## 11. Get the Istio Revision

The AKS Istio add-on requires revision-based namespace labeling.

```bash
export ISTIO_REVISION=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "serviceMeshProfile.istio.revisions[0]" \
  -o tsv)

echo $ISTIO_REVISION
```

Example output:

```text
asm-1-24
```

---

## 12. Add the AKS Store Namespace to the Mesh

Label the application namespace with the Istio revision:

```bash
kubectl label namespace $APP_NS istio.io/rev=$ISTIO_REVISION --overwrite
```

Validate:

```bash
kubectl get namespace $APP_NS --show-labels
```

Expected label:

```text
istio.io/rev=asm-x-xx
```

---

## 13. Create Dedicated ServiceAccounts

Zero Trust policies work best when each workload has a distinct identity.

Create dedicated service accounts:

```bash
kubectl create serviceaccount store-front-sa -n $APP_NS --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount product-service-sa -n $APP_NS --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount order-service-sa -n $APP_NS --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount rabbitmq-sa -n $APP_NS --dry-run=client -o yaml | kubectl apply -f -
```

Validate:

```bash
kubectl get serviceaccount -n $APP_NS
```

Expected service accounts:

```text
store-front-sa
product-service-sa
order-service-sa
rabbitmq-sa
```

---

## 14. Patch Deployments to Use Dedicated ServiceAccounts

Patch each deployment.

```bash
kubectl patch deployment store-front -n $APP_NS \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"store-front-sa"}}}}'

kubectl patch deployment product-service -n $APP_NS \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"product-service-sa"}}}}'

kubectl patch deployment order-service -n $APP_NS \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"order-service-sa"}}}}'

kubectl patch statefulset rabbitmq -n $APP_NS \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"rabbitmq-sa"}}}}'
```

Restart the deployments to trigger new pods with sidecar injection:

```bash
kubectl rollout restart deployment -n $APP_NS
```

Wait for all deployments:

```bash
kubectl rollout status deployment/store-front -n $APP_NS
kubectl rollout status deployment/product-service -n $APP_NS
kubectl rollout status deployment/order-service -n $APP_NS
kubectl rollout status statefulset/rabbitmq -n $APP_NS
```

Validate pods:

```bash
kubectl get pods -n $APP_NS
```

Expected result:

```text
READY 2/2
```

The `2/2` means:

```text
application container + istio-proxy sidecar
```

---

## 15. Validate Sidecar Injection

Check one deployment:

```bash
kubectl get pod -n $APP_NS -l app=store-front \
  -o jsonpath='{.items[0].spec.initContainers[*].name}'
echo
```

Expected output should include:

```text
istio-proxy
```

If you do not see `istio-proxy`, check:

```bash
kubectl get namespace $APP_NS --show-labels
kubectl describe pod -n $APP_NS -l app=store-front
```

---

## 16. Deploy Test Clients with Different Identities

We will deploy small test clients using the same service accounts as the application workloads.

These clients are used only to validate authorization policies.

```bash
cat <<EOF > aks-store-test-clients.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-store-front
  namespace: ${APP_NS}
  labels:
    app: client-store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-store-front
  template:
    metadata:
      labels:
        app: client-store-front
    spec:
      serviceAccountName: store-front-sa
      containers:
      - name: netshoot
        image: nicolaka/netshoot:v0.13
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-product
  namespace: ${APP_NS}
  labels:
    app: client-product
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-product
  template:
    metadata:
      labels:
        app: client-product
    spec:
      serviceAccountName: product-service-sa
      containers:
      - name: netshoot
        image: nicolaka/netshoot:v0.13
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-order
  namespace: ${APP_NS}
  labels:
    app: client-order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-order
  template:
    metadata:
      labels:
        app: client-order
    spec:
      serviceAccountName: order-service-sa
      containers:
      - name: netshoot
        image: nicolaka/netshoot:v0.13
        command: ["sh", "-c", "sleep 365d"]
EOF
```

Apply:

```bash
kubectl apply -f aks-store-test-clients.yaml
```

Wait for the test clients:

```bash
kubectl rollout status deployment/client-store-front -n $APP_NS
kubectl rollout status deployment/client-product -n $APP_NS
kubectl rollout status deployment/client-order -n $APP_NS
```

Validate:

```bash
kubectl get pods -n $APP_NS | grep client
```

Expected:

```text
READY 2/2
```

---

## 17. Baseline Test Before Zero Trust Authorization

At this point, workloads are inside the mesh, but we have not yet applied restrictive authorization policies.

Test `store-front` identity to `product-service`:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://product-service:${PRODUCT_SERVICE_PORT}/health
```

Expected result:

```text
200
```

Test `store-front` identity to `order-service`:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://order-service:${ORDER_SERVICE_PORT}/health
```

Expected result:

```text
200
```

Test `store-front` identity to `rabbitmq`:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  nc -vz rabbitmq ${RABBITMQ_PORT}
```

Expected result before applying Zero Trust authorization:

```text
succeeded
```

This is intentionally insecure.

Right now, `store-front` can reach RabbitMQ directly.

We will fix that.

---

## 18. Apply mTLS STRICT

Create a namespace-wide `PeerAuthentication` policy:

```bash
cat <<EOF > peer-authentication-strict.yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ${APP_NS}
spec:
  mtls:
    mode: STRICT
EOF
```

Apply:

```bash
kubectl apply -f peer-authentication-strict.yaml
```

Validate:

```bash
kubectl get peerauthentication -n $APP_NS
```

Expected:

```text
NAME      MODE     AGE
default   STRICT   ...
```

Test mesh-to-mesh traffic:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://product-service:${PRODUCT_SERVICE_PORT}/health
```

Expected result:

```text
200
```

This works because both workloads are inside the mesh and use Istio sidecars.

---

## 19. Validate Non-Mesh Traffic Is Rejected

Create a namespace outside the mesh:

```bash
kubectl create namespace $OUTSIDE_NS --dry-run=client -o yaml | kubectl apply -f -
```

Create a non-mesh test client:

```bash
kubectl run outside-client \
  -n $OUTSIDE_NS \
  --image=nicolaka/netshoot:v0.13 \
  --command -- sleep 365d
```

Wait:

```bash
kubectl wait --for=condition=Ready pod/outside-client -n $OUTSIDE_NS --timeout=180s
```

Try to call `product-service` from outside the mesh:

```bash
kubectl exec -n $OUTSIDE_NS outside-client -- \
  curl -sS --max-time 5 http://product-service.${APP_NS}.svc.cluster.local:${PRODUCT_SERVICE_PORT}/health || true
```

Expected result:

```text
Connection reset
```

or:

```text
upstream connect error
```

or:

```text
curl timeout
```

This confirms that plaintext traffic from outside the mesh is rejected.

---

## 20. Apply Default Deny

Now apply the most important Zero Trust step:

```text
Deny everything by default.
```

Create the default deny policy:

```bash
cat <<EOF > default-deny.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: default-deny
  namespace: ${APP_NS}
spec: {}
EOF
```

Apply:

```bash
kubectl apply -f default-deny.yaml
```

Test `store-front` identity to `product-service`:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://product-service:${PRODUCT_SERVICE_PORT}/health
```

Expected result:

```text
403
```

This is correct.

Now all traffic is denied unless explicitly allowed.

---

## 21. Apply Least-Privilege Authorization Policies

Allowed communication paths:

```text
store-front   → product-service
store-front   → order-service
order-service → rabbitmq
```

Create the allow policies:

```bash
cat <<EOF > aks-store-allow-policies.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-store-front-to-product-service
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/store-front-sa
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-store-front-to-order-service
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/store-front-sa
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-order-service-to-rabbitmq
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: rabbitmq
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/order-service-sa
EOF
```

Apply:

```bash
kubectl apply -f aks-store-allow-policies.yaml
```

Validate:

```bash
kubectl get authorizationpolicy -n $APP_NS
```

Expected policies:

```text
default-deny
allow-store-front-to-product-service
allow-store-front-to-order-service
allow-order-service-to-rabbitmq
```

---

## 22. Test Allowed Paths

### 22.1 store-front identity to product-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://product-service:${PRODUCT_SERVICE_PORT}/health
```

Expected result:

```text
200
```

### 22.2 store-front identity to order-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://order-service:${ORDER_SERVICE_PORT}/health
```

Expected result:

```text
200
```

### 22.3 order-service identity to rabbitmq

```bash
kubectl exec -n $APP_NS deploy/client-order -c netshoot -- \
  nc -vz rabbitmq ${RABBITMQ_PORT}
```

Expected result:

```text
succeeded
```

---

## 23. Test Denied Paths

### 23.1 store-front identity to rabbitmq

The front end should not connect directly to RabbitMQ.

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  sh -c "nc -vz -w 5 rabbitmq ${RABBITMQ_PORT}; echo EXIT_CODE:\$?"
```

Expected result:

```text
EXIT_CODE:1
```

or connection failure.

### 23.2 product-service identity to order-service

Product Service should not call Order Service in this security model.

```bash
kubectl exec -n $APP_NS deploy/client-product -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://order-service:${ORDER_SERVICE_PORT}/health
```

Expected result:

```text
403
```

### 23.3 product-service identity to rabbitmq

Product Service should not connect to RabbitMQ.

```bash
kubectl exec -n $APP_NS deploy/client-product -c netshoot -- \
  sh -c "nc -vz -w 5 rabbitmq ${RABBITMQ_PORT}; echo EXIT_CODE:\$?"
```

Expected result:

```text
EXIT_CODE:1
```

or connection failure.

### 23.4 outside-mesh to product-service

```bash
kubectl exec -n $OUTSIDE_NS outside-client -- \
  curl -sS --max-time 5 http://product-service.${APP_NS}.svc.cluster.local:${PRODUCT_SERVICE_PORT}/health || true
```

Expected result:

```text
Connection failure
```

---

## 24. Validate the Actual Application Still Works

If `store-front` is exposed through a Kubernetes LoadBalancer service, get the public IP:

```bash
kubectl get svc store-front -n $APP_NS
```

If there is an external IP, test it in the browser.

If the service is internal only, use port-forward:

```bash
kubectl port-forward svc/store-front -n $APP_NS 8080:${STORE_FRONT_PORT}
```

Open:

```text
http://localhost:8080
```

Expected behavior:

- The store front loads.
- Products can be displayed.
- Orders can be placed.
- Unauthorized internal paths remain blocked.

---

## 25. Inspect Sidecar Proxies

Check containers inside a pod:

```bash
kubectl get pod -n $APP_NS -l app=store-front \
  -o jsonpath='{.items[0].spec.containers[*].name}'
echo
```

Expected:

```text
store-front istio-proxy
```

or:

```text
app istio-proxy
```

Check Envoy proxy logs:

```bash
kubectl logs -n $APP_NS deploy/store-front -c istio-proxy --tail=30
```

Check application logs:

```bash
kubectl logs -n $APP_NS deploy/store-front --all-containers=false --tail=30
```

Check denied traffic in proxy logs:

```bash
kubectl logs -n $APP_NS deploy/product-service -c istio-proxy --tail=50
```

Look for indications of denied requests or RBAC enforcement.

---

## 26. Troubleshooting

### 26.1 Pods show READY 1/1 instead of 2/2

Cause:

```text
Istio sidecar injection did not happen.
```

Check namespace label:

```bash
kubectl get namespace $APP_NS --show-labels
```

Expected:

```text
istio.io/rev=asm-x-xx
```

Fix:

```bash
kubectl label namespace $APP_NS istio.io/rev=$ISTIO_REVISION --overwrite
kubectl rollout restart deployment -n $APP_NS
```

Wait:

```bash
kubectl get pods -n $APP_NS -w
```

---

### 26.2 Allowed traffic returns 403

Check policies:

```bash
kubectl get authorizationpolicy -n $APP_NS
```

Check workload labels:

```bash
kubectl get deploy -n $APP_NS --show-labels
```

Check service account used by the client:

```bash
kubectl get pod -n $APP_NS -l app=client-store-front \
  -o jsonpath='{.items[0].spec.serviceAccountName}'
echo
```

Check that the principal in the policy matches your namespace and service account:

```text
cluster.local/ns/<namespace>/sa/<service-account>
```

---

### 26.3 Application breaks after default deny

This usually means one required communication path is missing.

Check application logs:

```bash
kubectl logs -n $APP_NS deploy/store-front --all-containers=true --tail=50
kubectl logs -n $APP_NS deploy/order-service --all-containers=true --tail=50
```

Check denied paths by testing manually:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://product-service:${PRODUCT_SERVICE_PORT}/health

kubectl exec -n $APP_NS deploy/client-store-front -c netshoot -- \
  curl -sS -o /dev/null -w "%{http_code}\n" http://order-service:${ORDER_SERVICE_PORT}/health

kubectl exec -n $APP_NS deploy/client-order -c netshoot -- \
  nc -vz rabbitmq ${RABBITMQ_PORT}
```

Add missing `AuthorizationPolicy` rules if needed.

---

### 26.4 RabbitMQ traffic does not work

Validate the service port:

```bash
kubectl get svc rabbitmq -n $APP_NS -o yaml
```

Validate the service selector:

```bash
kubectl get svc rabbitmq -n $APP_NS -o jsonpath='{.spec.selector}'
echo
```

Validate pod labels:

```bash
kubectl get pods -n $APP_NS --show-labels | grep rabbitmq
```

The `AuthorizationPolicy` selector must match the RabbitMQ pod labels.

---

### 26.5 Non-mesh traffic still works

Check mTLS policy:

```bash
kubectl get peerauthentication -n $APP_NS
kubectl describe peerauthentication default -n $APP_NS
```

Expected mode:

```text
STRICT
```

Check that target service pods have sidecars:

```bash
kubectl get pods -n $APP_NS
```

Expected:

```text
READY 2/2
```

---

## 27. Optional: Expose store-front Through Istio Ingress Gateway

This section is optional.

Enable the external Istio ingress gateway:

```bash
az aks mesh enable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --ingress-gateway-type external
```

Check the gateway service:

```bash
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```

Create Gateway and VirtualService:

```bash
cat <<EOF > store-front-istio-ingress.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: store-front-gateway
  namespace: ${APP_NS}
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: store-front-vs
  namespace: ${APP_NS}
spec:
  hosts:
  - "*"
  gateways:
  - store-front-gateway
  http:
  - route:
    - destination:
        host: store-front.${APP_NS}.svc.cluster.local
        port:
          number: ${STORE_FRONT_PORT}
EOF
```

Apply:

```bash
kubectl apply -f store-front-istio-ingress.yaml
```

Allow ingress gateway traffic to `store-front`.

```bash
export INGRESS_GATEWAY_SA=$(kubectl get pod -n aks-istio-ingress -l istio=aks-istio-ingressgateway-external \
  -o jsonpath='{.items[0].spec.serviceAccountName}')

cat <<EOF > allow-ingress-to-store-front.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-to-store-front
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: store-front
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/aks-istio-ingress/sa/${INGRESS_GATEWAY_SA}
EOF
```

Apply:

```bash
kubectl apply -f allow-ingress-to-store-front.yaml
```

Get public IP:

```bash
export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "http://${INGRESS_HOST_EXTERNAL}"
```

Test:

```bash
curl -sS "http://${INGRESS_HOST_EXTERNAL}"
```

---

## 28. Optional Advanced Challenge: JWT at the Edge

This section is optional.

mTLS protects workload-to-workload identity.

JWT protects end-user or client identity.

A common production model is:

```text
External Client
  ↓ JWT
Istio Ingress Gateway
  ↓ mTLS
Internal Microservices
```

With Istio, this can be implemented using:

```text
RequestAuthentication
+
AuthorizationPolicy
```

Do not apply this section unless you have a valid OIDC provider and test token.

Example structure:

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: store-front-jwt
  namespace: aks-store
spec:
  selector:
    matchLabels:
      app: store-front
  jwtRules:
  - issuer: "https://login.microsoftonline.com/<TENANT_ID>/v2.0"
    jwksUri: "https://login.microsoftonline.com/<TENANT_ID>/discovery/v2.0/keys"
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt-store-front
  namespace: aks-store
spec:
  selector:
    matchLabels:
      app: store-front
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals:
        - "*"
```

---

## 29. Instructor Talking Points

### Zero Trust

Do not trust the internal Kubernetes network by default.

### Identity

ServiceAccount identity becomes workload identity inside the mesh.

### mTLS

mTLS encrypts service-to-service traffic and authenticates workloads.

### Default deny

Default deny changes the security model from:

```text
Everything is allowed unless blocked.
```

to:

```text
Everything is blocked unless explicitly allowed.
```

### Least privilege

Each service should only communicate with the services required by the business workflow.

### Validation

Always test both sides:

```text
Allowed traffic must work.
Denied traffic must fail.
```

### Layered security

Istio AuthorizationPolicy is not the same as Kubernetes NetworkPolicy.

- Kubernetes NetworkPolicy controls Layer 3 / Layer 4 connectivity.
- Istio AuthorizationPolicy controls workload identity and service-level access through Envoy.

They can complement each other.

---

## 30. Cleanup

Delete test clients:

```bash
kubectl delete -f aks-store-test-clients.yaml
```

Delete policies:

```bash
kubectl delete -f aks-store-allow-policies.yaml
kubectl delete -f default-deny.yaml
kubectl delete -f peer-authentication-strict.yaml
```

Delete outside namespace:

```bash
kubectl delete namespace $OUTSIDE_NS
```

Optional: remove namespace from the mesh:

```bash
kubectl label namespace $APP_NS istio.io/rev-
```

Restart workloads to remove sidecars:

```bash
kubectl rollout restart deployment -n $APP_NS
```

Optional: disable external ingress gateway:

```bash
az aks mesh disable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --ingress-gateway-type external
```

Optional: disable Istio add-on completely:

```bash
az aks mesh disable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

Do not disable Istio if other labs or applications are using it.

---

## 31. Expected Final State

At the end of the lab:

- AKS Istio add-on is enabled.
- AKS Store application namespace is part of the mesh.
- Application pods have Istio sidecars.
- Each workload has a dedicated ServiceAccount.
- mTLS STRICT is enabled.
- Default deny authorization is enabled.
- Only required service-to-service paths are allowed.
- Unauthorized calls return `403` or connection failure.
- Non-mesh workloads cannot call mesh-protected services.
- The application still works through approved paths.

Final security posture:

```text
Workload identity
+
mTLS encryption
+
Default deny
+
Least privilege
+
Explicit validation
```

This is a practical Zero Trust foundation for distributed microservices on Azure Kubernetes Service.
