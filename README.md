# ChatFlow Kubernetes Manifests

Kubernetes deployment manifests for the ChatFlow AI chat application.

## Directory Structure

```
chatflow-k8s-manifests/
├── namespace.yaml          # chatflow namespace
├── mongodb/
│   ├── pvc.yaml           # 10GB persistent storage
│   ├── deployment.yaml    # MongoDB database (1 replica)
│   └── service.yaml       # ClusterIP service
├── redis/
│   ├── deployment.yaml    # Redis cache (1 replica)
│   └── service.yaml       # ClusterIP service
├── ollama/
│   ├── pvc.yaml           # 5GB for AI models
│   ├── deployment.yaml    # Ollama AI engine (1 replica)
│   └── service.yaml       # ClusterIP service
├── backend/
│   ├── configmap.yaml     # Environment variables
│   ├── deployment.yaml    # Node.js API (3 replicas)
│   └── service.yaml       # LoadBalancer service
└── frontend/
    ├── deployment.yaml    # React UI (2 replicas)
    └── service.yaml       # LoadBalancer service
```

## Prerequisites

- Kubernetes cluster (EKS, Minikube, or other)
- kubectl configured to connect to your cluster
- Docker images pushed to registry:
  - `arsalananwer0/chatflow-backend:latest`
  - `arsalananwer0/chatflow-frontend:latest`

## Deployment Order

Apply manifests in this order:

```bash
# 1. Create namespace
kubectl apply -f namespace.yaml

# 2. Create storage (PVCs)
kubectl apply -f mongodb/pvc.yaml
kubectl apply -f ollama/pvc.yaml

# 3. Deploy databases and cache
kubectl apply -f mongodb/deployment.yaml
kubectl apply -f mongodb/service.yaml
kubectl apply -f redis/deployment.yaml
kubectl apply -f redis/service.yaml

# 4. Deploy Ollama AI engine
kubectl apply -f ollama/deployment.yaml
kubectl apply -f ollama/service.yaml

# 5. Deploy backend API
kubectl apply -f backend/configmap.yaml
kubectl apply -f backend/deployment.yaml
kubectl apply -f backend/service.yaml

# 6. Deploy frontend
kubectl apply -f frontend/deployment.yaml
kubectl apply -f frontend/service.yaml
```

Or apply all at once:

```bash
kubectl apply -f ./ --recursive
```

## Verify Deployment

```bash
# Check all resources in chatflow namespace
kubectl get all -n chatflow

# Check pods status
kubectl get pods -n chatflow

# Check services and external IPs
kubectl get svc -n chatflow

# Check persistent volumes
kubectl get pvc -n chatflow
```

## Access the Application

After deployment, get the LoadBalancer URLs:

```bash
# Get frontend URL
kubectl get svc frontend -n chatflow
# Access at: http://<EXTERNAL-IP>

# Get backend URL
kubectl get svc backend -n chatflow
# API available at: http://<EXTERNAL-IP>:5001/api
```

## Pull AI Model (One-time Setup)

After Ollama pod is running, pull the phi model:

```bash
# Find Ollama pod name
kubectl get pods -n chatflow | grep ollama

# Exec into the pod
kubectl exec -it <ollama-pod-name> -n chatflow -- ollama pull phi
```

## Service Communication

Internal services use DNS names:
- Backend connects to: `mongodb://mongodb:27017/chatflow`
- Backend connects to: `redis://redis:6379`
- Backend connects to: `http://ollama:11434`

External access via LoadBalancers:
- Users access Frontend LoadBalancer
- Frontend (browser) calls Backend LoadBalancer

## Resource Allocation

| Service  | Replicas | CPU Request | Memory Request | Storage |
|----------|----------|-------------|----------------|---------|
| MongoDB  | 1        | 250m        | 512Mi          | 10Gi    |
| Redis    | 1        | 100m        | 128Mi          | -       |
| Ollama   | 1        | 500m        | 2Gi            | 5Gi     |
| Backend  | 3        | 250m        | 256Mi          | -       |
| Frontend | 2        | 100m        | 128Mi          | -       |

**Total:** ~1.5 CPU cores, ~3.5GB RAM minimum

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n chatflow
kubectl logs <pod-name> -n chatflow
```

### PVC not binding
```bash
kubectl get pvc -n chatflow
kubectl describe pvc <pvc-name> -n chatflow
# Check if StorageClass exists
kubectl get storageclass
```

### Service has no external IP
```bash
# On AWS EKS, it takes 2-3 minutes to provision ELB
kubectl get svc -n chatflow -w  # Watch for changes
```

### Backend can't connect to MongoDB
```bash
# Verify MongoDB service exists
kubectl get svc mongodb -n chatflow

# Check backend logs
kubectl logs deployment/backend -n chatflow
```

## Scaling

Scale deployments up or down:

```bash
# Scale backend to 5 replicas
kubectl scale deployment backend -n chatflow --replicas=5

# Scale frontend to 3 replicas
kubectl scale deployment frontend -n chatflow --replicas=3
```

## Updates

Update image versions:

```bash
# Update backend image
kubectl set image deployment/backend backend=arsalananwer0/chatflow-backend:v2.0.0 -n chatflow

# Update frontend image
kubectl set image deployment/frontend frontend=arsalananwer0/chatflow-frontend:v2.0.0 -n chatflow
```

## Cleanup

Remove all resources:

```bash
# Delete all resources in namespace
kubectl delete namespace chatflow

# Or delete individually
kubectl delete -f ./ --recursive
```

## Next Steps

- **Phase 4**: Provision infrastructure with Terraform (chatflow-infrastructure repo)
- **Phase 5**: CI/CD pipeline with Jenkins
- **Phase 6**: GitOps with ArgoCD

## Notes

- MongoDB and Ollama use PersistentVolumeClaims for data persistence
- Backend has 3 replicas for high availability and load distribution
- Health probes ensure only healthy pods receive traffic
- LoadBalancer services will provision AWS ELBs (costs ~$18/month each)
- Consider using Ingress to reduce LoadBalancer costs

## Repository Structure

This is part of the ChatFlow project with 3 repositories:
- **chatflow-app**: Application source code
- **chatflow-k8s-manifests** (this repo): Kubernetes manifests
- **chatflow-infrastructure**: Terraform configurations