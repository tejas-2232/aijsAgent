# Cloud Run Deployment Guide

This guide covers deploying your Next.js application to Google Cloud Run using Docker and Artifact Registry.

## Prerequisites

1. **Google Cloud Project** with billing enabled
2. **gcloud CLI** installed and authenticated
3. **Docker** installed locally (for testing)
4. **Git** repository with your code

## Required IAM Roles and Service Accounts

### 1. Service Account for Cloud Run

Create a service account for your Cloud Run service:

```bash
# Create service account
gcloud iam service-accounts create myjobsearchagent-sa \
    --display-name="My Job Search Agent Service Account" \
    --description="Service account for My Job Search Agent Cloud Run service"

# Get the service account email
SERVICE_ACCOUNT_EMAIL="myjobsearchagent-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com"
```

### 2. Required IAM Roles

Assign the following roles to your service account:

```bash
# Basic Cloud Run roles
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
    --role="roles/run.invoker"

# For accessing other Google Cloud services (if needed)
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
    --role="roles/storage.objectViewer"

# For accessing Secret Manager (if you use it for environment variables)
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
    --role="roles/secretmanager.secretAccessor"
```

### 3. Cloud Build Service Account

Cloud Build needs specific roles to build and deploy:

```bash
# Get the Cloud Build service account
CLOUD_BUILD_SA="YOUR_PROJECT_NUMBER@cloudbuild.gserviceaccount.com"

# Grant necessary roles to Cloud Build
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${CLOUD_BUILD_SA}" \
    --role="roles/run.admin"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${CLOUD_BUILD_SA}" \
    --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member="serviceAccount:${CLOUD_BUILD_SA}" \
    --role="roles/artifactregistry.writer"
```

## Artifact Registry Setup

### 1. Enable Required APIs

```bash
# Enable required APIs
gcloud services enable artifactregistry.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable container.googleapis.com
```

### 2. Create Artifact Registry Repository

```bash
# Set variables
PROJECT_ID="your-project-id"
REGION="us-central1"
REPOSITORY_NAME="myjobsearchagent-repo"

# Create Artifact Registry repository
gcloud artifacts repositories create ${REPOSITORY_NAME} \
    --repository-format=docker \
    --location=${REGION} \
    --description="Docker repository for My Job Search Agent"

# Configure Docker authentication
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

### 3. Verify Artifact Registry Access

Test pushing an image to verify everything works:

```bash
# Build and tag your image
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest .

# Push to Artifact Registry
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest
```

## Environment Variables Setup

### 1. Create Secret Manager Secrets (Recommended)

For sensitive environment variables, use Secret Manager:

```bash
# Create secrets for sensitive data
echo -n "your-firebase-api-key" | gcloud secrets create firebase-api-key --data-file=-
echo -n "your-openai-api-key" | gcloud secrets create openai-api-key --data-file=-
echo -n "your-tavus-api-key" | gcloud secrets create tavus-api-key --data-file=-

# Grant access to Cloud Run service account
gcloud secrets add-iam-policy-binding firebase-api-key \
    --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
    --role="roles/secretmanager.secretAccessor"
```

### 2. Set Build-Time Environment Variables

Update your `cloudbuild.yaml` to use Secret Manager:

```yaml
steps:
  # Get secrets from Secret Manager
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        export NEXT_PUBLIC_FIREBASE_API_KEY=$$(gcloud secrets versions access latest --secret="firebase-api-key")
        export NEXT_PUBLIC_OPENAI_API_KEY=$$(gcloud secrets versions access latest --secret="openai-api-key")
        # Add other secrets as needed
        echo "Secrets retrieved successfully"
    id: 'get-secrets'

  # Build and deploy steps...
```

## Deployment Methods

### Method 1: Using Cloud Build (Recommended)

1. **Set up Cloud Build Trigger**:
   ```bash
   # Create trigger
   gcloud builds triggers create github \
       --repo-name=YOUR_REPO_NAME \
       --repo-owner=YOUR_GITHUB_USERNAME \
       --branch-pattern="^main$" \
       --build-config=ci-cd-cloudrun/cloudbuild.yaml \
       --substitutions=_GAR_LOCATION=${REGION},_REPOSITORY=${REPOSITORY_NAME},_SERVICE_NAME=myjobsearchagent
   ```

2. **Manual deployment**:
   ```bash
   gcloud builds submit \
       --config=ci-cd-cloudrun/cloudbuild.yaml \
       --substitutions=_GAR_LOCATION=${REGION},_REPOSITORY=${REPOSITORY_NAME},_SERVICE_NAME=myjobsearchagent \
       .
   ```

### Method 2: Direct Docker Deployment

```bash
# Build image
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest .

# Push to Artifact Registry
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest

# Deploy to Cloud Run
gcloud run deploy myjobsearchagent \
    --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest \
    --region=${REGION} \
    --platform=managed \
    --allow-unauthenticated \
    --port=8080 \
    --memory=1Gi \
    --cpu=1 \
    --min-instances=0 \
    --max-instances=10 \
    --timeout=300 \
    --service-account=${SERVICE_ACCOUNT_EMAIL}
```

## Configuration Files

### 1. Updated cloudbuild.yaml

Your existing `ci-cd-cloudrun/cloudbuild.yaml` is already well-configured. Make sure to set the substitution variables:

```yaml
substitutions:
  _GAR_LOCATION: 'us-central1'  # Your preferred region
  _REPOSITORY: 'myjobsearchagent-repo'
  _SERVICE_NAME: 'myjobsearchagent'
  # Environment variables (use Secret Manager for sensitive data)
  _NEXT_PUBLIC_FIREBASE_API_KEY: 'your-firebase-api-key'
  # ... other environment variables
```

### 2. Environment Variables in Cloud Run

Set runtime environment variables:

```bash
gcloud run services update myjobsearchagent \
    --region=${REGION} \
    --set-env-vars="NODE_ENV=production,PORT=8080"
```

## Monitoring and Logging

### 1. Enable Cloud Logging

```bash
# View logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=myjobsearchagent" --limit=50
```

### 2. Set up Monitoring

```bash
# Enable monitoring
gcloud services enable monitoring.googleapis.com

# View metrics in Cloud Console
echo "Visit: https://console.cloud.google.com/run/detail/${REGION}/myjobsearchagent/metrics"
```

## Security Best Practices

1. **Use Secret Manager** for sensitive environment variables
2. **Enable VPC Connector** if accessing private resources
3. **Set up Cloud Armor** for DDoS protection
4. **Use HTTPS only** (enabled by default)
5. **Regular security updates** for base images

## Troubleshooting

### Common Issues

1. **Build failures**: Check Cloud Build logs
2. **Permission errors**: Verify IAM roles
3. **Image pull errors**: Check Artifact Registry permissions
4. **Service startup issues**: Check Cloud Run logs

### Useful Commands

```bash
# Check service status
gcloud run services describe myjobsearchagent --region=${REGION}

# View recent logs
gcloud logging read "resource.type=cloud_run_revision" --limit=100

# Test locally
docker run -p 8080:8080 ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/myjobsearchagent:latest
```

## Cost Optimization

1. **Set appropriate CPU and memory limits**
2. **Use min-instances=0** for cost savings
3. **Monitor usage** with Cloud Monitoring
4. **Use preemptible instances** for non-critical workloads

## Next Steps

1. Set up your environment variables
2. Create the Artifact Registry repository
3. Configure IAM roles and service accounts
4. Deploy using your preferred method
5. Set up monitoring and alerts
6. Configure custom domain (optional)

For more information, visit the [Google Cloud Run documentation](https://cloud.google.com/run/docs).

