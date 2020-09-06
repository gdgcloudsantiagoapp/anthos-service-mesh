# Guía de Instalación de Anthos Service Mesh en Google Kubernetes Engine

## Decisión de utilizar Anthos Service Mesh para el proyecto GDG Cloud Santiago
Link: https://github.com/gdgcloudsantiagoapp/adr/tree/master/anthos-service-mesh

## Paso 1 - Habilitar las APIs de Google.

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

## Paso 2 - Configurar variables de entorno útiles para la configuración.

La variable CLUSTER_NAME indica el nombre del cluster en GKE y la variable CLUSTER_ZONE indica la zona donde se encontrarán los Nodos del cluster.

Para el caso de la aplicación GDG Cloud Santiago CLUSTER_NAME=prod-cluster y CLUSTER_ZONE=us-east1-b

```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=NOMBRE_DEL_CLUSTER
export CLUSTER_ZONE=NOMBRE_DE_LA_ZONA
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
```

## Paso 3 - Configurar zona de computo basado en la variable CLUSTER_ZONE.

```
gcloud config set compute/zone ${CLUSTER_ZONE}
```

## Paso 4 - Crear Cluster de Google Kubernetes Engine con Stackdriver habilitado.

La variable machine-type corresponde al tipo de máquina de los Nodos que tendrá el cluster, la variable num-nodes corresponde a la cantidad de nodos que tendrá el cluster y la variable subnetwork corresponde a la subred donde correrán los Nodos del cluster. Además se etiqueta el Cluster con la etiqueta mesh_id=MESH_ID.

Para el caso de la aplicación GDG Cloud Santiago machine-type=e2-highcpu-8, num-nodes=1 y subnetwork=default

Para entender más sobre las decisiones en relación al tipo de máquina, número de nodos y subred seleccionada, se pueden visitar las ADR (Architecture Decision Records) en el siguiente repositorio: https://github.com/gdgcloudsantiagoapp/adr/blob/master/google-kubernetes-engine/README.md

```
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=TIPO_DE_MÁQUINA \
    --num-nodes=CANTIDAD_DE_NODOS \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=NOMBRE_DE_LA_SUBRED \
    --labels mesh_id=${MESH_ID}
```

## Paso 5 - Verificar que el usuario actual tenga los permisos según el RBAC en el cluster de Kubernetes para realizar las siguientes operaciones.

```
kubectl auth can-i '*' '*' --all-namespaces
```

## Paso 6 - Crear una cuenta de servicio llamada connect-sa en Google Cloud para posteriormente conectar el cluster con Anthos.

```
gcloud iam service-accounts create connect-sa
```

## Paso 7 - Agregar el rol roles/gkehub.connect a la cuenta de servicio connect-sa.

```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:connect-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
```

## Paso 8 - Crear una llave privada .JSON desde la cuenta de servicio connect-sa.

```
gcloud iam service-accounts keys create connect-sa-key.json \
  --iam-account=connect-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

## Paso 9 - Registrar el cluster con Anthos utilizando la llave privada de la cuenta de servicio de connect-sa.

```
gcloud container hub memberships register ${CLUSTER_NAME}-connect \
   --gke-cluster=${CLUSTER_ZONE}/${CLUSTER_NAME}  \
   --service-account-key-file=./connect-sa-key.json
```

## Paso 10 - Crear una cuenta de servicio para Istio Service Mesh.

```
curl --request POST \
  --header "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data '' \
  https://meshconfig.googleapis.com/v1alpha1/projects/${PROJECT_ID}:initialize
```

## Paso 11 - Descargar el archivo de instalación de Anthos Service Mesh (ASM).

```
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.4.7-asm.0-linux.tar.gz
```
## Paso 12 - Descargar y verificar la firma del instalador.

```
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.4.7-asm.0-linux.tar.gz.1.sig
openssl dgst -verify - -signature istio-1.4.7-asm.0-linux.tar.gz.1.sig istio-1.4.7-asm.0-linux.tar.gz <<'EOF'
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrGCUaJJr1H8a36sG4UUoXvlXvZ
wQfk16sxprI2gOJ2vFFggdq3ixF2h4qNBt0kI7ciDhgpwS8t+/960IsIgw==
-----END PUBLIC KEY-----
EOF
```

## Paso 13 - Extraer el contenido del instalador.

```
tar xzf istio-1.4.7-asm.0-linux.tar.gz
```

## Paso 14 - Moverse al directorio del instalador descomprimido.

```
cd istio-1.4.7-asm.0
```
## Paso 15 - Agregar las herramientas en el directorio /BIN de tu ruta.

```
export PATH=$PWD/bin:$PATH
```

## Paso 16 - Instalar Anthos Service Mesh utilizando istioctl.

```
istioctl manifest apply --set profile=asm \
  --set values.global.trustDomain=${WORKLOAD_POOL} \
  --set values.global.sds.token.aud=${WORKLOAD_POOL} \
  --set values.nodeagent.env.GKE_CLUSTER_URL=https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${CLUSTER_ZONE}/clusters/${CLUSTER_NAME} \
  --set values.global.meshID=${MESH_ID} \
  --set values.global.proxy.env.GCP_METADATA="${PROJECT_ID}|${PROJECT_NUMBER}|${CLUSTER_NAME}|${CLUSTER_ZONE}"
```

## Paso 17 - Esperar hasta que todos los pods dentro del namespace istio-systems se encuentren disponibles.

```
 kubectl wait --for=condition=available --timeout=600s deployment \
    --all -n istio-system
```

## Paso 18 - Verificar que los pods se encuentren disponibles.

```
 kubectl get pod -n istio-system
```

## Paso 19 - Validar la instalación de Anthos Service Mesh utilizando asmctl.

```
 asmctl validate --with-testing-workloads
```
