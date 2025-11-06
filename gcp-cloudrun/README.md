# Solace Agent Mesh on GCP Cloud Run

This document provides Google Cloud CLI (`gcloud`) commands for deploying and managing Solace Agent Mesh on Google Cloud Run.

## Directory Structure

```
gcp-cloudrun/
├── core-sam.yaml       # Cloud Run service definition
├── configs/                # Configuration files directory
│   ├── agents/             # Agent configuration files
│   │   └── main_orchestrator.yaml
│   ├── gateways/           # Gateway configuration files
│   │   └── webui.yaml
│   └── shared_config.yaml  # Shared configuration
```

## Managing Configuration Files

Cloud Run requires specific approaches for configuration management:

### Option 1: Using Cloud Storage Buckets

1. **Create a Cloud Storage bucket**:

```bash
gsutil mb gs://solace-agent-configs
```

2. **Upload configuration files to the bucket**:

```bash
gsutil cp -r configs/* gs://solace-agent-configs/
```

3. **Create a service account for your Cloud Run service** (if you don't have one already):

```bash
# Create a service account
gcloud iam service-accounts create solace-agent-sa \
  --display-name="Solace Agent Service Account"

# Grant access to the bucket
gsutil iam ch serviceAccount:solace-agent-sa@PROJECT_ID.iam.gserviceaccount.com:objectViewer gs://solace-agent-configs
```

Note: Replace `PROJECT_ID` in all commands with your actual Google Cloud project ID.

### Option 2: Using Secret Manager for Configuration Files

For configuration files that may contain sensitive information:

```bash
# Create a secret for each configuration file
gcloud secrets create main-orchestrator-config --replication-policy="automatic"
cat configs/agents/main_orchestrator.yaml | gcloud secrets versions add main-orchestrator-config --data-file=-

gcloud secrets create webui-config --replication-policy="automatic"
cat configs/gateways/webui.yaml | gcloud secrets versions add webui-config --data-file=-

gcloud secrets create shared-config --replication-policy="automatic"
cat configs/shared_config.yaml | gcloud secrets versions add shared-config --data-file=-

# Grant access to the service account
gcloud secrets add-iam-policy-binding main-orchestrator-config \
  --member="serviceAccount:solace-agent-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding webui-config \
  --member="serviceAccount:solace-agent-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding shared-config \
  --member="serviceAccount:solace-agent-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Deploying to GCP Cloud Run

1. **Deploy using the Cloud Run compatible service definition**:

```bash
gcloud run services replace core-sam-gcp.yaml
```

2. **Verify the deployment**:

```bash
gcloud run services describe solace-agent-mesh
```

### Handling Secrets on GCP

For sensitive information like API keys and passwords:

1. **Create secrets in GCP Secret Manager**:

```bash
# Create a secret
gcloud secrets create solace-broker-password --replication-policy="automatic"

# Add the secret value
echo -n "your-secret-password" | gcloud secrets versions add solace-broker-password --data-file=-
```

2. **Grant access to the service account**:

```bash
# Grant access to the secret
gcloud secrets add-iam-policy-binding solace-broker-password \
  --member="serviceAccount:solace-agent-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```


## Updating Configurations

To update the configuration files:

1. **Update the local files** in the `configs/` directory

2. **Upload the updated files to Cloud Storage**:

```bash
gsutil cp -r configs/* gs://solace-agent-configs/
```

3. **Restart the service** to pick up the new configuration:

```bash
gcloud run services update solace-agent-mesh --region=us-central1
```

## Troubleshooting

If you encounter issues with the configuration files:

1. **Verify the Cloud Storage bucket contents**:

```bash
gsutil ls -r gs://solace-agent-configs/
```

2. **Check the service logs**:

```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=solace-agent-mesh" --limit=50
```

3. **Test the service directly**:

```bash
# Get the service URL
SERVICE_URL=$(gcloud run services describe solace-agent-mesh --format="value(status.url)")

# Test the service
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

4. **Check service configuration**:

```bash
gcloud run services describe solace-agent-mesh
```

## Important Notes

1. **Service Account Setup**: Make sure to create and configure the service account **before** deploying your Cloud Run service.

2. **Project ID**: Replace `PROJECT_ID` in all commands with your actual Google Cloud project ID.

3. **Region**: The examples use `us-central1` as the region. Change this to your preferred region.

4. **First Deployment**: When deploying for the first time, you'll need to create resources in this order:
   - Service account
   - Cloud Storage bucket or Secret Manager secrets
   - Grant permissions
   - Deploy the Cloud Run service

5. **Configuration Management**: Cloud Run requires configuration files to be:
   - Packaged into your container image, or
   - Stored in Cloud Storage buckets, or
   - Managed via Secret Manager for sensitive configurations