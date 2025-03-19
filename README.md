# no-proxy-01
setting up CSM with gRPC proxyless 

using two gRPC services (frontend and backend) to test [CSM gRPC proxyless](https://cloud.google.com/service-mesh/docs/gateway/proxyless-grpc-mesh#python_1)

in this initial version, goal is just to set up 2 clusters each hosting a single gRPC service (frontend and backend, and verifying that locality is working)

### create clusters

```
export PROJECT_ID=no-proxy-01
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
gcloud config set project ${PROJECT_ID}
export REGION=us-central1
export CLUSTER_1=frontend
export CLUSTER_2=backend

# enable APIs (probably too many here)
gcloud services enable \
  container.googleapis.com \
  mesh.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  trafficdirector.googleapis.com 

gcloud container clusters create-auto --async \
${CLUSTER_1} --region ${REGION} \
--release-channel rapid --labels mesh_id=proj-${PROJECT_NUMBER} \
--enable-private-nodes --enable-fleet

gcloud container clusters create-auto --async \
${CLUSTER_2} --region ${REGION} \
--release-channel rapid --labels mesh_id=proj-${PROJECT_NUMBER} \
--enable-private-nodes --enable-fleet
```

### mesh setup 

```
# once clusters are ready (check console or CLI for status), get creds and add to kube context 
gcloud container clusters get-credentials ${CLUSTER_1} --location ${REGION} --project ${PROJECT_ID}
gcloud container clusters get-credentials ${CLUSTER_2} --location ${REGION} --project ${PROJECT_ID}

# rename the kube contexts for readability
kubectl config rename-context gke_${PROJECT_ID}_${REGION}_${CLUSTER_1} ${CLUSTER_1}
kubectl config rename-context gke_${PROJECT_ID}_${REGION}_${CLUSTER_2} ${CLUSTER_2}

# enable mesh for the project
gcloud container fleet mesh enable --project ${PROJECT_ID}

# update the clusters to use Gateway API for mesh
gcloud alpha container fleet mesh update \
--config-api gateway \
--memberships ${CLUSTER_1} \
--project ${PROJECT_ID}

gcloud alpha container fleet mesh update \
--config-api gateway \
--memberships ${CLUSTER_2} \
--project ${PROJECT_ID}

# install gRPC Gateway CRDs
curl https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml \
| kubectl --context ${CLUSTER_1} apply -f -

curl https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml \
| kubectl --context ${CLUSTER_2} apply -f -

# enable permissions for workloads to act as a Traffic Director client
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member "group:${PROJECT_ID}.svc.id.goog:/allAuthenticatedUsers/" \
    --role "roles/trafficdirector.client"
```

### create workloads 

```
# create namespaces and enable [Proxyless Bootstrap Injector](https://cloud.google.com/service-mesh/docs/gateway/proxyless-grpc-mesh#set_up_the_proxyless_grpc_service)
# frontend
kubectl --context $CLUSTER_1 apply -f - <<EOF
---
kind: Namespace
apiVersion: v1
metadata:
 name: frontend
EOF
kubectl --context $CLUSTER_1 label namespace frontend mesh.cloud.google.com/csm-injection=proxyless
# backend
kubectl --context $CLUSTER_2 apply -f - <<EOF
---
kind: Namespace
apiVersion: v1
metadata:
 name: backend
EOF
kubectl --context $CLUSTER_2 label namespace backend mesh.cloud.google.com/csm-injection=proxyless

# deploy workloads
# backend first
kubectl --context $CLUSTER_2 apply -k backend/variant/

# set up HTTPRoute in backend cluster
kubectl --context $CLUSTER_2 apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: whereami-grpc-backend
  namespace: backend
spec:
  parentRefs:
  - name: whereami-grpc-backend
    namespace: backend
    kind: Service
    group: ""
  rules:
  - backendRefs:
    - name: whereami-grpc-backend
      port: 9090
EOF

# then frontend
kubectl --context $CLUSTER_1 apply -k frontend/variant/
```

### oh no

realized that gateway API is single cluster only, if memory serves, so no x-cluster comms are happening!
```
$ grpcurl -plaintext 34.135.122.113:9090 whereami.Whereami.GetPayload
{
  "cluster_name": "frontend",
  "metadata": "backend",
  "node_name": "gk3-frontend-nap-1djjtomd-95900ceb-hsrp",
  "pod_ip": "10.108.129.9",
  "pod_name": "whereami-grpc-frontend-68c9655c9f-qkvgn",
  "pod_name_emoji": "👵🏻",
  "pod_namespace": "frontend",
  "pod_service_account": "whereami-grpc-frontend",
  "project_id": "no-proxy-01",
  "timestamp": "2025-03-19T04:36:18",
  "zone": "us-central1-c",
  "gce_instance_id": "3680379306077845076",
  "gce_service_account": "no-proxy-01.svc.id.goog"
}
```

OK, let's try to get the frontend stuff set up on the backend cluster and see if that works

```
# frontend
kubectl --context $CLUSTER_2 apply -f - <<EOF
---
kind: Namespace
apiVersion: v1
metadata:
 name: frontend
EOF
kubectl --context $CLUSTER_2 label namespace frontend mesh.cloud.google.com/csm-injection=proxyless

# set up HTTPRoute in backend cluster
kubectl --context $CLUSTER_2 apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: whereami-grpc-backend
  namespace: backend
spec:
  parentRefs:
  - name: whereami-grpc-frontend
    namespace: frontend
    kind: Service
    group: ""
  rules:
  - backendRefs:
    - name: whereami-grpc-backend
      port: 9090
EOF

# then frontend
kubectl --context $CLUSTER_2 apply -k frontend/variant/
```

Yep - single cluster works!
```
$ grpcurl -plaintext 34.71.248.188:9090 whereami.Whereami.GetPayload
{
  "backend_result": {
    "cluster_name": "backend",
    "metadata": "backend",
    "node_name": "gk3-backend-nap-1fr11neh-c6c9dc2a-w7tk",
    "pod_ip": "10.8.129.25",
    "pod_name": "whereami-grpc-backend-57dcd49d9c-jwsv2",
    "pod_name_emoji": "👩🏼‍❤️‍💋‍👩🏼",
    "pod_namespace": "backend",
    "pod_service_account": "whereami-grpc-backend",
    "project_id": "no-proxy-01",
    "timestamp": "2025-03-19T05:13:26",
    "zone": "us-central1-c",
    "gce_instance_id": "7339505044984166567",
    "gce_service_account": "no-proxy-01.svc.id.goog"
  },
  "cluster_name": "backend",
  "metadata": "frontend",
  "node_name": "gk3-backend-nap-1fr11neh-c6c9dc2a-w7tk",
  "pod_ip": "10.8.129.23",
  "pod_name": "whereami-grpc-frontend-68c9655c9f-dpfn4",
  "pod_name_emoji": "👩🏼‍❤‍👨🏻",
  "pod_namespace": "frontend",
  "pod_service_account": "whereami-grpc-frontend",
  "project_id": "no-proxy-01",
  "timestamp": "2025-03-19T05:13:26",
  "zone": "us-central1-c",
  "gce_instance_id": "7339505044984166567",
  "gce_service_account": "no-proxy-01.svc.id.goog"
}
```