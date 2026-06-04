
# Lab: Distributed Application + Zero Trust Security with Istio on AKS

## Purpose

This lab shows how to deploy a distributed microservices-style application on Azure Kubernetes Service (AKS) and secure service-to-service communication using Istio with a Zero Trust approach.

The lab is designed for the Azure environment used during the course. It assumes that students already have an AKS cluster, kubectl access, and basic familiarity with Docker, Kubernetes Deployments, Services, and namespaces.

## What You Will Build

You will deploy a simplified e-commerce platform with these services:

```text
store-front
order-service
payment-service
inventory-service
shipping-service
notifications-service
```

The intended business flow is:

```text
Customer → store-front → order-service → payment-service
                                  └──→ inventory-service → shipping-service → notifications-service
```

The Zero Trust objective is to ensure that each service can only call the services it is explicitly allowed to call.

Allowed paths:

```text
store-front        → order-service
order-service      → payment-service
order-service      → inventory-service
inventory-service  → shipping-service
shipping-service   → notifications-service
```

Denied examples:

```text
store-front        → payment-service
store-front        → inventory-service
payment-service    → inventory-service
outside workload   → any mesh-protected service
```

## Learning Objectives

By the end of this lab, students will be able to:

1. Enable the Istio add-on on AKS.
2. Add an application namespace to the Istio service mesh.
3. Deploy a distributed application with multiple service identities.
4. Validate Istio sidecar injection.
5. Enforce mutual TLS between services.
6. Apply a default deny authorization model.
7. Apply least-privilege service-to-service authorization.
8. Test allowed and denied communication paths.
9. Validate that non-mesh traffic cannot access mesh-protected services.
10. Explain how Istio supports Zero Trust in AKS.

## Lab Architecture

```text
                    +----------------+
                    |   Customer     |
                    +----------------+
                            |
                            v
                    +----------------+
                    |  store-front   |
                    +----------------+
                            |
                            v
                    +----------------+
                    | order-service  |
                    +----------------+
                     /              \
                    v                v
          +----------------+   +-------------------+
          | payment-service|   | inventory-service |
          +----------------+   +-------------------+
                                      |
                                      v
                              +------------------+
                              | shipping-service |
                              +------------------+
                                      |
                                      v
                            +-----------------------+
                            | notifications-service |
                            +-----------------------+
```

Each workload will run with its own Kubernetes ServiceAccount:

```text
store-front-sa
order-service-sa
payment-service-sa
inventory-service-sa
shipping-service-sa
notifications-service-sa
```

Istio uses these service accounts as workload identities. That allows us to write authorization policies based on identity, not only on IP addresses or ports.

## Prerequisites

You need:

- Azure subscription access.
- An existing AKS cluster.
- Azure CLI.
- kubectl.
- Permission to enable the Istio add-on on AKS.
- AKS credentials configured in your terminal or Azure Cloud Shell.
- Linux node pool.
- Outbound internet access from AKS to pull public container images.

Recommended cluster sizing:

```text
Node count: 2
VM size: Standard_D4_v4 or similar
Region: centralus or the assigned classroom region
```

## Lab Variables

Set these variables according to your assigned environment.

```bash
export RESOURCE_GROUP="user20"
export CLUSTER_NAME="myAKSCluster0325720758"
export LOCATION="centralus"
export APP_NS="zt-ecommerce"
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

Validate the nodes:

```bash
kubectl get nodes
```

Expected result:

```text
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-xxxxxxxx-vmss000000    Ready    agent   ...   v1.xx.x
```

---

# Part 1: Enable Istio on AKS

## 1.1 Check if Istio is already enabled

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

then Istio is already enabled.

## 1.2 Enable the Istio add-on

If the previous command returns empty output, enable Istio:

```bash
az aks mesh enable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

Verify again:

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

## 1.3 Verify the Istio control plane

```bash
kubectl get ns
```

```bash
kubectl get pods -n aks-istio-system
```

Expected result:

