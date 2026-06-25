# Book-My-Show - Kubernetes Deployment Guide

This directory contains Kubernetes manifests for deploying the Book-My-Show application to EKS.

## Prerequisites

- EKS Cluster (v1.24+)
- kubectl configured for your EKS cluster
- Docker image pushed to Docker Hub (varunspark/bookmyshow:latest)
- Prometheus & Grafana installed (optional, for monitoring)

## File Structure

```
k8s/
├── namespace.yml              # Create bookmyshow namespace
├── configmap-secret.yml       # Configuration and secrets
├── deployment.yml             # Deployment, Service, HPA, PDB
├── network-policy.yml         # Network security policies
├── service-monitor.yml        # Prometheus monitoring
└── README.md                  # This file
```

## Deployment Steps

### 1. Create Namespace
```bash
kubectl apply -f k8s/namespace.yml
```

### 2. Create ConfigMap and Secrets
```bash
kubectl apply -f k8s/configmap-secret.yml
```

**⚠️ IMPORTANT:** Update the secrets in `configmap-secret.yml` with actual values before applying.

### 3. Apply Network Policies (Optional but Recommended)
```bash
kubectl apply -f k8s/network-policy.yml
```

### 4. Deploy Application
```bash
kubectl apply -f k8s/deployment.yml
```

### 5. Verify Deployment
```bash
# Check namespace
kubectl get namespaces

# Check pods
kubectl get pods -n bookmyshow

# Check services
kubectl get svc -n bookmyshow

# Check deployment status
kubectl get deployment -n bookmyshow

# View pod logs
kubectl logs -n bookmyshow -l app=bookmyshow --tail=50 -f
```

### 6. Access Application (Optional)
```bash
# Port forward (local testing)
kubectl port-forward svc/bookmyshow-service 3000:80 -n bookmyshow

# Then access: http://localhost:3000
```

### 7. Monitor Application
```bash
# Check HPA status
kubectl get hpa -n bookmyshow

# Watch replica scaling
kubectl get pods -n bookmyshow -w
```

## Configuration

### Environment Variables
Modify in `configmap-secret.yml`:
- `NODE_ENV`: Set to "production" for EKS
- `PORT`: Internal port (default: 3000)
- `LOG_LEVEL`: Debug level (info, warn, error)

### Resource Limits
Adjust in `deployment.yml`:
```yaml
resources:
  requests:
    cpu: 100m        # Minimum CPU required
    memory: 256Mi    # Minimum memory required
  limits:
    cpu: 500m        # Maximum CPU allowed
    memory: 512Mi    # Maximum memory allowed
```

### Scaling
- **Replicas:** Default 3 pods (min: 2, max: 10)
- **Triggers:**
  - CPU: Scale up at 70% utilization
  - Memory: Scale up at 80% utilization
- **Scale Down:** 300 seconds stabilization, max 50% reduction per 60 seconds
- **Scale Up:** Immediate, max 100% increase per 30 seconds

## Health Checks

### Liveness Probe
- Checks if pod is alive
- Restarts pod if it fails 3 times in a row
- Initial delay: 30 seconds
- Check interval: 10 seconds

### Readiness Probe
- Checks if pod is ready to serve traffic
- Removes from load balancer if it fails 3 times
- Initial delay: 10 seconds
- Check interval: 5 seconds

## Monitoring

### Prometheus Integration
ServiceMonitor is configured to scrape metrics from `/metrics` endpoint every 30 seconds.

```bash
# View Prometheus targets
kubectl port-forward -n prometheus svc/prometheus 9090:9090

# Access: http://localhost:9090/targets
```

### Grafana Dashboards
Import these dashboards:
- Node Exporter Full: https://grafana.com/grafana/dashboards/1860
- Kubernetes Cluster: https://grafana.com/grafana/dashboards/7249

## Security Best Practices

✅ **Implemented:**
- Resource quotas (CPU/Memory limits)
- Health checks (Liveness & Readiness probes)
- Network policies restricting traffic
- Pod disruption budget for high availability
- ReadOnlyRootFilesystem (can be enabled)
- Non-root user (recommend in Dockerfile)

✅ **Recommended:**
- Use AWS Secrets Manager for secrets
- Enable Pod Security Policy
- Use RBAC for access control
- Enable API auditing
- Implement OPA/Gatekeeper for policy enforcement

## Updating Application

### Rolling Update
```bash
# Update image tag
kubectl set image deployment/bookmyshow \
  bookmyshow=varunspark/bookmyshow:v2.0 \
  -n bookmyshow

# Monitor rollout
kubectl rollout status deployment/bookmyshow -n bookmyshow

# Rollback if needed
kubectl rollout undo deployment/bookmyshow -n bookmyshow
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod -n bookmyshow <pod-name>
kubectl logs -n bookmyshow <pod-name>
```

### Service not reachable
```bash
# Check service
kubectl get svc -n bookmyshow

# Check endpoints
kubectl get endpoints -n bookmyshow

# Test DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup bookmyshow-service.bookmyshow.svc.cluster.local
```

### HPA not scaling
```bash
# Check metrics
kubectl get hpa -n bookmyshow -w

# Check metrics server
kubectl get deployment metrics-server -n kube-system
```

## Cleanup

```bash
# Delete namespace (deletes all resources inside)
kubectl delete namespace bookmyshow

# Or delete individual resources
kubectl delete -f k8s/
```

## CI/CD Integration

These manifests are automatically applied by the Jenkins pipeline:
```groovy
stage('Deploy to EKS Cluster') {
    steps {
        sh '''
        kubectl apply -f k8s/namespace.yml
        kubectl apply -f k8s/configmap-secret.yml
        kubectl apply -f k8s/deployment.yml
        '''
    }
}
```

## Support

For issues or questions:
- Check Kubernetes documentation: https://kubernetes.io/docs/
- Check EKS documentation: https://docs.aws.amazon.com/eks/
- Review application logs: `kubectl logs -n bookmyshow -l app=bookmyshow -f`
