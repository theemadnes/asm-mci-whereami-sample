# asm-mci-whereami-sample
sample yaml for using whereami across multiple clusters

### setup

have two clusters
- autopilot-cluster-1
- autopilot-cluster-2

and enable ASM at cluster creation (use the checkbox if doing click-ops

```
export PROJECT=mc-e2m-01
export IG_NAMESPACE=asm-ingress
export REGION_1=us-central1
export REGION_2=us-east4
export CLUSTER_1=autopilot-cluster-1 # designated as config cluster
export CLUSTER_2=autopilot-cluster-2
export MCI_ENDPOINT=frontend.endpoints.${PROJECT}.cloud.goog

# connect to new cluster
gcloud container clusters get-credentials autopilot-cluster-1 --region us-central1 --project mc-e2m-01
gcloud container clusters get-credentials autopilot-cluster-2 --region us-east4 --project mc-e2m-01

# rename second cluster context
kubectl config rename-context gke_${PROJECT}_${REGION_1}_${CLUSTER_1} ${CLUSTER_1}
kubectl config rename-context gke_${PROJECT}_${REGION_2}_${CLUSTER_2} ${CLUSTER_2}

# create static IP (for MCI) and DNS entry

gcloud compute addresses create mc-ingress-ip --global

export GCLB_IP=$(gcloud compute addresses describe mc-ingress-ip --global --format=json | jq -r '.address')
echo ${GCLB_IP}

cat <<EOF > dns-spec.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "frontend.endpoints.${PROJECT}.cloud.goog"
x-google-endpoints:
- name: "frontend.endpoints.${PROJECT}.cloud.goog"
  target: "${GCLB_IP}"
EOF
gcloud endpoints services deploy dns-spec.yaml

```

get fleet status
```
gcloud container fleet mesh describe --project $PROJECT
```
if Multi Cluster Ingress not enabled
```
gcloud container fleet ingress enable --project $PROJECT --config-membership ${CLUSTER_1}
```

set up ingress gateway on clusters
```
kubectl --context=${CLUSTER_1} create namespace ${IG_NAMESPACE}

kubectl --context=${CLUSTER_1} label namespace asm-ingress istio-injection=enabled

kubectl --context=${CLUSTER_2} create namespace ${IG_NAMESPACE}

kubectl --context=${CLUSTER_2} label namespace asm-ingress istio-injection=enabled

# create local self-signed certs for the ingress gateways
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
-subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
-keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
-out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl --context=${CLUSTER_1} -n ${IG_NAMESPACE} create secret tls edge2mesh-credential --key=frontend.endpoints.${PROJECT}.cloud.goog.key --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt


kubectl --context=${CLUSTER_2} -n ${IG_NAMESPACE} create secret tls edge2mesh-credential --key=frontend.endpoints.${PROJECT}.cloud.goog.key --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt


kubectl --context=${CLUSTER_1} apply -f ingress-deployment.yaml
kubectl --context=${CLUSTER_2} apply -f ingress-deployment.yaml


kubectl --context=${CLUSTER_1} apply -f ingress-service.yaml
kubectl --context=${CLUSTER_2} apply -f ingress-service.yaml
```

set up application 
```
# get app namespaces created
kubectl --context=${CLUSTER_1} create ns frontend
kubectl --context=${CLUSTER_1} label namespace frontend istio-injection=enabled
kubectl --context=${CLUSTER_1} create ns backend
kubectl --context=${CLUSTER_1} label namespace backend istio-injection=enabled
kubectl --context=${CLUSTER_2} create ns frontend
kubectl --context=${CLUSTER_2} label namespace frontend istio-injection=enabled
kubectl --context=${CLUSTER_2} create ns backend
kubectl --context=${CLUSTER_2} label namespace backend istio-injection=enabled

mkdir whereami-backend
mkdir whereami-backend/base

cat <<EOF > whereami-backend/base/kustomization.yaml 
bases:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

mkdir whereami-backend/variant

cat <<EOF > whereami-backend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "backend"
EOF

cat <<EOF > whereami-backend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-backend/variant/kustomization.yaml 
nameSuffix: "-backend"
namespace: backend
commonLabels:
  app: whereami-backend
bases:
- ../base
patches:
- cm-flag.yaml
- service-type.yaml
EOF

kubectl --context=${CLUSTER_1} apply -k whereami-backend/variant
kubectl --context=${CLUSTER_2} apply -k whereami-backend/variant

mkdir whereami-frontend
mkdir whereami-frontend/base

cat <<EOF > whereami-frontend/base/kustomization.yaml 
bases:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

mkdir whereami-frontend/variant

cat <<EOF > whereami-frontend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "True" # assuming you don't want a chain of backend calls
  BACKEND_SERVICE:        "http://whereami-backend.backend.svc.cluster.local"
EOF

cat <<EOF > whereami-frontend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-frontend/variant/kustomization.yaml 
nameSuffix: "-frontend"
namespace: frontend
commonLabels:
  app: whereami-frontend
bases:
- ../base
patches:
- cm-flag.yaml
- service-type.yaml
EOF

kubectl --context=${CLUSTER_1} apply -k whereami-frontend/variant
kubectl --context=${CLUSTER_2} apply -k whereami-frontend/variant
```

set up gateways
```

# apply to both clusters 
kubectl --context=${CLUSTER_1} apply -f whereami-mesh/
kubectl --context=${CLUSTER_2} apply -f whereami-mesh/
```

Create SSL Cert
```
gcloud compute ssl-certificates create gke-mc-ingress-cert \
    --domains=frontend.endpoints.${PROJECT}.cloud.goog \
    --global
```
**NOTE:** u can use ManagedCertificate CR, but use the ```networking.gke.io/pre-shared-certs``` annotation and get the name via:
```
kubectl get managedcertificate managed-cert -n whereami -o json | jq '.status.certificateName'
```


fix MCS and MCI
```
mkdir whereami-mci

cat <<EOF > whereami-mci/mci.yaml
apiVersion: networking.gke.io/v1beta1
kind: MultiClusterIngress
metadata:
  name: asm-ingressgateway-multicluster-ingress
  namespace: ${IG_NAMESPACE}
  annotations:
    networking.gke.io/static-ip: "${GCLB_IP}"
    #kubernetes.io/ingress.allow-http: "false" # not recognized
    networking.gke.io/pre-shared-certs: "gke-mc-ingress-cert"
spec:
  template:
    spec:
      backend:
       serviceName: asm-ingressgateway-multicluster-svc-1
       servicePort: 443
      rules:
        - host: 'frontend.endpoints.${PROJECT}.cloud.goog'
          http:
            paths:
            - path: "/"
              backend:
                serviceName: asm-ingressgateway-multicluster-svc-1
                servicePort: 443
EOF


cat <<EOF > whereami-mci/mcs.yaml
apiVersion: networking.gke.io/v1beta1
kind: MultiClusterService
metadata:
  name: asm-ingressgateway-multicluster-svc-1
  namespace: ${IG_NAMESPACE}
  annotations:
    beta.cloud.google.com/backend-config: '{"ports": {"443":"gke-ingress-config"}}'
    networking.gke.io/app-protocols: '{"https":"HTTP2"}' # HTTP/2 with TLS encryption
spec:
  template:
    spec:
      selector:
        app: asm-ingressgateway
      ports:
      - name: https
        protocol: TCP
        port: 443
        targetPort: 8443
  clusters:
  - link: "${REGION_1}/${CLUSTER_1}"
  - link: "${REGION_2}/${CLUSTER_2}"
EOF

kubectl --context=$CLUSTER_1 apply -f whereami-mci/

```

try to switch to automatic management
```
gcloud container fleet mesh update \
    --management automatic \
    --memberships autopilot-cluster-1-1 \
    --project mc-e2m-01

gcloud container fleet mesh update \
    --management automatic \
    --memberships autopilot-cluster-3-1 \
    --project mc-e2m-01

# verify
gcloud container fleet mesh describe --project mc-e2m-01
```

set up locality failover for frontend
```
kubectl --context=${CLUSTER_1} apply -f destinationrules/whereami-frontend.yaml
kubectl --context=${CLUSTER_2} apply -f destinationrules/whereami-frontend.yaml
```

and for backend
```
kubectl --context=${CLUSTER_1} apply -f destinationrules/whereami-backend.yaml
kubectl --context=${CLUSTER_2} apply -f destinationrules/whereami-backend.yaml
```


### test with hey

```
hey -c 50 -n 10000 https://$MCI_ENDPOINT
```

### test with curl forever

```
while true
do
   curl https://$MCI_ENDPOINT
done
```

or 
```
while true; do    curl -s https://$MCI_ENDPOINT; done
```

just get the frontend zone 
```
while true
do
   curl -s https://$MCI_ENDPOINT | jq '.zone'
done
```

or 
```
while true; do    curl -s https://$MCI_ENDPOINT | jq '.zone'; done
```

just get the backend zone 
```
while true
do
   curl -s https://$MCI_ENDPOINT | jq '.backend_result.zone'
done
```

or 
```
while true; do    curl -s https://$MCI_ENDPOINT | jq '.backend_result.zone'; done
```

### scaling resources to `0` and back again `0`


```
# cluster_1 ingress gateway replicas to 0
kubectl --context=${CLUSTER_1} -n asm-ingress patch deployment asm-ingressgateway --subresource='scale' --type='merge' -p '{"spec":{"replicas":0}}'

# back to 3
kubectl --context=${CLUSTER_1} -n asm-ingress patch deployment asm-ingressgateway --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'


# patch cluster_1 frontend deployment replicas to 0
kubectl --context=${CLUSTER_1} -n frontend patch deployment whereami-frontend --subresource='scale' --type='merge' -p '{"spec":{"replicas":0}}'

# back to 3
kubectl --context=${CLUSTER_1} -n frontend patch deployment whereami-frontend --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'

# patch cluster_1 backend deployment replicas to 0
kubectl --context=${CLUSTER_1} -n backend patch deployment whereami-backend --subresource='scale' --type='merge' -p '{"spec":{"replicas":0}}'

# back to 3
kubectl --context=${CLUSTER_1} -n backend patch deployment whereami-backend --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'
```

### enable cloud trace

```
cat <<EOF | kubectl --context=${CLUSTER_1} apply -f -
apiVersion: v1
data:
   mesh: |-
      defaultConfig:
        tracing:
          stackdriver: {}
kind: ConfigMap
metadata:
   name: istio-asm-managed-rapid
   namespace: istio-system
EOF

cat <<EOF | kubectl --context=${CLUSTER_2} apply -f -
apiVersion: v1
data:
   mesh: |-
      defaultConfig:
        tracing:
          stackdriver: {}
kind: ConfigMap
metadata:
   name: istio-asm-managed-rapid
   namespace: istio-system
EOF

# restart proxies
kubectl --context=${CLUSTER_1} rollout restart deployment -n frontend whereami-frontend
kubectl --context=${CLUSTER_2} rollout restart deployment -n frontend whereami-frontend
kubectl --context=${CLUSTER_1} rollout restart deployment -n backend whereami-backend
kubectl --context=${CLUSTER_2} rollout restart deployment -n backend whereami-backend
kubectl --context=${CLUSTER_1} rollout restart deployment -n asm-ingress asm-ingressgateway
kubectl --context=${CLUSTER_2} rollout restart deployment -n asm-ingress asm-ingressgateway
```