```text
NAME                               READY   STATUS    RESTARTS   AGE
istiod-asm-x-xx-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
```

## 1.4 Get the Istio revision

AKS managed Istio uses revision-based sidecar injection.

```bash
export ISTIO_REVISION=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "serviceMeshProfile.istio.revisions[0]" \
  -o tsv)

echo $ISTIO_REVISION
```

Example:

```text
asm-1-24
```

---

# Part 2: Create the Application Namespace

Create the application namespace:

```bash
kubectl create namespace $APP_NS
```

Enable Istio injection using the AKS Istio revision:

```bash
kubectl label namespace $APP_NS istio.io/rev=$ISTIO_REVISION --overwrite
```

Validate the label:

```bash
kubectl get namespace $APP_NS --show-labels
```

Expected output includes:

```text
istio.io/rev=asm-x-xx
```

---

# Part 3: Deploy the Distributed E-commerce Application

The application uses simple HTTP echo containers. This keeps the lab focused on service identity, mTLS, and authorization policy rather than application code.

## 3.1 Create the application manifest

```bash
cat <<'YAML' > zt-ecommerce-app.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: store-front-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory-service-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shipping-service-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notifications-service-sa
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
  labels:
    app: store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      serviceAccountName: store-front-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=store-front: customer entry point"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: store-front
spec:
  selector:
    app: store-front
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      serviceAccountName: order-service-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=order-service: order created"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app: payment-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      serviceAccountName: payment-service-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=payment-service: payment authorized"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
  labels:
    app: inventory-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inventory-service
  template:
    metadata:
      labels:
        app: inventory-service
    spec:
      serviceAccountName: inventory-service-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=inventory-service: inventory reserved"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: inventory-service
spec:
  selector:
    app: inventory-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping-service
  labels:
    app: shipping-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shipping-service
  template:
    metadata:
      labels:
        app: shipping-service
    spec:
      serviceAccountName: shipping-service-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=shipping-service: shipment created"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: shipping-service
spec:
  selector:
    app: shipping-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notifications-service
  labels:
    app: notifications-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notifications-service
  template:
    metadata:
      labels:
        app: notifications-service
    spec:
      serviceAccountName: notifications-service-sa
      containers:
      - name: app
        image: hashicorp/http-echo:1.0
        args:
        - "-text=notifications-service: notification sent"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: notifications-service
spec:
  selector:
    app: notifications-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
YAML
```

## 3.2 Deploy the application

```bash
kubectl apply -n $APP_NS -f zt-ecommerce-app.yaml
```

Wait for the pods:

```bash
kubectl wait --for=condition=Ready pod --all -n $APP_NS --timeout=180s
```

Validate:

```bash
kubectl get pods -n $APP_NS
```

Expected result:

```text
READY 2/2
```

The `2/2` status means each pod has two containers:

```text
application container + istio-proxy sidecar
```

If pods show `1/1`, the namespace was not injected correctly.

---

# Part 4: Deploy Mesh-Aware Test Clients

These clients allow us to test different service identities.

## 4.1 Create client deployments

```bash
cat <<'YAML' > zt-ecommerce-clients.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-store-front
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
      - name: curl
        image: curlimages/curl:8.8.0
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-order
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
      - name: curl
        image: curlimages/curl:8.8.0
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-payment
  labels:
    app: client-payment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-payment
  template:
    metadata:
      labels:
        app: client-payment
    spec:
      serviceAccountName: payment-service-sa
      containers:
      - name: curl
        image: curlimages/curl:8.8.0
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-inventory
  labels:
    app: client-inventory
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-inventory
  template:
    metadata:
      labels:
        app: client-inventory
    spec:
      serviceAccountName: inventory-service-sa
      containers:
      - name: curl
        image: curlimages/curl:8.8.0
        command: ["sh", "-c", "sleep 365d"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-shipping
  labels:
    app: client-shipping
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-shipping
  template:
    metadata:
      labels:
        app: client-shipping
    spec:
      serviceAccountName: shipping-service-sa
      containers:
      - name: curl
        image: curlimages/curl:8.8.0
        command: ["sh", "-c", "sleep 365d"]
YAML
```

