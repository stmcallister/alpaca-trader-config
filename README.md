# Kubernetes Deployment

This directory contains Kubernetes manifests for deploying the Alpaca Schedule Trader with a web UI.

## Files

- `deployment.yaml` - Main deployment configuration
- `service.yaml` - Service to expose the web UI (LoadBalancer type for external access)
- `service-clusterip.yaml` - Alternative ClusterIP service (use with Ingress or port-forward)
- `ingress.yaml` - Ingress resource for domain-based access (optional)
- `configmap.yaml` - Initial trading schedule configuration (optional, can be managed via UI)
- `secret.yaml.example` - Template for API credentials (create your own `secret.yaml`)

## Features

- **Web UI**: Access the web interface to create, update, and delete scheduled trading jobs
- **Dynamic Updates**: Changes made via the web UI are immediately reflected in the scheduler
- **RESTful API**: Full CRUD API for programmatic access

## Deployment Steps

### 1. Create the Secret

**Never hardcode secrets in YAML files!** Use one of these secure methods:

#### Method 1: Using kubectl create secret (Recommended)

Create the secret directly using `kubectl`:

```bash
kubectl create secret generic alpaca-credentials \
  --from-literal=api-key-id='YOUR_ACTUAL_API_KEY' \
  --from-literal=api-secret-key='YOUR_ACTUAL_SECRET_KEY'
```

#### Method 2: Using Environment Variables

If your credentials are already in environment variables:

```bash
export ALPACA_API_KEY_ID='YOUR_ACTUAL_API_KEY'
export ALPACA_API_SECRET_KEY='YOUR_ACTUAL_SECRET_KEY'

kubectl create secret generic alpaca-credentials \
  --from-literal=api-key-id="$ALPACA_API_KEY_ID" \
  --from-literal=api-secret-key="$ALPACA_API_SECRET_KEY"
```

#### Method 3: Using a .env File

