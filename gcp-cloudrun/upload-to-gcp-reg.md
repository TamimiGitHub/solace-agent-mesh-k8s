# Uploading Images to Google Cloud Artifact Registry

This guide provides step-by-step instructions for uploading Docker images to Google Cloud Artifact Registry for use with Cloud Run.

## Prerequisites

- Google Cloud SDK (gcloud) installed
- Docker installed
- A Docker image you want to upload

## Steps to Upload an Image

### 1. Initialize Google Cloud SDK

If you haven't already configured gcloud, run:

```bash
gcloud init
```

Follow the prompts to authenticate and select your project.

Note: to get the Project ID `gcloud config get-value project`

### 2. Enable the Artifact Registry API

```bash
gcloud services enable artifactregistry.googleapis.com
```

### 3. Create a Docker Repository

```bash
gcloud artifacts repositories create solace-repo \
  --repository-format=docker \
  --location=us-east1 \
  --description="Solace Docker repository"
```

Replace:
- `solace-repo` with your desired repository name
- `us-east1` with your preferred region

### 4. Configure Docker Authentication

```bash
gcloud auth configure-docker us-east1-docker.pkg.dev
```

Replace `us-east1` with the region you selected in the previous step.

### 5. Tag Your Docker Image

```bash
docker tag solace-agent-mesh:latest \
  us-east1-docker.pkg.dev/YOUR_PROJECT_ID/solace-repo/solace-agent-mesh:latest
```

Replace:
- `solace-agent-mesh:latest` with your local image name and tag
- `YOUR_PROJECT_ID` with your Google Cloud project ID
- `solace-repo` with your repository name
- `solace-agent-mesh:latest` with your desired image name and tag in the registry

### 6. Push the Image to Artifact Registry

```bash
docker push us-east1-docker.pkg.dev/YOUR_PROJECT_ID/solace-repo/solace-agent-mesh:latest
```

### 7. Update Your Cloud Run Service Definition

Edit your `core-sam-gcp.yaml` file to use the image from Artifact Registry:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: solace-agent-mesh
spec:
  template:
    spec:
      containers:
        - image: us-east1-docker.pkg.dev/YOUR_PROJECT_ID/solace-repo/solace-agent-mesh:latest
          # Rest of your configuration...
```

## Verifying Your Upload

To verify that your image was uploaded successfully:

```bash
gcloud artifacts docker images list us-east1-docker.pkg.dev/YOUR_PROJECT_ID/solace-repo
```

## Managing Image Access

To grant access to your Artifact Registry repository:

```bash
gcloud artifacts repositories add-iam-policy-binding solace-repo \
  --location=us-east1 \
  --member=serviceAccount:YOUR_SERVICE_ACCOUNT_EMAIL \
  --role=roles/artifactregistry.reader
```

Replace:
- `YOUR_SERVICE_ACCOUNT_EMAIL` with the service account email that needs access

## Best Practices

1. **Use Specific Tags**: Always use specific version tags for production deployments rather than `latest`.

2. **Clean Up Old Images**: Regularly clean up unused images to save storage costs.

3. **Automate Uploads**: Consider setting up CI/CD pipelines to automate image building and uploading.

4. **Use Service Accounts**: For production environments, use dedicated service accounts with minimal permissions.

5. **Enable Vulnerability Scanning**: Enable Container Analysis API to scan your container images for vulnerabilities.