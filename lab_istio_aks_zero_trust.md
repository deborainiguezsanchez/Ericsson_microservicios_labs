# Laboratorio completo: Zero Trust Security con Istio en AKS

**Duracion estimada:** 5 horas  
**Nivel:** intermedio-avanzado  
**Entorno:** Azure Kubernetes Service (AKS), Azure Container Registry (ACR), Jenkins, kubectl e Istio  
**Aplicacion base sugerida:** `azure-vote-front` + `azure-vote-back`/Redis

---

## 1. Objetivo del laboratorio

En este laboratorio, los participantes tomaran una aplicacion ya desplegada en AKS mediante Jenkins y la protegeran con Istio Service Mesh.

Al finalizar, la arquitectura tendra:

```text
Cliente
  -> Istio Ingress Gateway
  -> Validacion JWT con RequestAuthentication
  -> AuthorizationPolicy basada en claims
  -> azure-vote-front
  -> mTLS STRICT
  -> AuthorizationPolicy service-to-service
  -> azure-vote-back / Redis
```

Durante el laboratorio se implementara:

- Instalacion de Istio sobre AKS.
- Inyeccion de sidecar Envoy en los workloads.
- Publicacion de la aplicacion mediante Istio Gateway y VirtualService.
- Comunicacion interna protegida con mTLS en modo STRICT.
- Microsegmentacion con AuthorizationPolicy.
- Validacion de JWT desde Istio con RequestAuthentication.
- Autorizacion basada en claims del token.
- Pruebas positivas y negativas de seguridad.
- Integracion opcional con Jenkins como Security as Code.

---

## 2. Arquitectura objetivo

```text
GitHub
  -> Jenkins
  -> Docker build
  -> ACR
  -> AKS
  -> Istio Ingress Gateway
  -> Envoy sidecar
  -> Microservicios protegidos
```

El objetivo no es solo desplegar una aplicacion, sino convertirla en una arquitectura Zero Trust:

- Nadie es confiable por defecto.
- Todo trafico debe estar autenticado.
- Todo acceso debe estar autorizado.
- La comunicacion interna debe estar cifrada.
- Las politicas deben ser declarativas y versionables.

---

## 3. Distribucion sugerida para 5 horas

| Bloque | Actividad | Tiempo estimado |
|---|---|---:|
| 1 | Validar AKS y aplicacion existente | 30 min |
| 2 | Instalar Istio y activar sidecars | 45 min |
| 3 | Exponer la app con Istio Gateway | 40 min |
| 4 | Activar mTLS STRICT | 45 min |
| 5 | Crear identidades y AuthorizationPolicy | 60 min |
| 6 | Validar JWT con Istio | 60 min |
| 7 | Pruebas, troubleshooting y debrief | 40 min |

---

## 4. Prerrequisitos

Antes de iniciar, se asume que ya existe un entorno con:

- AKS funcionando.
- `kubectl` conectado al cluster.
- Jenkins desplegando hacia AKS.
- Azure Container Registry operativo.
- Aplicacion `azure-vote` desplegada.
- Deployments similares a:
  - `azure-vote-front`
  - `azure-vote-back`
- Servicios Kubernetes similares a:
  - `azure-vote-front`
  - `azure-vote-back`

Herramientas necesarias:

```bash
az version
kubectl version --client
helm version
python3 --version
```

Si se trabaja desde Azure Cloud Shell, normalmente `az`, `kubectl` y `helm` ya estan disponibles.

---

## 5. Parte 1 - Validar la aplicacion actual

### 5.1 Verificar conexion con AKS

```bash
kubectl get nodes
```

Resultado esperado:

```text
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-xxxxxxxx-vmss000000   Ready    agent   ...   v1.xx.x
```

### 5.2 Verificar deployments

```bash
kubectl get deploy
```