## 4.2 Deploy the clients

```bash
kubectl apply -n $APP_NS -f zt-ecommerce-clients.yaml
```

Wait for them:

```bash
kubectl wait --for=condition=Ready pod -l app=client-store-front -n $APP_NS --timeout=180s
kubectl wait --for=condition=Ready pod -l app=client-order -n $APP_NS --timeout=180s
kubectl wait --for=condition=Ready pod -l app=client-payment -n $APP_NS --timeout=180s
kubectl wait --for=condition=Ready pod -l app=client-inventory -n $APP_NS --timeout=180s
kubectl wait --for=condition=Ready pod -l app=client-shipping -n $APP_NS --timeout=180s
```

Validate:

```bash
kubectl get pods -n $APP_NS
```

---

# Part 5: Baseline Connectivity Test

Before applying policies, traffic is allowed.

## 5.1 store-front identity calls order-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  curl -sS http://order-service:8080
```

Expected result:

```text
order-service: order created
```

## 5.2 store-front identity calls payment-service directly

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  curl -sS http://payment-service:8080
```

Expected result:

```text
payment-service: payment authorized
```

This is intentionally insecure. The store front should not call Payments directly. We will fix this with authorization policies.

---

# Part 6: Enforce mTLS STRICT

Create a namespace-wide mTLS policy:

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

Apply it:

```bash
kubectl apply -f peer-authentication-strict.yaml
```

Validate:

```bash
kubectl get peerauthentication -n $APP_NS
```

Test mesh-to-mesh traffic:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  curl -sS http://order-service:8080
```

Expected result:

```text
order-service: order created
```

This still works because both workloads are inside the mesh.

---

# Part 7: Validate That Non-Mesh Traffic Fails

Create a namespace without Istio injection:

```bash
kubectl create namespace $OUTSIDE_NS
```

Create a curl pod outside the mesh:

```bash
kubectl run outside-client \
  -n $OUTSIDE_NS \
  --image=curlimages/curl:8.8.0 \
  --command -- sleep 365d
```

Wait for the pod:

```bash
kubectl wait --for=condition=Ready pod/outside-client -n $OUTSIDE_NS --timeout=180s
```

Try to call a mesh-protected service:

```bash
kubectl exec -n $OUTSIDE_NS outside-client -- \
  curl -sS --max-time 5 http://order-service.${APP_NS}.svc.cluster.local:8080 || true
```

Expected result:

```text
Connection reset, upstream connect error, or timeout
```

This confirms that plaintext traffic from outside the mesh is not accepted.

---

# Part 8: Apply Default Deny

Now deny all traffic by default.

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

Apply it:

```bash
kubectl apply -f default-deny.yaml
```

Test traffic from store-front to order-service:

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  sh -c 'curl -s -o /dev/null -w "%{http_code}\n" http://order-service:8080'
```

Expected result:

```text
403
```

This is correct. The mesh now denies all service-to-service traffic unless we explicitly allow it.

---

# Part 9: Apply Least-Privilege Authorization Policies

Create explicit allow policies for the business workflow.

```bash
cat <<EOF > ecommerce-allow-policies.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-store-front-to-order
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
  name: allow-order-to-payment
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/order-service-sa
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-order-to-inventory
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: inventory-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/order-service-sa
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-inventory-to-shipping
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: shipping-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/inventory-service-sa
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-shipping-to-notifications
  namespace: ${APP_NS}
spec:
  selector:
    matchLabels:
      app: notifications-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/${APP_NS}/sa/shipping-service-sa
EOF
```

Apply:

```bash
kubectl apply -f ecommerce-allow-policies.yaml
```

Validate:

```bash
kubectl get authorizationpolicy -n $APP_NS
```

---

# Part 10: Test Allowed Paths

