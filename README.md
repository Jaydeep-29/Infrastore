# InfraStore Kubernetes Deployment

A production-ready Kubernetes deployment for the InfraStore REST API with comprehensive security, scalability, and reliability features.

## Overview

This project deploys InfraStore (a Django-based file storage service) on Kubernetes using Kustomize for environment-specific configuration. The deployment includes all modern Kubernetes best practices for security, high availability, and observability.

## ** Key Highlight: Health Checks
    ## very Important: deployment uses different health check strategies for different environments:
            Local Development: exec probes (process check) - simple and forgiving
            Production: httpGet probes (application health) - strict and accurate

## Project Structure

```
infrastore-deployment/
    base/                           # Shared base configuration
       -namespace.yaml             # infrastore namespace
       -deployment.yaml            # Main application deployment
       -service.yaml               # Service configuration
       -rbac.yaml                  # RBAC (ServiceAccount, Role, RoleBinding)
       -network-policy.yaml        # Network security policies
       -secret.yaml                # Sealed secrets for credentials
       -pvc-media.yaml             # Media persistent volume claim
       -pvc-db.yaml                # Database persistent volume claim
       -kustomization.yaml         # Base Kustomize configuration

    overlays/
       -local/                     # Local development configuration
        -kustomization.yaml     # Local-specific settings (1 replica, dev probes)
       -prod/                      # Production configuration
        -kustomization.yaml     # Production-specific settings (3+ replicas)
        -hpa.yaml               # Horizontal Pod Autoscaler (3-20 replicas)
        -pdb.yaml               # Pod Disruption Budget (minAvailable: 2)
        -resource-quota.yaml    # Namespace resource limits

    README.md                       # This file
```

## Security Features Implemented

### 1. **Sealed Secrets**
- Encrypted credentials stored safely in Git
- Uses Bitnami sealed-secrets controller
- Credentials automatically decrypted at deployment time
- Supports safe credential rotation

### 2. **RBAC (Role-Based Access Control)**
- ServiceAccount with minimal permissions
- Pod can only access its own secret (`infrastore-admin`)
- Restricted to reading logs and pod information
- Follows principle of least privilege

### 3. **Network Policies**
- Default deny all ingress/egress traffic
- Selective allow rules for:
  - Ingress from ingress-nginx controller on port 8000
  - Inter-pod communication within namespace
  - Egress for DNS (port 53 UDP) and HTTPS (port 443 TCP)

### 4. **Pod Security**
- Non-root user (UID 10001)
- Read-only root filesystem where possible
- No elevated privileges
- Dropped all Linux capabilities
- RuntimeDefault seccomp profile

## Scalability Features

### 1. **Pod Affinity & Topology Spread**
- Pod anti-affinity: spreads replicas across nodes
- Topology spread: distributes across availability zones
- Ensures high availability in multi-node clusters

### 2. **Horizontal Pod Autoscaling (Production)**
- Scales from 3 to 20 replicas
- Based on CPU (70% threshold) and memory metrics
- Respects PDB for safe scaling

### 3. **Resource Management**
- CPU requests: 200m, limits: 1000m
- Memory requests: 256Mi, limits: 1Gi
- Namespace-level quotas in production

### 4. **Persistent Storage**
- Media uploads: local-storage PVC (5Gi)
- Database: local-storage PVC (5Gi)
- Survives pod restarts and updates

## Reliability Features

### 1. **Health Checks**
- Readiness probe: checks process is running (20s initial delay, 15s period)
- Liveness probe: ensures pod stays healthy (60s initial delay, 30s period)
- Both support graceful recovery

### 2. **Graceful Shutdown**
- 60-second termination grace period
- Allows in-flight requests to complete
- Clean application shutdown on pod termination

### 3. **Deployment Strategy**
- Rolling updates with zero downtime
- maxSurge: 1 (can temporarily run 2 pods)
- maxUnavailable: 0 (no pods down during update)

### 4. **Pod Disruption Budget (Production)**
- minAvailable: 2 pods
- Prevents node maintenance from disrupting service
- Protects against voluntary disruptions

## Prerequisites

### Local Testing
- **Docker Desktop** (with Kubernetes enabled) or any local Kubernetes cluster
- **kubectl** (comes with Docker Desktop)
- **kubeseal** (`brew install kubeseal`)
- **kustomize** (`brew install kustomize`)

### Production
- **Multi-node Kubernetes cluster** (1.20+)
- **Persistent storage provisioner** (e.g., local-storage, EBS, GCP Persistent Disks)
- **Ingress controller** (e.g., nginx-ingress)
- **Sealed-secrets controller** (for secret management)

## Quick Start

### 1. Enable Kubernetes (Local Only)

```bash
# Open Docker Desktop
# Settings → Kubernetes → Enable Kubernetes → Apply & Restart
# Wait 2-3 minutes for cluster to start
```

### 2. Create Namespace

```bash
kubectl create namespace infrastore
```

### 3. Setup Sealed Secrets

```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml -n kube-system

# Wait for controller to be ready
kubectl rollout status deployment/sealed-secrets-controller -n kube-system --timeout=5m
```

