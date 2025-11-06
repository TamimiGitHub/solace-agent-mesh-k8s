# Solace Agent Mesh on GCP Cloud Run

This directory contains configuration files and deployment manifests for running Solace Agent Mesh on Google Cloud Run.

## Directory Structure

```
gcp-cloudrun/
├── core-sam.yaml           # Cloud Run service definition
├── configs/                # Configuration files directory
│   ├── agents/             # Agent configuration files
│   │   └── main_orchestrator.yaml
│   ├── gateways/           # Gateway configuration files
│   │   └── webui.yaml
│   └── shared_config.yaml  # Shared configuration
```

## Creating ConfigMaps

The Solace Agent Mesh service uses Kubernetes ConfigMaps to mount configuration files into the container. Follow these steps to create the necessary ConfigMap with all configuration files:

### Option 1: Create ConfigMap from all files recursively

This command will create a ConfigMap containing all files in the configs directory, preserving the directory structure:

```bash
kubectl create configmap solace-agent-configs \
  --from-file=configs/
```

### Option 2: Create ConfigMap with specific files and directories

If you need more control over which files are included, you can specify each file or directory:

```bash
kubectl create configmap solace-agent-configs \
  --from-file=agents/main_orchestrator.yaml=configs/agents/main_orchestrator.yaml \
  --from-file=gateways/webui.yaml=configs/gateways/webui.yaml \
  --from-file=shared_config.yaml=configs/shared_config.yaml
```

### Option 3: Create ConfigMap from individual files with custom paths

For complete control over the file paths in the ConfigMap:

```bash
kubectl create configmap solace-agent-configs \
  --from-file=agents/main_orchestrator.yaml=configs/agents/main_orchestrator.yaml \
  --from-file=gateways/webui.yaml=configs/gateways/webui.yaml \
  --from-file=shared_config.yaml=configs/shared_config.yaml
```

## Deploying to GCP Cloud Run

Once you've created the ConfigMap, you can deploy the Solace Agent Mesh service to Cloud Run:

1. **Apply the Cloud Run service definition**:

```bash
kubectl apply -f core-sam.yaml
```

2. **Verify the deployment**:

```bash
kubectl get ksvc solace-agent-mesh
```

3. **Check the logs** (if needed):

```bash
kubectl logs -l serving.knative.dev/service=solace-agent-mesh -c user-container
```

## How It Works

The `core-sam.yaml` file defines a Knative Service that:

1. Uses the `solace-agent-mesh:latest` container image
2. Mounts the `solace-agent-configs` ConfigMap at `/app/configs` in the container
3. Overrides the container's command to run specific configuration files:
   - `/app/configs/agents/main_orchestrator.yaml`
   - `/app/configs/gateways/webui.yaml`
4. Sets necessary environment variables for the service

## Environment Variables Management

The `solace-agent-mesh run` command automatically reads environment variables from a `.env` file located at `/app/.env` in the container. You can simplify environment variable management by including the `.env` file in the same ConfigMap as your configuration files:

1. **Create a .env file** with all your environment variables:

```
SOLACE_DEV_MODE=FALSE
GOOGLE_GENAI_USE_VERTEXAI=TRUE
NAMESPACE=<namespace>
SOLACE_BROKER_URL=<url>
SOLACE_BROKER_VPN=<vpn>
SOLACE_BROKER_USERNAME=<user>
SOLACE_BROKER_PASSWORD=<password>
```

2. **Include the .env file in your ConfigMap**:

```bash
kubectl create configmap solace-agent-configs \
  --from-file=configs/ \
  --from-file=.env=./.env
```

3. **Update the volumeMount in core-sam.yaml** to ensure the .env file is mounted at the correct location:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /app/configs
  - name: config-volume
    mountPath: /app/.env
    subPath: .env
```

This approach eliminates the need to define environment variables inline in the deployment YAML file and allows for easier management of environment-specific configurations.

### Handling Secrets on GCP

For sensitive information like API keys and passwords, use GCP Secret Manager instead of storing them in ConfigMaps:

1. **Create secrets in GCP Secret Manager**:

```bash
# Create a secret
gcloud secrets create solace-broker-password --replication-policy="automatic"

# Add the secret value
echo -n "your-secret-password" | gcloud secrets versions add solace-broker-password --data-file=-
```

2. **Grant access to the service account**:

```bash
# Get the service account used by your Cloud Run service
SERVICE_ACCOUNT=$(gcloud run services describe solace-agent-mesh --format="value(spec.template.spec.serviceAccountName)")

# Grant access to the secret
gcloud secrets add-iam-policy-binding solace-broker-password \
    --member="serviceAccount:$SERVICE_ACCOUNT" \
    --role="roles/secretmanager.secretAccessor"
```

3. **Reference secrets in your .env file or directly in the deployment**:

```
# In your .env file, reference the secret
SOLACE_BROKER_PASSWORD=sm://projects/YOUR_PROJECT_ID/secrets/solace-broker-password/versions/latest
```

Or directly in the core-sam.yaml file:

```yaml
env:
  - name: SOLACE_BROKER_PASSWORD
    valueFrom:
      secretKeyRef:
        name: solace-broker-pw
        key: latest
```

This ensures sensitive information is properly secured and managed according to GCP best practices.

## Updating Configurations

To update the configuration files:

1. **Update the local files** in the `configs/` directory
2. **Delete the existing ConfigMap**:

```bash
kubectl delete configmap solace-agent-configs
```

3. **Create a new ConfigMap** using one of the methods above
4. **Restart the service** to pick up the new configuration:

```bash
kubectl rollout restart deployment $(kubectl get deployment -l serving.knative.dev/service=solace-agent-mesh -o jsonpath="{.items[0].metadata.name}")
```

## Troubleshooting

If you encounter issues with the configuration files not being properly mounted:

1. **Verify the ConfigMap exists**:

```bash
kubectl get configmap solace-agent-configs
```

2. **Check the ConfigMap contents**:

```bash
kubectl describe configmap solace-agent-configs
```

3. **Inspect the pod's mounted volumes**:

```bash
kubectl exec -it $(kubectl get pod -l serving.knative.dev/service=solace-agent-mesh -o jsonpath="{.items[0].metadata.name}") -- ls -la /app/configs