Create a `.env` file (make sure it's in `.gitignore`):

```bash
# .env file (DO NOT COMMIT THIS)
ALPACA_API_KEY_ID=your_actual_api_key
ALPACA_API_SECRET_KEY=your_actual_secret_key
```

Then create the secret:

```bash
kubectl create secret generic alpaca-credentials \
  --from-env-file=.env
```

#### Method 4: Using stdin (Most Secure)

For maximum security, you can be prompted for each value:

```bash
kubectl create secret generic alpaca-credentials \
  --from-literal=api-key-id="$(read -s -p 'Enter API Key: ' && echo $REPLY)" \
  --from-literal=api-secret-key="$(read -s -p 'Enter Secret Key: ' && echo $REPLY)"
```

Or use a more interactive approach:

```bash
echo -n "Enter API Key: "
read -s API_KEY
echo
echo -n "Enter Secret Key: "
read -s SECRET_KEY
echo

kubectl create secret generic alpaca-credentials \
  --from-literal=api-key-id="$API_KEY" \
  --from-literal=api-secret-key="$SECRET_KEY"
```

#### Verify the Secret

After creating, verify it exists (values will be base64 encoded):

```bash
kubectl get secret alpaca-credentials
kubectl describe secret alpaca-credentials
```

**Important:** 
- Never commit secrets to version control
- The `secret.yaml.example` file is just a template - don't fill it with real values
- Secrets created with `kubectl create secret` are stored in the cluster

#### Updating an Existing Secret

If you need to update the secret later:

```bash
# Delete the old secret
kubectl delete secret alpaca-credentials

# Create a new one with updated values
kubectl create secret generic alpaca-credentials \
  --from-literal=api-key-id='NEW_API_KEY' \
  --from-literal=api-secret-key='NEW_SECRET_KEY'

# Restart the deployment to pick up the new secret
kubectl rollout restart deployment/alpaca-schedule-trader
```

Or use `kubectl patch` to update individual keys:

```bash
kubectl patch secret alpaca-credentials -p='{"data":{"api-key-id":"'$(echo -n 'NEW_KEY' | base64)'"}}'
```

### 2. Build and Push Docker Image

```bash
# Build the image
docker build -t alpaca-schedule-trader:latest .

# Tag for your registry (replace with your registry)
docker tag alpaca-schedule-trader:latest your-registry/alpaca-schedule-trader:latest

# Push to registry
docker push your-registry/alpaca-schedule-trader:latest
```

### 3. Update Deployment Image

If using a private registry, update the `image` field in `deployment.yaml`:

```yaml
image: your-registry/alpaca-schedule-trader:latest
```

### 4. Deploy to Kubernetes

```bash
# Apply all manifests
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Optional: Apply initial config (or manage via UI)
kubectl apply -f configmap.yaml

# Optional: Apply ingress for domain-based access
# kubectl apply -f ingress.yaml

# Check deployment status
kubectl get deployments
kubectl get pods -l app=alpaca-schedule-trader
kubectl get svc alpaca-schedule-trader

# View logs
kubectl logs -f deployment/alpaca-schedule-trader
```

### 5. Access the Web UI

The web UI is exposed via the Service. Choose one of these access methods:

#### Option 1: LoadBalancer Service (Easiest)

The `service.yaml` is configured with `type: LoadBalancer`. After applying, get the external IP:

```bash
kubectl get svc alpaca-schedule-trader
# Wait for EXTERNAL-IP to be assigned, then access http://<EXTERNAL-IP>
```

**Note:** On local clusters (like minikube, kind, or Docker Desktop), LoadBalancer may not work. In that case:
- Use `service-clusterip.yaml` instead, or
- Use port-forward (Option 2), or
- Use NodePort (Option 3)

#### Option 2: Port Forward (Quick Local Access)

For local development or testing:

```bash
kubectl port-forward svc/alpaca-schedule-trader 8080:80
# Then open http://localhost:8080 in your browser
```

#### Option 3: NodePort Service (Alternative)

If LoadBalancer isn't available, you can change the service type to NodePort:

```bash
kubectl patch svc alpaca-schedule-trader -p '{"spec":{"type":"NodePort"}}'
kubectl get svc alpaca-schedule-trader
# Access via http://<NODE-IP>:<NODE-PORT>
```

#### Option 4: Ingress (Production)

For production with a domain name and SSL:

1. Update `ingress.yaml` with your domain name
2. Apply the ingress:

```bash
kubectl apply -f ingress.yaml
```

3. Update your DNS to point to the ingress controller's IP
4. Access via your domain: `http://alpaca-trader.example.com`

**Note:** Make sure you have an ingress controller installed (e.g., nginx-ingress, traefik).

## Using the Web UI

1. Open the web interface in your browser
2. Click "+ Add New Job" to create a new scheduled trade
3. Fill in the job details:
   - **Job Name**: Unique identifier for the job
   - **Action**: Buy or Sell
   - **Ticker**: Stock symbol (e.g., TSLA, SPY)
   - **Quantity**: Number of shares
   - **Day of Week**: When to execute (e.g., mon-fri, mon, fri)
   - **Hour & Minute**: Time in 24-hour format
4. Click "Save" to add the job
5. Use "Edit" or "Delete" buttons to manage existing jobs

## API Endpoints

The application also exposes a REST API:

- `GET /api/jobs` - Get all scheduled jobs
- `POST /api/jobs` - Create a new job
- `PUT /api/jobs/<job_name>` - Update an existing job
- `DELETE /api/jobs/<job_name>` - Delete a job
- `GET /api/timezone` - Get current timezone
- `PUT /api/timezone` - Update timezone

## Persistent Storage (Optional)

**Note**: By default, schedule changes are stored in the container's filesystem. If the pod restarts, changes will be lost unless you use persistent storage.

To persist configuration across pod restarts, add a PersistentVolumeClaim to the deployment:

```yaml
volumes:
- name: config-storage
  persistentVolumeClaim:
    claimName: alpaca-config-pvc
```

And mount it:

```yaml
volumeMounts:
- name: config-storage
  mountPath: /app
```

## Troubleshooting

- Check pod logs: `kubectl logs -l app=alpaca-schedule-trader`
- Describe pod: `kubectl describe pod -l app=alpaca-schedule-trader`
- Check service: `kubectl describe svc alpaca-schedule-trader`
- Check events: `kubectl get events --sort-by=.metadata.creationTimestamp`
- Test web UI: `curl http://localhost:8080` (after port-forward)
