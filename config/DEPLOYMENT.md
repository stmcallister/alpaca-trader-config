# Kubernetes Deployment Guide

This guide explains how to deploy the Alpaca Schedule Trader application to a Kubernetes cluster.

## Prerequisites

- A Kubernetes cluster (v1.20+)
- `kubectl` configured to access your cluster
- Docker images built and pushed to a registry accessible by your cluster
- Alpaca API credentials

## Deployment Steps

### 1. Build and Push Docker Images

You need to build two Docker images (or one image that can run both commands):

```bash
# Build schedule image
docker build -f DOCKERFILE.schedule -t YOUR_REGISTRY/alpaca-schedule-trader:schedule-latest .

# Build worker image  
docker build -f DOCKERFILE.worker -t YOUR_REGISTRY/alpaca-schedule-trader:worker-latest .

# Or build a single image that can run both (recommended)
docker build -t YOUR_REGISTRY/alpaca-schedule-trader:latest .

# Push to registry
docker push YOUR_REGISTRY/alpaca-schedule-trader:latest
```

### 2. Update Image References

Edit `config/k8s.yaml` and replace `YOUR_DOCKER_REGISTRY/alpaca-schedule-trader:latest` with your actual image path in both the `schedule` and `worker` deployments.

### 3. Create Secrets

Create the required Kubernetes secrets:

```bash
# Temporal database secrets
kubectl create secret generic temporal-secrets \
  --from-literal=postgres-user=temporal \
  --from-literal=postgres-password=YOUR_SECURE_PASSWORD \
  --namespace=alpaca-trader

# Alpaca API secrets
kubectl create secret generic alpaca-secrets \
  --from-literal=api-key-id=YOUR_ALPACA_API_KEY_ID \
  --from-literal=api-secret-key=YOUR_ALPACA_API_SECRET_KEY \
  --from-literal=paper=true \
  --namespace=alpaca-trader
```

**Note:** Replace `YOUR_SECURE_PASSWORD`, `YOUR_ALPACA_API_KEY_ID`, and `YOUR_ALPACA_API_SECRET_KEY` with your actual values.

### 4. Deploy to Kubernetes

Apply the manifests:

```bash
kubectl apply -f config/k8s.yaml
```

### 5. Verify Deployment

Check that all pods are running:

```bash
kubectl get pods -n alpaca-trader
```

You should see:
- `postgres-*` (1 pod)
- `temporal-server-*` (1 pod)
- `temporal-ui-*` (1 pod)
- `schedule-*` (1 pod)
- `worker-*` (2 pods)

### 6. Access Temporal UI

The Temporal UI is exposed via ngrok operator with OAuth authentication.

**Prerequisites:**
- ngrok operator installed in your cluster
- ngrok API credentials configured

**Get the ngrok URL:**

```bash
# Get the ngrok ingress URL
kubectl get ingress temporal-ui-ngrok -n alpaca-trader

# Or check ngrok edges
kubectl get ngrokedge -n alpaca-trader
```

The Temporal UI will be accessible at the ngrok URL provided by the operator. Access is protected by OAuth (configured as Google OAuth by default in the ingress annotations).

**To change OAuth provider or configure access restrictions:**
Edit the `ngrok.com/oauth` and `ngrok.com/oauth-options` annotations in the `temporal-ui-ngrok` Ingress resource in `config/k8s.yaml`.

### 7. Update Trade Schedule

To update the trade schedule, edit `trade_schedule.yaml` in your repository, then rebuild and redeploy:

```bash
# Rebuild the image with updated schedule
docker build -t YOUR_REGISTRY/alpaca-schedule-trader:latest .
docker push YOUR_REGISTRY/alpaca-schedule-trader:latest

# Restart the deployment to pull the new image
kubectl rollout restart deployment/schedule -n alpaca-trader
```

## Architecture

The deployment includes:

1. **PostgreSQL**: Database for Temporal persistence
2. **Temporal Server**: Workflow orchestration engine
3. **Temporal UI**: Web interface for monitoring workflows
4. **Schedule Pod**: Creates and manages Temporal schedules from YAML config
5. **Worker Pods**: Execute trading workflows (2 replicas for redundancy)

## Configuration

### Environment Variables

- `TEMPORAL_HOST`: Temporal server hostname (default: `temporal-server`)
- `TEMPORAL_PORT`: Temporal server port (default: `7233`)
- `TEMPORAL_NAMESPACE`: Temporal namespace (default: `default`)
- `ALPACA_API_KEY_ID`: Alpaca API key (from secret)
- `ALPACA_API_SECRET_KEY`: Alpaca API secret (from secret)
- `ALPACA_PAPER`: Use paper trading (from secret, optional)

### Scaling

To scale workers:

```bash
kubectl scale deployment worker --replicas=5 -n alpaca-trader
```

### Storage

PostgreSQL uses a PersistentVolumeClaim (20Gi). Adjust the size in `k8s.yaml` if needed.

## Troubleshooting

### Check pod logs:

```bash
# Schedule logs
kubectl logs -f deployment/schedule -n alpaca-trader

# Worker logs
kubectl logs -f deployment/worker -n alpaca-trader

# Temporal server logs
kubectl logs -f deployment/temporal-server -n alpaca-trader
```

### Check pod status:

```bash
kubectl describe pod <pod-name> -n alpaca-trader
```

### Restart deployments:

```bash
kubectl rollout restart deployment/schedule -n alpaca-trader
kubectl rollout restart deployment/worker -n alpaca-trader
```
