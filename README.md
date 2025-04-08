# vllm-llama3-gcs
Deploy a Llama3.3 70B model on GKE using vllm and GCSFuse for faster loading and avoid downloading model from the internet

## Set up 
### Set the needed environment variables
```
CLUSTER_NAME=ducdo-llama3-gcs
PROJECT_ID=gpu-launchpad-playground
REGION=us-central1
LOCATION=us-central1-c
LOG_BUCKET_NAME=ducdo-vllm
DISK_IMAGE_NAME=ducdo-vllm
```
### Prepare the secondary boot disk image
Create a Cloud Storage bucket to store the execution logs
```
gsutil mb -l $REGION gs://$LOG_BUCKET_NAME --uniform-bucket-level-access
```
Build the **gke-disk-image-builder** tool
```
git clone https://github.com/GoogleCloudPlatform/ai-on-gke.git
cd ai-on-gke/tools/gke-disk-image-builder
go build -o cli ./cli
```
Prepare the secondary boot disk image
```
go run ./cli \
    --project-name=$PROJECT_ID \
    --image-name=$DISK_IMAGE_NAME \
    --zone=$LOCATION \
    --gcs-path=gs://$LOG_BUCKET_NAME \
    --disk-size-gb=100 \
    --container-image=docker.io/vllm/vllm-openai:v0.7.3
```
## Create the cluster
```
gcloud container clusters create $CLUSTER_NAME \
    --project=$PROJECT_ID \
    --region=$REGION \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --release-channel=rapid \
    --num-nodes=1 \
    --image-type="COS_CONTAINERD" \
    --addons=GcePersistentDiskCsiDriver \
    --addons=GcsFuseCsiDriver \
    --enable-image-streaming
```
Or edit an existing cluster
```
gcloud container clusters update $CLUSTER_NAME \
    --region=$REGION \
    --update-addons=GcsFuseCsiDriver=ENABLED,GcePersistentDiskCsiDriver=ENABLED
```
### Create the node pool
Set the needed environment variables
```
NODE_POOL_NAME=ducdo-a3-high4m
ZONE=us-central1-c
```
Create the node pool
```
gcloud container node-pools create $NODE_POOL_NAME \
    --node-locations=$ZONE \
    --enable-autoscaling \
    --total-min-nodes=0 \
    --total-max-nodes=3 \
     --disk-type=pd-ssd \
    --machine-type=a3-highgpu-4g \
    --accelerator type=nvidia-h100-80gb,count=4,gpu-driver-version=LATEST  \
    --cluster=$CLUSTER_NAME \
    --spot \
    --image-type="COS_CONTAINERD" \
    --enable-image-streaming \
    --secondary-boot-disk=disk-image=projects/$PROJECT_ID/global/images/$DISK_IMAGE_NAME,mode=CONTAINER_IMAGE_CACHE \
    --location=$REGION
```
### Set up a Kubernetes ServiceAccount to access the bucket
Create the Kubernetes ServiceAccount
```
KSA_NAME=ksa-ducdo
NAMESPACE=default
```
```
kubectl create serviceaccount $KSA_NAME --namespace $NAMESPACE
```
Grant read-write access to the Kubernetes ServiceAccount in order to access the Cloud Storage bucket
```
MODEL_BUCKET=ducdo-models
PROJECT_NUMBER=604327164091
```
```
gcloud storage buckets add-iam-policy-binding gs://${MODEL_BUCKET} \
  --member "principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${PROJECT_ID}.svc.id.goog/subject/ns/${NAMESPACE}/sa/${KSA_NAME}" \
  --role "roles/storage.objectUser"
```