## 10.1 store-front to order-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  sh -c 'curl -s -o /tmp/out -w "%{http_code}\n" http://order-service:8080 && cat /tmp/out'
```

Expected result:

```text
200
order-service: order created
```

## 10.2 order-service to payment-service

```bash
kubectl exec -n $APP_NS deploy/client-order -c curl -- \
  sh -c 'curl -s -o /tmp/out -w "%{http_code}\n" http://payment-service:8080 && cat /tmp/out'
```

Expected result:

```text
200
payment-service: payment authorized
```

## 10.3 order-service to inventory-service

```bash
kubectl exec -n $APP_NS deploy/client-order -c curl -- \
  sh -c 'curl -s -o /tmp/out -w "%{http_code}\n" http://inventory-service:8080 && cat /tmp/out'
```

Expected result:

```text
200
inventory-service: inventory reserved
```

## 10.4 inventory-service to shipping-service

```bash
kubectl exec -n $APP_NS deploy/client-inventory -c curl -- \
  sh -c 'curl -s -o /tmp/out -w "%{http_code}\n" http://shipping-service:8080 && cat /tmp/out'
```

Expected result:

```text
200
shipping-service: shipment created
```

## 10.5 shipping-service to notifications-service

```bash
kubectl exec -n $APP_NS deploy/client-shipping -c curl -- \
  sh -c 'curl -s -o /tmp/out -w "%{http_code}\n" http://notifications-service:8080 && cat /tmp/out'
```

Expected result:

```text
200
notifications-service: notification sent
```

---

# Part 11: Test Denied Paths

## 11.1 store-front to payment-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  sh -c 'curl -s -o /dev/null -w "%{http_code}\n" http://payment-service:8080'
```

Expected result:

```text
403
```

## 11.2 store-front to inventory-service

```bash
kubectl exec -n $APP_NS deploy/client-store-front -c curl -- \
  sh -c 'curl -s -o /dev/null -w "%{http_code}\n" http://inventory-service:8080'
```

Expected result:

```text
403
```

## 11.3 payment-service to inventory-service

```bash
kubectl exec -n $APP_NS deploy/client-payment -c curl -- \
  sh -c 'curl -s -o /dev/null -w "%{http_code}\n" http://inventory-service:8080'
```

Expected result:

```text
403
```

## 11.4 outside-mesh to order-service

```bash
kubectl exec -n $OUTSIDE_NS outside-client -- \
  curl -sS --max-time 5 http://order-service.${APP_NS}.svc.cluster.local:8080 || true
```

Expected result:

```text
Connection failure, upstream connect error, or timeout
```

---

# Part 12: Inspect Istio Sidecars

Check containers in a pod:

```bash
kubectl get pod -n $APP_NS -l app=order-service -o jsonpath='{.items[0].spec.containers[*].name}'
echo
```

Expected result:

```text
app istio-proxy
```

View proxy logs:

```bash
kubectl logs -n $APP_NS deploy/order-service -c istio-proxy --tail=30
```

View application logs:

```bash
kubectl logs -n $APP_NS deploy/order-service -c app --tail=30
```

Describe the workload:

```bash
kubectl describe pod -n $APP_NS -l app=order-service
```

---

# Part 13: Optional - Expose store-front Through Istio Ingress Gateway

Enable the external gateway:

```bash
az aks mesh enable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --ingress-gateway-type external
```

Validate the gateway service:

```bash
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```

Create Gateway and VirtualService:

```bash
cat <<EOF > store-front-gateway.yaml
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
          number: 8080
EOF
```

Apply:

```bash
kubectl apply -f store-front-gateway.yaml
```

Allow ingress gateway to call `store-front`:

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

Get the public endpoint:

```bash
export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "http://${INGRESS_HOST_EXTERNAL}"
```

Test:

```bash
curl -sS "http://${INGRESS_HOST_EXTERNAL}"
```

Expected result:

```text
store-front: customer entry point
```

---

