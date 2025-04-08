# vllm-llama3-gcs
Deploy a Llama3.3 70B model on GKE using vllm and Llama 3.3 70B model. In this lab, we will use secondary boot disk for caching the vllm image and GCSFuse for caching the Llama 3.3 model. These techniques help improving faster loading and avoid downloading model directly from the internet.

Notes: for even faster model loading time, consider using Hyperdisk ML option.

## Set up 
### Set the needed environment variables
```
CLUSTER_NAME=ducdo-llama3-gcs
PROJECT_ID=gpu-launchpad-playground
REGION=us-central1
```
### Prepare the secondary boot disk image
Create a Cloud Storage bucket to store the execution logs
```
LOG_BUCKET_NAME=ducdo-vllm
```
```
gsutil mb -l $REGION gs://$LOG_BUCKET_NAME
```
Build the **gke-disk-image-builder** tool
```
git clone https://github.com/GoogleCloudPlatform/ai-on-gke.git
cd ai-on-gke/tools/gke-disk-image-builder
go build -o cli ./cli
```
Prepare the secondary boot disk image

Notes: feel free to change to vllm-openai to the later version (e.g. 0.8.3)
```
DISK_IMAGE_NAME=ducdo-vllm
ZONE=us-central1-c
```
```
go run ./cli \
    --project-name=$PROJECT_ID \
    --image-name=$DISK_IMAGE_NAME \
    --zone=$ZONE \
    --gcs-path=gs://$LOG_BUCKET_NAME \
    --disk-size-gb=100 \
    --container-image=docker.io/vllm/vllm-openai:v0.7.3
```
### Download the LLM model to a GCS bucket
Create the bucket
```
MODEL_BUCKET=ducdo-llm-models
```
```
gsutil mb -l $REGION gs://$MODEL_BUCKET
```
### Set up a Kubernetes ServiceAccount to access the bucket that will has/has the downloaded model
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
PROJECT_NUMBER=604327164091
```
```
gcloud storage buckets add-iam-policy-binding gs://${MODEL_BUCKET} \
  --member "principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${PROJECT_ID}.svc.id.goog/subject/ns/${NAMESPACE}/sa/${KSA_NAME}" \
  --role "roles/storage.objectUser"
```
### Create a GKE job to download the LLM model to GCS
Create a nodepool with 48 vCPU
```
CPU_NODE_POOL_NAME=cpu48
ZONE=us-central1-c
```
```
gcloud container node-pools create $CPU_NODE_POOL_NAME \
    --node-locations=$ZONE \
    --enable-autoscaling \
    --total-min-nodes=0 \
    --total-max-nodes=3 \
    --machine-type=c4-standard-48 \
    --cluster=$CLUSTER_NAME \
    --image-type="COS_CONTAINERD" \
    --location=$REGION
```
Create a GKE job to download the LLM model
Create a secret for HF token
```
HF_TOKEN=
```
```
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=$HF_TOKEN \
    --namespace $NAMESPACE
```
Job configuration file producer-job.yaml
```
apiVersion: batch/v1
kind: Job
metadata:
  name: producer-job
spec:
  template:  # Template for the Pods the Job will create
    metadata:
      annotations:
        gke-gcsfuse/volumes: "true"
        gke-gcsfuse/cpu-limit: "0"
        gke-gcsfuse/memory-limit: "0"
        gke-gcsfuse/ephemeral-storage-limit: "0"
    spec:
      serviceAccountName: ksa-ducdo
      containers:
      - name: copy
        resources:
          requests:
            cpu: "32"
          limits:
            cpu: "32"
        image: huggingface/downloader:0.17.3
        command: [ "huggingface-cli" ]
        args:
        - download
        - meta-llama/Llama-3.3-70B-Instruct
        - --local-dir=/data/Llama-3.3-70B-Instruct
        - --local-dir-use-symlinks=False
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: hf_api_token
        volumeMounts:
        - name: gcs-fuse-csi-ephemeral
          mountPath: /data
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: gke-gcsfuse-cache
        emptyDir:
          medium: Memory
      - name: dshm
        emptyDir:
          medium: Memory
      - name: gcs-fuse-csi-ephemeral
        csi:
          driver: gcsfuse.csi.storage.gke.io
          volumeAttributes:
            bucketName: ducdo-llm-models
            mountOptions: "implicit-dirs,file-cache:enable-parallel-downloads:true,file-cache:parallel-downloads-per-file:100,file-cache:max-parallel-downloads:-1,file-cache:download-chunk-size-mb:10,file-cache:max-size-mb:-1"
      restartPolicy: Never
  parallelism: 1         # Run 1 Pods concurrently
  completions: 1         # Once 1 Pods complete successfully, the Job is done
  backoffLimit: 4        # Max retries on failure
```
```
kubectl apply -f producer-job.yaml
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
    --addons=GcsFuseCsiDriver \
    --enable-image-streaming
```
Or edit an existing cluster to enable the GCS Fuse support
```
gcloud container clusters update $CLUSTER_NAME \
    --region=$REGION \
    --update-addons=GcsFuseCsiDriver=ENABLED
```
### Create the node pool
Set the needed environment variables
```
NODE_POOL_NAME=ducdo-a3-high4m
ZONE=us-central1-c
```
Create the node pool with the configuration of secondary boot disk
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
## Deployment and service
Create the deployment file vllm-llama3-70b.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-gcs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-gcs
  template:
    metadata:
      labels:
        app: vllm-gcs
      annotations:
        gke-gcsfuse/volumes: "true"
        gke-gcsfuse/cpu-limit: "0"
        gke-gcsfuse/memory-limit: "0"
        gke-gcsfuse/ephemeral-storage-limit: "0"
    spec:
      serviceAccountName: ksa-ducdo
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-h100-80gb
      containers:
      - name: vllm-gcs
        image: vllm/vllm-openai:v0.7.3
        env:
        - name: VLLM_XLA_CACHE_PATH
          value: "/data"
        command:
          - sh
          - -c
          - "USE_VLLM_V1=1 vllm serve '/data/Llama-3.3-70B-Instruct' --tensor-parallel-size 4 --max_model_len 128000 --port 8080 --gpu-memory-utilization 0.95 --trust-remote-code"
        resources:
          limits:
            nvidia.com/gpu: "4"
        ports:
          - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        volumeMounts:
        - name: gcs-fuse-csi-ephemeral
          mountPath: /data
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: gke-gcsfuse-cache
        emptyDir:
          medium: Memory
      - name: dshm
        emptyDir:
          medium: Memory
      - name: gcs-fuse-csi-ephemeral
        csi:
          driver: gcsfuse.csi.storage.gke.io
          volumeAttributes:
            bucketName: ducdo-llm-models
            mountOptions: "implicit-dirs,file-cache:enable-parallel-downloads:true,file-cache:parallel-downloads-per-file:100,file-cache:max-parallel-downloads:-1,file-cache:download-chunk-size-mb:10,file-cache:max-size-mb:-1"
```
```
kubectl apply -f vllm-llama3-70b.yaml
```
Create the service file llm-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: llm-service
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: vllm-gcs
  type: ClusterIP
```
```
kubectl apply -f llm-service.yaml
```
Test the service
```
kubectl port-forward svc/llm-service 8080:8080
```
On another console
```
curl http://localhost:8080/v1/completions -H "Content-Type: application/json" -d '{
    "model": "/data/Llama-3.3-70B-Instruct",
    "prompt": "Where can I find the best biryani in India?",    
    "max_tokens": 1024,
    "temperature": 0
}'
```
