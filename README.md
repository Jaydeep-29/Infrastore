# InfraStore Kubernetes Deployment
## Overview

The deployment includes modern Kubernetes best practices for security, reliability, and observability, within the constraints of the provided application.

## ** Key Highlight: 
    ## very Important: 
    Health Checks
    - The deployment uses exec-based health probes by default to ensure compatibility
      with the provided container image.
    - HTTP-based probes are documented and recommended for production environments,
      but are not enabled by default
            Local Development: exec probes (process check) - simple and forgiving
            Production: httpGet probes (application health) - strict and accurate

      - The application runs as a single replica by design because it uses
      file-based persistence (SQLite) backed by ReadWriteOnce persistent volumes.

      - Horizontal Pod Autoscaling is included as a reference configuration,
      but scaling beyond one replica is intentionally avoided to prevent
      database corruption and volume mount conflicts.

      - In a real production environment, the database would be migrated to
      PostgreSQL and media storage to object storage (e.g., S3 or EFS),
      after which multi-replica scaling and HPA would be enabled.

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
      -kustomization.yaml         # Base Kustomize configuration
      -kustomizeconfig.yaml
    media/
       -pvc-media.yaml             # Media persistent volume claim
       -pvc-db.yaml                # Database persistent volume claim
      
    overlays/
       -local/                     # Local development configuration
        -kustomization.yaml     # Local-specific settings (1 replica, dev probes)
      -prod/                      # Production configuration
        -kustomization.yaml     # Production-specific settings
        -hpa.yaml               # Horizontal Pod Autoscaler
        -pdb.yaml               # Pod Disruption Budget (minAvailable: 1)
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
- Root filesystem write access is enabled to ensure compatibility with the provided application image.
- No elevated privileges
- Dropped all Linux capabilities
- RuntimeDefault seccomp profile

## Scalability Features

### 1. **Pod Affinity & Topology Spread**
- Pod anti-affinity: spreads replicas across nodes
- Topology spread: distributes across availability zones
- Ensures high availability in multi-node clusters

### 3. **Resource Management**
- CPU requests: 200m, limits: 1000m
- Memory requests: 256Mi, limits: 1Gi
- Namespace-level quotas in production

### 4. **Persistent Storage**
- Media uploads: local-storage PVC (2Gi)
- Database: local-storage PVC (1Gi)
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
- The application is deployed as a single replica and may experience
brief unavailability during updates.

- This trade-off is intentional and ensures safe updates when using
file-based persistence with ReadWriteOnce volumes.

- For a stateless, database-backed deployment (e.g., PostgreSQL),
rolling updates with zero downtime would be enabled.


### 4. **Pod Disruption Budget (Production)**
- minAvailable: 1 pods
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
Open Docker Desktop and enable kubernetes after wait 2-3 mins 
```

### 2. Setup Sealed Secrets

```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml -n kube-system

# Wait for controller to be ready
kubectl rollout status deployment/sealed-secrets-controller -n kube-system --timeout=5m

# verify pod has been created
kubectl get pods -n kube-system | grep sealed-secrets
```

### 3. Create and Seal Credentials

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

### 4. Deploy to Local Cluster

```bash
# Navigate to project directory
cd infrastore-deployment

# Deploy local configuration (1 replica, dev-friendly settings)
kubectl kustomize overlays/local | kubectl apply -f -

# Monitor rollout
kubectl rollout status deployment -n infrastore -l app=infrastore --timeout=5m


# Check pod status
kubectl get pods -n infrastore -l app=infrastore -o wide
```

### 5. Test the Application

```bash
# Port forward to access app locally
kubectl port-forward -n infrastore svc/infrastore 8080:8000

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
kubectl get pods -n infrastore  -l app=infrastore -o wide
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
- Health checks use HTTP endpoints (stricter)
- maxUnavailable: 0 (zero-downtime)
- DoNotSchedule for topology spread (strict HA)
- Pod Disruption Budget protects availability
- Resource quotas prevent runaway usage

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
kubectl describe pod -n infrastore -l app=infrastore

# Check container logs
kubectl logs -n infrastore -l app=infrastore

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
curl https://www.google.com  # Test HTTPS egress
nslookup google.com        # Test DNS resolution
```

### Persistent volume issues

```bash
# Check PVC status
kubectl get pvc -n infrastore -l app=infrastore

# Describe PVC for details
kubectl describe pvc -n infrastore

# Check physical volume
kubectl get pv
```

### RBAC permission denied

```bash
# Verify ServiceAccount is bound to Role
kubectl get role,rolebinding -n infrastore
# Check role permissions
kubectl describe role -n infrastore 
kubectl describe rolebinding -n infrastore

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

1. Check pod logs: `kubectl logs -n infrastore -l app=infrastore`
2. Check events: `kubectl get events -n infrastore`
3. Describe resources: `kubectl describe pod/deployment/svc -n infrastore`
4. Review Kubernetes documentation: https://kubernetes.io/docs/
