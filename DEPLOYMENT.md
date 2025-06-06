# GKE Deployment Guide

This guide explains how to deploy the Deep Research application to Google Kubernetes Engine (GKE).

## Prerequisites

1. **Google Cloud Project** with billing enabled
2. **GKE Cluster** named `next-demo-cluster` (or customize in cloudbuild.yaml)
3. **Required APIs enabled**:
   ```bash
   gcloud services enable container.googleapis.com
   gcloud services enable cloudbuild.googleapis.com
   gcloud services enable compute.googleapis.com
   gcloud services enable servicenetworking.googleapis.com
   ```

## Initial Setup

### 1. Create GKE Cluster (if not exists)
```bash
gcloud container clusters create next-demo-cluster \
  --zone=us-central1-a \
  --machine-type=e2-standard-2 \
  --num-nodes=3 \
  --enable-autorepair \
  --enable-autoupgrade \
  --workload-pool=$(gcloud config get-value project).svc.id.goog
```

### 2. Configure Cloud Build Permissions
Grant required roles to Cloud Build service account:
```bash
export PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")

gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/container.developer"

gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/container.clusterAdmin"

gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

### 3. Enable Config Connector (for GCP resources)
```bash
gcloud container clusters update next-demo-cluster \
  --zone=us-central1-a \
  --workload-pool=$(gcloud config get-value project).svc.id.goog

kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-config-connector/master/install-bundles/install-bundle-workload-identity/0-cnrm-system.yaml

kubectl create namespace cnrm-system

kubectl annotate namespace cnrm-system cnrm.cloud.google.com/project-id=$(gcloud config get-value project)
```

## Deployment Process

### Automated Deployment via Cloud Build
```bash
# Set your API keys as substitution variables
gcloud builds submit . \
  --substitutions=_GEMINI_API_KEY="your_gemini_api_key",_LANGSMITH_API_KEY="your_langsmith_api_key"
```

### Manual Deployment Steps

1. **Connect to GKE cluster**:
   ```bash
   gcloud container clusters get-credentials next-demo-cluster --zone=us-central1-a
   ```

2. **Deploy infrastructure resources**:
   ```bash
   kubectl apply -f k8s/static-ip-certificate.yaml
   kubectl apply -f k8s/service-account.yaml
   ```

3. **Deploy database services**:
   ```bash
   kubectl apply -f k8s/redis-postgres.yaml
   ```

4. **Create application secrets**:
   ```bash
   kubectl create secret generic deep-research-secrets \
     --from-literal=gemini-api-key="your_gemini_api_key" \
     --from-literal=langsmith-api-key="your_langsmith_api_key"
   ```

5. **Deploy application services**:
   ```bash
   kubectl apply -f k8s/backend-deployment.yaml
   kubectl apply -f k8s/frontend-deployment.yaml
   ```

6. **Deploy ingress**:
   ```bash
   kubectl apply -f k8s/ingress.yaml
   ```

## DNS Configuration

After deployment, configure your DNS to point `deep-research.maoye.demo.altostrat.com` to the static IP:

1. **Get the static IP address**:
   ```bash
   gcloud compute addresses describe deep-research-ip --global --format="value(address)"
   ```

2. **Create DNS A record** pointing your domain to this IP address.

## Architecture Overview

- **Frontend**: React app served via nginx, handles `/app` routes
- **Backend**: LangGraph API service, handles all other routes  
- **Infrastructure**: Redis (caching), PostgreSQL (state persistence)
- **Ingress**: HTTPS with managed SSL certificate, path-based routing
- **Service Account**: Workload Identity for Vertex AI API access

## Monitoring and Troubleshooting

### Check deployment status:
```bash
kubectl get pods,services,ingress
kubectl get managedcertificate
kubectl get computeaddress
```

### View logs:
```bash
kubectl logs -l app=deep-research-backend
kubectl logs -l app=deep-research-frontend
```

### Debug ingress:
```bash
kubectl describe ingress deep-research-ingress
kubectl get managedcertificate deep-research-ssl-cert -o yaml
```

## Scaling and Updates

### Scale deployments:
```bash
kubectl scale deployment deep-research-backend --replicas=2
kubectl scale deployment deep-research-frontend --replicas=2
```

### Update images:
```bash
kubectl set image deployment/deep-research-backend backend=gcr.io/PROJECT_ID/deep-research-backend:v2
kubectl set image deployment/deep-research-frontend frontend=gcr.io/PROJECT_ID/deep-research-frontend:v2
```

## Cleanup

To remove all resources:
```bash
kubectl delete -f k8s/
gcloud compute addresses delete deep-research-ip --global
```

## Security Notes

- Service account uses Workload Identity for secure GCP API access
- Secrets are stored in Kubernetes secrets (consider using Secret Manager)
- HTTPS enforced via managed certificate
- Minimal resource requests for cost optimization
- Network policies can be added for additional security