# Part 14: Optional Advanced Challenge - JWT Validation at the Edge

This section requires a real identity provider such as Microsoft Entra ID.

Conceptually:

- mTLS protects service-to-service identity.
- JWT protects end-user or client identity.
- AuthorizationPolicy can combine both models.

Example structure only. Do not apply unless you have a valid token and identity provider configuration.

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: store-front-jwt
  namespace: zt-ecommerce
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
  namespace: zt-ecommerce
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

# Part 15: Troubleshooting

## Pods show READY 1/1 instead of 2/2

Cause: sidecar injection did not happen.

Check namespace labels:

```bash
kubectl get namespace $APP_NS --show-labels
```

Fix:

```bash
kubectl label namespace $APP_NS istio.io/rev=$ISTIO_REVISION --overwrite
kubectl rollout restart deployment -n $APP_NS
kubectl wait --for=condition=Ready pod --all -n $APP_NS --timeout=180s
```

## Allowed traffic returns 403

Check policies:

```bash
kubectl get authorizationpolicy -n $APP_NS
```

Check the service account used by the client:

```bash
kubectl get pod -n $APP_NS -l app=client-store-front -o jsonpath='{.items[0].spec.serviceAccountName}'
echo
```

Check the policy principal:

```text
cluster.local/ns/<namespace>/sa/<service-account>
```

The namespace and service account must match exactly.

## Non-mesh traffic still works

Check PeerAuthentication:

```bash
kubectl get peerauthentication -n $APP_NS
kubectl describe peerauthentication default -n $APP_NS
```

Expected mode:

```text
STRICT
```

Check the target pod has a sidecar:

```bash
kubectl get pod -n $APP_NS -l app=order-service
```

Expected:

```text
READY 2/2
```

## Images cannot be pulled

Check events:

```bash
kubectl get events -n $APP_NS --sort-by='.lastTimestamp'
```

If the environment has restricted outbound connectivity, preload images into Azure Container Registry and update the manifests to use ACR images.

---

# Part 16: Cleanup

Delete the application namespace:

```bash
kubectl delete namespace $APP_NS
```

Delete the outside namespace:

```bash
kubectl delete namespace $OUTSIDE_NS
```

Optional: disable the external Istio ingress gateway:

```bash
az aks mesh disable-ingress-gateway \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --ingress-gateway-type external
```

Optional: disable the Istio add-on completely:

```bash
az aks mesh disable \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME
```

Do not disable Istio if other labs or applications are using the service mesh.

---

# Instructor Talking Points

## Zero Trust principle

Do not assume that traffic is trusted just because it is inside the Kubernetes cluster.

## Why mTLS matters

mTLS provides encrypted communication and workload identity between services.

## Why ServiceAccounts matter

Istio can map workload identity to Kubernetes ServiceAccounts. This lets us write policies such as:

```text
Only order-service-sa can call payment-service.
```

## Why default deny matters

Default deny prevents accidental access. Access must be explicitly allowed.

## Why test both allowed and denied traffic

A security control is not validated until both cases are tested:

```text
Allowed traffic works.
Denied traffic fails.
```

## Kubernetes NetworkPolicy vs Istio AuthorizationPolicy

Kubernetes NetworkPolicy controls network-level access, typically at Layer 3 and Layer 4.

Istio AuthorizationPolicy controls service identity and application-level authorization through the sidecar proxy.

They are complementary controls.

---

# Expected Final State

At the end of the lab:

- Istio is enabled on AKS.
- The `zt-ecommerce` namespace is part of the mesh.
- All application workloads have Istio sidecars.
- mTLS STRICT is enabled.
- Default deny is applied.
- Only explicitly approved service-to-service communication works.
- Unauthorized service calls return `403`.
- Non-mesh traffic cannot call mesh-protected services.

Final security posture:

```text
Identity-based access
+
Encrypted service-to-service traffic
+
Least-privilege communication
+
Explicit validation
```

This is the foundation of Zero Trust Security for microservices on AKS.
