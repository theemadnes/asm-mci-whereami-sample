# asm-mci-whereami-sample
sample yaml for using whereami across multiple clusters

### setup

already have two clusters
- autopilot-cluster-1
- autopilot-cluster-2

> needed to create autopilot-cluster-3 since 1 and 2 where in the same region 

going to need to set up MCS and ingress-gateway service 
```
$ kubectl get mcs -A -o yaml
apiVersion: v1
items:
- apiVersion: networking.gke.io/v1
  kind: MultiClusterService
  metadata:
    annotations:
      beta.cloud.google.com/backend-config: '{"ports": {"443":"gke-ingress-config"}}'
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"networking.gke.io/v1","kind":"MultiClusterService","metadata":{"annotations":{"beta.cloud.google.com/backend-config":"{\"ports\": {\"443\":\"gke-ingress-config\"}}","networking.gke.io/app-protocols":"{\"https\":\"HTTP2\"}"},"name":"asm-ingressgateway-multicluster-svc-1","namespace":"asm-ingress"},"spec":{"clusters":[{"link":"us-central1/autopilot-cluster-1"},{"link":"us-central1/autopilot-cluster-2"}],"template":{"spec":{"ports":[{"name":"https","port":443,"protocol":"TCP"}],"selector":{"app":"istio-ingressgateway","istio":"ingressgateway"}}}}}
      networking.gke.io/app-protocols: '{"https":"HTTP2"}'
    creationTimestamp: "2023-01-27T05:04:45Z"
    finalizers:
    - mcs.finalizer.networking.gke.io
    generation: 7
    name: asm-ingressgateway-multicluster-svc-1
    namespace: asm-ingress
    resourceVersion: "2287991"
    uid: d663e7ca-8786-49bf-9de9-9b41760c2c19
  spec:
    clusters:
    - link: us-central1/autopilot-cluster-1
    - link: us-central1/autopilot-cluster-2
    template:
      spec:
        ports:
        - name: https
          port: 443
          protocol: TCP
        selector:
          app: istio-ingressgateway
          istio: ingressgateway
kind: List
metadata:
  resourceVersion: ""
```

```
export PROJECT=mc-e2m-01
export IG_NAMESPACE=asm-ingress
export REGION_1=us-central1
export REGION_2=us-east4
export CLUSTER_1=autopilot-cluster-1 # designated as config cluster
export CLUSTER_2=autopilot-cluster-3
export MCI_ENDPOINT=frontend.endpoints.${PROJECT}.cloud.goog

# connect to new cluster
gcloud container clusters get-credentials autopilot-cluster-3 --region us-east4 --project mc-e2m-01

# rename second cluster context
kubectl config rename-context gke_${PROJECT}_${REGION_2}_${CLUSTER_2} ${CLUSTER_2}

```

get fleet status
```
gcloud container fleet mesh describe --project $PROJECT
```

set up ingress gateway on cluster_2
```
kubectl --context=${CLUSTER_2} create namespace ${IG_NAMESPACE}

kubectl --context=${CLUSTER_2} label namespace asm-ingress istio-injection=enabled

# create local self-signed certs for the ingress gateways
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
-subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
-keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
-out frontend.endpoints.${PROJECT}.cloud.goog.crt

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
    networking.gke.io/static-ip: "mcg-ingress-ip"
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
kubectl --context=${CLUSTER_1} apply -f destinationrules/whereami-frontend-cluster-1.yaml
kubectl --context=${CLUSTER_2} apply -f destinationrules/whereami-frontend-cluster-2.yaml
```

and for backend
```
kubectl --context=${CLUSTER_1} apply -f destinationrules/whereami-backend-cluster-1.yaml
kubectl --context=${CLUSTER_2} apply -f destinationrules/whereami-backend-cluster-2.yaml
```

### test with hey

```
hey -c 50 -n 10000 https://$MCI_ENDPOINT
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