### 4. Create and Seal Credentials

```bash
# Create unsealed secret
kubectl create secret generic infrastore-admin \
  --from-literal=DJANGO_SUPERUSER_USERNAME=admin \
  --from-literal=DJANGO_SUPERUSER_PASSWORD=YourSecurePassword123! \
  -n infrastore \
  --dry-run=client -o yaml | kubeseal > base/secret.yaml

# Verify sealed secret
cat base/secret.yaml
```

### 5. Deploy to Local Cluster

```bash
# Navigate to project directory
cd infrastore-deployment

# Deploy local configuration (1 replica, dev-friendly settings)
kubectl kustomize overlays/local | kubectl apply -f -

# Monitor rollout
kubectl rollout status deployment/infrastore -n infrastore --timeout=5m

# Check pod status
kubectl get pods -n infrastore -o wide
```

### 6. Test the Application

```bash
# Port forward to access app locally
kubectl port-forward -n infrastore svc/infrastore 8080:8000 &

# Wait a moment for port-forward to establish
sleep 2

# Test endpoints
curl -v http://localhost:8080/
curl -v http://localhost:8080/api/files/

# Get authentication token
curl -X POST http://localhost:8080/api/token/ \
  -d "username=admin&password=YourSecurePassword123!"
```

## Verification Checklist

After deployment, verify all components are healthy:

```bash
# Check pods are Ready
kubectl get pods -n infrastore -o wide
# Expected: STATUS=Running, READY=1/1

# Check deployment
kubectl get deployment -n infrastore
# Expected: READY=1/1, AVAILABLE=1

# Check services
kubectl get svc -n infrastore
# Expected: service/infrastore, port 8000/TCP

# Check security (RBAC)
kubectl get serviceaccount,role,rolebinding -n infrastore
# Expected: infrastore ServiceAccount, Role, RoleBinding

# Check network policies
kubectl get networkpolicies -n infrastore
# Expected: 3 policies (default-deny, allow-ingress, allow-egress)

# Check secrets
kubectl get secrets -n infrastore
# Expected: infrastore-admin (Opaque, 2 data items)

# Check persistent volumes
kubectl get pvc -n infrastore
# Expected: infrastore-media, infrastore-db (Bound)
```

## Environment-Specific Configurations

### Local Development

```bash
kubectl kustomize overlays/local | kubectl apply -f -
```

**Configuration:**
- 1 replica (single pod)
- Health checks use process check (very forgiving)
- maxUnavailable: 1 (faster updates in dev)
- ScheduleAnyway for topology spread (works on single node)

**Best for:** Developer testing, debugging, rapid iteration

### Production

```bash
kubectl kustomize overlays/prod | kubectl apply -f -
```

**Configuration:**
- 3 minimum replicas, up to 20 with HPA
- Health checks use HTTP endpoints (stricter)
- maxUnavailable: 0 (zero-downtime)
- DoNotSchedule for topology spread (strict HA)
- Pod Disruption Budget protects availability
- Resource quotas prevent runaway usage
- Autoscaling based on CPU and memory metrics


### View Logs

```bash
# Real-time logs
kubectl logs -f deployment/infrastore -n infrastore

```

### Check Pod Events

```bash
# Recent events in namespace
kubectl get events -n infrastore --sort-by='.lastTimestamp'

```

## Troubleshooting

### Pod not becoming Ready

```bash
# Check pod status and events
kubectl describe pod -n infrastore <pod-name>

# Check container logs
kubectl logs -n infrastore <pod-name>

# If readiness probe fails, check health endpoint
kubectl exec -it -n infrastore <pod-name> -- sh
curl localhost:8000/
```

### Network connectivity issues

```bash
# Verify network policies
kubectl get networkpolicies -n infrastore

# Test connectivity from pod
kubectl exec -it -n infrastore <pod-name> -- sh
curl https://www.google.com  # Should work (egress to 443)
curl 8.8.8.8:53 -v          # Should work (DNS on 53)
```

### Persistent volume issues

```bash
# Check PVC status
kubectl get pvc -n infrastore

# Describe PVC for details
kubectl describe pvc infrastore-media -n infrastore

# Check physical volume
kubectl get pv
```

### RBAC permission denied

```bash
# Verify ServiceAccount is bound to Role
kubectl describe rolebinding infrastore -n infrastore

# Check role permissions
kubectl describe role infrastore -n infrastore

```

## Cleanup

### Stop local cluster

```bash
# Delete all resources in namespace (keeps namespace)
kubectl delete all --all -n infrastore

# Delete entire namespace (removes everything)
kubectl delete namespace infrastore

# Verify cleanup
kubectl get namespaces
```

### Clean sealed-secrets (if needed)

```bash
# Delete sealed-secrets controller
kubectl delete -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml -n kube-system
```

## Support

For issues or questions:

1. Check pod logs: `kubectl logs deployment/infrastore -n infrastore`
2. Check events: `kubectl get events -n infrastore`
3. Describe resources: `kubectl describe pod/deployment/svc -n infrastore`
4. Review Kubernetes documentation: https://kubernetes.io/docs/