Resultado esperado:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
azure-vote-back    1/1     1            1           ...
azure-vote-front   1/1     1            1           ...
```

### 5.3 Verificar pods

```bash
kubectl get pods
```

Resultado esperado antes de Istio:

```text
NAME                                READY   STATUS    RESTARTS   AGE
azure-vote-back-xxxxxxxxxx-xxxxx    1/1     Running   0          ...
azure-vote-front-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
```

Observacion: antes de habilitar Istio, normalmente cada pod tiene `1/1` contenedores. Despues de inyectar Envoy, deberia tener `2/2`.

### 5.4 Verificar servicios

```bash
kubectl get svc
```

Resultado esperado:

```text
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
azure-vote-back    ClusterIP      ...             <none>          6379/TCP
azure-vote-front   LoadBalancer   ...             x.x.x.x         80:xxxxx/TCP
```

### 5.5 Probar acceso actual

```bash
export APP_IP=<EXTERNAL-IP-ACTUAL>
curl http://$APP_IP
```

Si la aplicacion responde, el punto de partida esta listo.

**Mensaje para explicar en clase:**

> En este momento la aplicacion funciona, pero todavia no tenemos un modelo fuerte de Zero Trust. La comunicacion interna puede estar abierta, no tenemos control declarativo de quien puede hablar con quien, y la validacion de tokens depende de la aplicacion o de algun gateway externo.

---

## 6. Parte 2 - Instalar Istio en AKS

Para este laboratorio se usara `istioctl`, porque permite una instalacion simple, portable y facil de demostrar. En un entorno productivo de AKS tambien podria evaluarse el add-on administrado de Istio.

### 6.1 Descargar Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Entrar al directorio descargado:

```bash
cd istio-*
export PATH=$PWD/bin:$PATH
```

Validar:

```bash
istioctl version
```

### 6.2 Instalar Istio con perfil demo

```bash
istioctl install --set profile=demo -y
```

Validar pods de Istio:

```bash
kubectl get pods -n istio-system
```

Resultado esperado:

```text
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
istiod-xxxxxxxxxx-xxxxx                 1/1     Running   0          ...
```

**Mensaje para explicar en clase:**

> Kubernetes despliega y orquesta contenedores. Istio controla como se comunican. Istio agrega seguridad, control de trafico y observabilidad sin obligarnos a reescribir la aplicacion.

---

## 7. Parte 3 - Activar sidecar injection

### 7.1 Etiquetar el namespace de la aplicacion

Si la aplicacion esta en el namespace `default`:

```bash
kubectl label namespace default istio-injection=enabled --overwrite
```

Validar etiqueta:

```bash
kubectl get namespace default --show-labels
```

Debe aparecer:

```text
istio-injection=enabled
```

### 7.2 Reiniciar deployments

```bash
kubectl rollout restart deployment azure-vote-front
kubectl rollout restart deployment azure-vote-back
```

Esperar a que terminen:

```bash
kubectl rollout status deployment azure-vote-front
kubectl rollout status deployment azure-vote-back
```

### 7.3 Validar sidecars

```bash
kubectl get pods
```

Resultado esperado despues de Istio:

```text
NAME                                READY   STATUS    RESTARTS   AGE
azure-vote-back-xxxxxxxxxx-xxxxx    2/2     Running   0          ...
azure-vote-front-xxxxxxxxxx-xxxxx   2/2     Running   0          ...
```

Validar nombres de contenedores:

```bash
FRONT_POD=$(kubectl get pod -l app=azure-vote-front -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $FRONT_POD -o jsonpath='{.spec.containers[*].name}'
Review: kubectl get pod $FRONT_POD -o jsonpath='{.spec.initContainers[*].name} {.spec.containers[*].name}'; echo
echo
```

Resultado esperado:

```text
azure-vote-front istio-proxy
```

**Mensaje para explicar en clase:**

> El contenedor `istio-proxy` es Envoy. La aplicacion no fue modificada, pero ahora el trafico entra y sale por un proxy controlado por Istio. Esta es la base para aplicar politicas de seguridad desde la plataforma.

---

## 8. Parte 4 - Exponer la aplicacion con Istio Gateway

Ahora se publicara la aplicacion a traves de Istio Ingress Gateway.

### 8.1 Crear Gateway

```bash
cat <<EOF > gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: microservices-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```

Aplicar:

```bash
kubectl apply -f gateway.yaml
```

### 8.2 Crear VirtualService

```bash
cat <<EOF > virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: azure-vote-vs
spec:
  hosts:
  - "*"
  gateways:
  - microservices-gateway
  http:
  - route:
    - destination:
        host: azure-vote-front
        port:
          number: 80
EOF
```

Aplicar:

```bash
kubectl apply -f virtualservice.yaml
```

### 8.3 Obtener IP publica del Istio Gateway

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Guardar la IP:

```bash
export GATEWAY_IP=<EXTERNAL-IP-ISTIO>
```

Probar:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
```

Resultado esperado:

```text
200
```
Review: Si aparece el error 500:

kubectl rollout restart deployment azure-vote-front 

kubectl rollout restart deployment azure-vote-back

**Mensaje para explicar en clase:**

> Ahora el trafico entra por Istio Gateway. Desde este punto podremos controlar autenticacion, autorizacion, trafico y observabilidad desde el mesh.

---

## 9. Parte 5 - Activar mTLS STRICT

mTLS permite cifrado y autenticacion mutua entre servicios. No solo protege la confidencialidad del trafico, tambien permite validar la identidad criptografica de los workloads.

### 9.1 Crear politica mTLS STRICT

```bash
cat <<EOF > mtls-strict.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF
```

Aplicar:

```bash
kubectl apply -f mtls-strict.yaml
```

### 9.2 Validar estado de mTLS

```bash
istioctl authn tls-check
```

Si el comando no muestra suficiente detalle, validar con:

```bash
istioctl proxy-config cluster $FRONT_POD --fqdn azure-vote-back.default.svc.cluster.local
```

**Mensaje para explicar en clase:**

> mTLS no es solo cifrado. Es autenticacion mutua. El cliente sabe con quien habla y el servidor sabe quien lo esta llamando. Esta es una pieza central de Zero Trust para trafico east-west.

---

## 10. Parte 6 - Prueba negativa desde fuera del mesh

Ahora se demostrara que un pod sin sidecar no deberia poder comunicarse correctamente con servicios que exigen mTLS STRICT.

### 10.1 Crear namespace sin Istio

```bash
kubectl create namespace outside
```

Validar que no tenga inyeccion:

```bash
kubectl get namespace outside --show-labels
```

### 10.2 Crear pod de prueba sin sidecar

```bash
kubectl run curl-outside \
  -n outside \
  --image=curlimages/curl \
  --restart=Never \
  --command -- sleep 3600
```

Validar:

```bash
kubectl get pod -n outside
```

### 10.3 Intentar llamada al servicio interno

```bash
kubectl exec -n outside curl-outside -- \
  curl -s -o /dev/null -w "%{http_code}\n" \
  http://azure-vote-front.default.svc.cluster.local
```

Resultado esperado:

```text
000
```

Tambien podria verse un error de conexion o fallo TLS.

**Mensaje para explicar en clase:**

> Antes, cualquier pod dentro del cluster podia intentar llamar al servicio. Con mTLS STRICT, solo workloads dentro del mesh y con identidad valida pueden comunicarse correctamente.

---

## 11. Parte 7 - Crear identidades de servicio

Para aplicar autorizacion granular, los servicios no deberian usar el service account `default`.

### 11.1 Crear service accounts

```bash
kubectl create serviceaccount azure-vote-front-sa
kubectl create serviceaccount azure-vote-back-sa
```

### 11.2 Asociar service accounts a deployments

```bash
kubectl patch deployment azure-vote-front \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"azure-vote-front-sa"}}}}'
```

```bash
kubectl patch deployment azure-vote-back \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"azure-vote-back-sa"}}}}'
```

Esperar rollout:

```bash
kubectl rollout status deployment azure-vote-front
kubectl rollout status deployment azure-vote-back
```

Actualizar variable del pod front:

```bash
FRONT_POD=$(kubectl get pod -l app=azure-vote-front -o jsonpath='{.items[0].metadata.name}')
```

**Mensaje para explicar en clase:**

> En Istio la identidad del workload se basa en el service account de Kubernetes. Si todo usa `default`, no podemos aplicar reglas finas. Con service accounts dedicados podemos diferenciar identidades y aplicar microsegmentacion.

---

## 12. Parte 8 - AuthorizationPolicy: deny by default

La postura Zero Trust recomienda negar por defecto y permitir solo lo necesario.

### 12.1 Crear politica deny-all

```bash
cat <<EOF > deny-all.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec: {}
EOF
```

Aplicar:

```bash
kubectl apply -f deny-all.yaml
```

### 12.2 Probar acceso desde Istio Gateway

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
```

Resultado esperado:

```text
403
```

**Mensaje para explicar en clase:**

> Esta es una postura Zero Trust real: negar todo por defecto. Despues abrimos unicamente los flujos requeridos.

---

## 13. Parte 9 - Permitir trafico desde Istio Gateway hacia front

### 13.1 Identificar service account del ingress gateway

```bash
kubectl get sa -n istio-system
```

En una instalacion estandar con `istioctl`, normalmente sera:

```text
istio-ingressgateway-service-account
```

El principal esperado sera:

```text
cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
```

> Nota: si se usa el add-on administrado de AKS, el namespace y el service account pueden ser diferentes. En ese caso, ajustar el `principal` segun el service account real del ingress gateway.

### 13.2 Crear politica allow desde gateway hacia front

```bash
cat <<EOF > allow-ingress-to-front.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-to-front
  namespace: default
spec:
  selector:
    matchLabels:
      app: azure-vote-front
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
EOF
```

Aplicar:

```bash
kubectl apply -f allow-ingress-to-front.yaml
```

### 13.3 Crear politica allow desde front hacia back

```bash
cat <<EOF > allow-front-to-back.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-front-to-back
  namespace: default
spec:
  selector:
    matchLabels:
      app: azure-vote-back
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/default/sa/azure-vote-front-sa"
EOF
```

Aplicar:

```bash
kubectl apply -f allow-front-to-back.yaml
```

### 13.4 Probar acceso nuevamente

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
```

Resultado esperado:

```text
200
```

**Mensaje para explicar en clase:**

> No abrimos todo el namespace. Solo permitimos dos flujos: gateway hacia front y front hacia backend. Esto es microsegmentacion logica basada en identidad de workload.

---

## 14. Parte 10 - Validar JWT con Istio

Ahora se agregara autenticacion basada en token. Istio validara el JWT antes de permitir acceso al servicio.

Para simplificar el laboratorio, se generara un JWT local con una clave RSA y se insertara el JWKS directamente en la politica. Asi no dependemos de Keycloak ni de un proveedor externo durante la clase.

### 14.1 Instalar dependencias Python

```bash
python3 -m pip install --user pyjwt cryptography jwcrypto
```

Si Cloud Shell muestra advertencias de PATH, normalmente no afectan este laboratorio.

### 14.2 Crear script para generar JWT y JWKS

```bash
cat <<'EOF' > generate-jwt.py
from jwcrypto import jwk
import jwt
import time
import json

ISSUER = "lab@istio.local"
KID = "lab-key-1"

key = jwk.JWK.generate(kty="RSA", size=2048, kid=KID)

private_pem = key.export_to_pem(private_key=True, password=None)
public_jwk = json.loads(key.export(private_key=False))
public_jwk["kid"] = KID

jwks = {"keys": [public_jwk]}

now = int(time.time())

valid_payload = {
    "iss": ISSUER,
    "sub": "student-user",
    "aud": "azure-vote",
    "iat": now,
    "exp": now + 3600,
    "scope": "vote:read",
    "role": "student"
}

invalid_payload = {
    "iss": ISSUER,
    "sub": "guest-user",
    "aud": "azure-vote",
    "iat": now,
    "exp": now + 3600,
    "scope": "wrong:scope",
    "role": "guest"
}

valid_token = jwt.encode(
    valid_payload,
    private_pem,
    algorithm="RS256",
    headers={"kid": KID}
)

invalid_token = jwt.encode(
    invalid_payload,
    private_pem,
    algorithm="RS256",
    headers={"kid": KID}
)

with open("jwks.json", "w") as f:
    json.dump(jwks, f)

with open("valid-token.txt", "w") as f:
    f.write(valid_token)

with open("invalid-scope-token.txt", "w") as f:
    f.write(invalid_token)

print("Generated files:")
print("- jwks.json")
print("- valid-token.txt")
print("- invalid-scope-token.txt")
EOF
```

Ejecutar:

```bash
python3 generate-jwt.py
```

Validar archivos:

```bash
ls -l jwks.json valid-token.txt invalid-scope-token.txt
```

### 14.3 Revisar token valido

```bash
cat valid-token.txt
```

Opcional: copiar el token en `jwt.io` para revisar sus claims.

Claims esperados:

```json
{
  "iss": "lab@istio.local",
  "sub": "student-user",
  "aud": "azure-vote",
  "scope": "vote:read",
  "role": "student"
}
```

**Mensaje para explicar en clase:**

> El payload del JWT no esta cifrado. Cualquiera puede decodificarlo. La seguridad no viene de ocultar el payload, sino de verificar la firma, la expiracion, el issuer, la audience y los claims relevantes.

---

## 15. Parte 11 - Crear RequestAuthentication con JWKS embebido

### 15.1 Preparar JWKS para YAML

```bash
JWKS=$(cat jwks.json | sed 's/"/\\"/g')
```

### 15.2 Crear RequestAuthentication

```bash
cat <<EOF > request-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: azure-vote-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: azure-vote-front
  jwtRules:
  - issuer: "lab@istio.local"
    audiences:
    - "azure-vote"
    jwks: "$JWKS"
EOF
```

Aplicar:

```bash
kubectl apply -f request-authentication.yaml
```

Validar:

```bash
kubectl get requestauthentication
```

**Mensaje para explicar en clase:**

> RequestAuthentication le dice a Istio como validar el JWT: quien es el issuer, cual es la audience esperada y que clave publica debe usar para verificar la firma.

---

## 16. Parte 12 - Exigir JWT valido y scope correcto

La politica anterior permitia gateway hacia front sin exigir JWT. Ahora se reemplazara por una politica mas estricta.

### 16.1 Eliminar politica anterior

```bash
kubectl delete authorizationpolicy allow-ingress-to-front
```

### 16.2 Crear politica que exige JWT y claim `scope=vote:read`

```bash
cat <<EOF > allow-ingress-to-front-with-jwt.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-to-front-with-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: azure-vote-front
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
      requestPrincipals:
      - "lab@istio.local/*"
    when:
    - key: request.auth.claims[scope]
      values:
      - "vote:read"
EOF
```

Aplicar:

```bash
kubectl apply -f allow-ingress-to-front-with-jwt.yaml
```

**Mensaje para explicar en clase:**

> Ya no basta con entrar por el gateway. Ahora el request debe traer un JWT valido, emitido por el issuer esperado, con la audience correcta y con el scope requerido.

---

## 17. Parte 13 - Pruebas de seguridad

### 17.1 Prueba 1: sin token

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP
```

Resultado esperado:

```text
403
```

Explicacion:

```text
No hay JWT.
No hay requestPrincipal.
Istio rechaza el request.
```

### 17.2 Prueba 2: token valido en firma, pero con scope incorrecto

```bash
BAD_TOKEN=$(cat invalid-scope-token.txt)
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $BAD_TOKEN" \
  http://$GATEWAY_IP
```

Resultado esperado:

```text
403
```

Explicacion:

```text
El JWT esta firmado correctamente.
El issuer es valido.
La audience es valida.
Pero el scope no es vote:read.
Istio rechaza el request.
```

### 17.3 Prueba 3: token valido con scope correcto

```bash
VALID_TOKEN=$(cat valid-token.txt)
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $VALID_TOKEN" \
  http://$GATEWAY_IP
```

Resultado esperado:

```text
200
```

### 17.4 Prueba 4: token alterado manualmente

```bash
TAMPERED_TOKEN="${VALID_TOKEN}abc"
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $TAMPERED_TOKEN" \
  http://$GATEWAY_IP
```

Resultado esperado:

```text
401
```

O bien:

```text
403
```

Dependera de como Istio reporte el fallo de autenticacion/autorizacion.

**Mensaje para explicar en clase:**

> Ya tenemos una cadena de seguridad completa: entrada por Istio Gateway, JWT validado por Istio, autorizacion basada en claims, mTLS interno y autorizacion service-to-service.

---

## 18. Parte 14 - Validaciones finales

### 18.1 Ver politicas de autenticacion

```bash
kubectl get requestauthentication
```

Resultado esperado:

```text
NAME             AGE
azure-vote-jwt   ...
```

### 18.2 Ver politicas de autorizacion

```bash
kubectl get authorizationpolicy
```

Resultado esperado:

```text
NAME                              AGE
deny-all                          ...
allow-front-to-back               ...
allow-ingress-to-front-with-jwt   ...
```

### 18.3 Ver politicas mTLS

```bash
kubectl get peerauthentication
```

Resultado esperado:

```text
NAME      MODE     AGE
default   STRICT   ...
```

### 18.4 Analizar configuracion de Istio

```bash
istioctl analyze
```

Resultado esperado:

```text
No validation issues found.
```

### 18.5 Revisar configuracion distribuida al proxy

```bash
FRONT_POD=$(kubectl get pod -l app=azure-vote-front -o jsonpath='{.items[0].metadata.name}')
```

Listeners:

```bash
istioctl proxy-config listeners $FRONT_POD
```

Clusters:

```bash
istioctl proxy-config clusters $FRONT_POD
```

Secrets/certificados:

```bash
istioctl proxy-config secret $FRONT_POD
```

**Mensaje para explicar en clase:**

> En produccion no basta con aplicar YAML. Debemos validar que las politicas realmente llegaron al proxy Envoy y que el comportamiento observado coincide con la intencion de seguridad.

---

## 19. Parte 15 - Integracion opcional con Jenkins

Esta parte puede mostrarse como demo final o dejarse como ejercicio.

La idea es versionar los manifiestos de Istio dentro del repositorio:

```text
repo/
  azure-vote/
  k8s/
  istio/
    gateway.yaml
    virtualservice.yaml
    mtls-strict.yaml
    deny-all.yaml
    allow-front-to-back.yaml
    request-authentication.yaml
    allow-ingress-to-front-with-jwt.yaml
```

En Jenkins, despues del despliegue de la imagen:

```bash
kubectl apply -f istio/gateway.yaml
kubectl apply -f istio/virtualservice.yaml
kubectl apply -f istio/mtls-strict.yaml
kubectl apply -f istio/deny-all.yaml
kubectl apply -f istio/allow-front-to-back.yaml
kubectl apply -f istio/request-authentication.yaml
kubectl apply -f istio/allow-ingress-to-front-with-jwt.yaml
```

### 19.1 Smoke test sin token

```bash
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://$GATEWAY_IP)

if [ "$STATUS" != "403" ]; then
  echo "Security test failed: endpoint should reject unauthenticated requests"
  exit 1
fi
```

### 19.2 Smoke test con token valido

```bash
VALID_TOKEN=$(cat valid-token.txt)

STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $VALID_TOKEN" \
  http://$GATEWAY_IP)

if [ "$STATUS" != "200" ]; then
  echo "Security test failed: valid token should be accepted"
  exit 1
fi
```

**Mensaje para explicar en clase:**

> Ahora la seguridad no se aplica manualmente. Se versiona, se despliega y se prueba dentro del pipeline. Esto es Security as Code aplicado a microservicios.

---

## 20. Resultado esperado final

Al finalizar, la arquitectura queda asi:

```text
Cliente externo
  -> HTTP request con JWT
  -> Istio Ingress Gateway
  -> RequestAuthentication valida firma, issuer y audience
  -> AuthorizationPolicy valida principal y scope
  -> azure-vote-front
  -> mTLS STRICT hacia azure-vote-back
  -> AuthorizationPolicy permite solo front -> back
```

Pruebas esperadas:

| Prueba | Resultado esperado |
|---|---:|
| Sin token | 403 |
| Token alterado | 401 o 403 |
| Token con scope incorrecto | 403 |
| Token valido con `scope=vote:read` | 200 |
| Pod fuera del mesh llamando servicio protegido | Falla o codigo 000 |
| Front llamando a backend | Permitido |

---

## 21. Troubleshooting

### Problema 1: Los pods siguen mostrando `1/1`

Posible causa: el namespace no tiene inyeccion habilitada o los pods no fueron reiniciados.

Solucion:

```bash
kubectl get namespace default --show-labels
kubectl label namespace default istio-injection=enabled --overwrite
kubectl rollout restart deployment azure-vote-front
kubectl rollout restart deployment azure-vote-back
```

### Problema 2: El Gateway no responde

Validar servicio de ingreso:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Validar Gateway y VirtualService:

```bash
kubectl get gateway
kubectl get virtualservice
istioctl analyze
```

### Problema 3: Todo responde 403 despues de deny-all

Es esperado hasta crear las politicas ALLOW.

Validar:

```bash
kubectl get authorizationpolicy
kubectl describe authorizationpolicy allow-ingress-to-front-with-jwt
```

### Problema 4: El principal del ingress gateway no coincide

Validar service accounts:

```bash
kubectl get sa -n istio-system
```

Si el nombre cambia, modificar el principal en la AuthorizationPolicy:

```text
cluster.local/ns/<namespace>/sa/<service-account>
```

### Problema 5: El JWT siempre falla

Validar que el token no haya expirado:

```bash
python3 generate-jwt.py
VALID_TOKEN=$(cat valid-token.txt)
```

Validar que los valores coincidan:

```text
issuer: lab@istio.local
audience: azure-vote
scope: vote:read
```

### Problema 6: Istio no reconoce el JWKS embebido

Recrear el manifiesto:

```bash
JWKS=$(cat jwks.json | sed 's/"/\\"/g')
```

Luego recrear `request-authentication.yaml` y aplicar:

```bash
kubectl apply -f request-authentication.yaml
```

---

## 22. Limpieza del laboratorio

Eliminar politicas:

```bash
kubectl delete authorizationpolicy allow-ingress-to-front-with-jwt --ignore-not-found
kubectl delete authorizationpolicy allow-front-to-back --ignore-not-found
kubectl delete authorizationpolicy deny-all --ignore-not-found
kubectl delete requestauthentication azure-vote-jwt --ignore-not-found
kubectl delete peerauthentication default --ignore-not-found
kubectl delete virtualservice azure-vote-vs --ignore-not-found
kubectl delete gateway microservices-gateway --ignore-not-found
kubectl delete namespace outside --ignore-not-found
```

Quitar etiqueta de inyeccion:

```bash
kubectl label namespace default istio-injection- --overwrite
```

Opcional: desinstalar Istio por completo:

```bash
istioctl uninstall --purge -y
kubectl delete namespace istio-system --ignore-not-found
```

Reiniciar deployments para quitar sidecars:

```bash
kubectl rollout restart deployment azure-vote-front
kubectl rollout restart deployment azure-vote-back
```

---

## 23. Preguntas de cierre para los alumnos

1. Que problema resuelve mTLS que no resuelve solamente HTTPS en el ingreso?
2. Por que no deberiamos usar el service account `default` para todos los microservicios?
3. Que diferencia hay entre autenticacion JWT y autorizacion por claims?
4. Que ventaja tiene aplicar AuthorizationPolicy fuera del codigo de la aplicacion?
5. Como integrarian estas politicas en un pipeline CI/CD real?
6. Que riesgos quedan sin resolver aunque usemos Istio?

---

## 24. Mensaje final del laboratorio

La seguridad en microservicios no debe depender de una sola capa. En este laboratorio se implemento un enfoque de defensa en profundidad:

- El gateway controla la entrada.
- JWT valida identidad de usuario o cliente.
- Los claims limitan lo que se puede hacer.
- mTLS protege comunicacion interna.
- AuthorizationPolicy controla quien puede hablar con quien.
- Jenkins puede desplegar estas politicas como codigo.

La idea principal es clara: en una arquitectura cloud-native moderna, la seguridad debe ser declarativa, automatizable, observable y alineada con Zero Trust.

---

## 25. Referencias recomendadas

- Istio Documentation - Security: https://istio.io/latest/docs/concepts/security/
- Istio Task - Mutual TLS Migration: https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/
- Istio Task - Authorization with JWT: https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/
- Istio Reference - AuthorizationPolicy: https://istio.io/latest/docs/reference/config/security/authorization-policy/
- Microsoft Learn - Istio-based service mesh add-on for AKS: https://learn.microsoft.com/azure/aks/istio-about
- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
