# anthos-service-mesh
Anthos Service Mesh Installation Guide

STEP 1 - ENABLE GOOGLE APIS

```
gcloud services enable \
    container.googleapis.com \
    compute.googleapis.com \
    stackdriver.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    anthos.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com
```

STEP 2 - SET INSTALLATION VARIABLES

```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=prod-cluster
export CLUSTER_ZONE=us-east1-b
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
```

STEP 3 - SET COMPUTE ZONE

```
gcloud config set compute/zone ${CLUSTER_ZONE}
```

STEP 4 - CREATE GKE CLUSTER ( 1 NODE E2-HIGHCPU-8 / ANTHOS SERVICE MESH ENABLE)

```
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=e2-highcpu-8 \
    --num-nodes=1 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --labels mesh_id=${MESH_ID}
```

STEP 5 - CHECK RBAC PERMISSIONS

```
kubectl auth can-i '*' '*' --all-namespaces
```

STEP 6 - CREATE SERVICE ACCOUNT

```
gcloud iam service-accounts create connect-sa
```

STEP 7 - ADD GKE HUB ROLE TO SERVICE ACCOUNT

```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:connect-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
```

STEP 8  - CREATE .JSON KEY FROM SERVICE ACCOUNT

```
gcloud iam service-accounts keys create connect-sa-key.json \
  --iam-account=connect-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

STEP 9 - REGISTER CLUSTER TO ANTHOS

```
gcloud container hub memberships register ${CLUSTER_NAME}-connect \
   --gke-cluster=${CLUSTER_ZONE}/${CLUSTER_NAME}  \
   --service-account-key-file=./connect-sa-key.json
```

STEP 10 - CREATE ISTIO SERVICE ACCOUNT

```
curl --request POST \
  --header "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data '' \
  https://meshconfig.googleapis.com/v1alpha1/projects/${PROJECT_ID}:initialize
```

STEP 11 - DONWLOAD ANTHOS SERVICE MESH INSTALLATION FILE

```
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.4.7-asm.0-linux.tar.gz
```

STEP 12 - DOWNLOAD AND VERIFY SIGNATURE

```
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.4.7-asm.0-linux.tar.gz.1.sig
openssl dgst -verify - -signature istio-1.4.7-asm.0-linux.tar.gz.1.sig istio-1.4.7-asm.0-linux.tar.gz <<'EOF'
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrGCUaJJr1H8a36sG4UUoXvlXvZ
wQfk16sxprI2gOJ2vFFggdq3ixF2h4qNBt0kI7ciDhgpwS8t+/960IsIgw==
-----END PUBLIC KEY-----
EOF
```

STEP 13 - EXTRACT THE CONTENT OF THE FILE

```
tar xzf istio-1.4.7-asm.0-linux.tar.gz
```

STEP 14 - CHANGE TO DIRECTORY

```
cd istio-1.4.7-asm.0
```

STEP 15 - ADD THE TOOLS IN THE /BIN DIRECTORY OF YOUR PATH

```
export PATH=$PWD/bin:$PATH
```

STEP 16 - INSTALL ANTHOS SERVICE MESH WITH ISTIO

```
istioctl manifest apply --set profile=asm \
  --set values.global.trustDomain=${WORKLOAD_POOL} \
  --set values.global.sds.token.aud=${WORKLOAD_POOL} \
  --set values.nodeagent.env.GKE_CLUSTER_URL=https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${CLUSTER_ZONE}/clusters/${CLUSTER_NAME} \
  --set values.global.meshID=${MESH_ID} \
  --set values.global.proxy.env.GCP_METADATA="${PROJECT_ID}|${PROJECT_NUMBER}|${CLUSTER_NAME}|${CLUSTER_ZONE}"
```

 STEP 17 - CHECK THE COMPLETION OF THE INSTALLATION

```
 kubectl wait --for=condition=available --timeout=600s deployment \
    --all -n istio-system
```

 STEP 18 - CHECK THAT THE CONTROL PLANE PODS ARE RUNNING

```
 kubectl get pod -n istio-system
```

 STEP 19  - VALIDATE THE INSTALLATION

```
 asmctl validate --with-testing-workloads
```