# Kubernetes Deployment Guide

## Overview

This directory contains Kubernetes manifests for deploying the ML inference service to a Kubernetes cluster.

## Files

- **`configmap.yaml`**: Application configuration (externalized from Docker image)
- **`deployment.yaml`**: Main deployment with 3 replicas, resource limits, and health probes
- **`service.yaml`**: ClusterIP service for internal load balancing
- **`hpa.yaml`**: Horizontal Pod Autoscaler for automatic scaling

## Quick Start

### Prerequisites

- Kubernetes cluster (v1.20+)
- `kubectl` configured to access your cluster
- Docker image pushed to a container registry

### Deploy

```bash
# 1. Update the image in deployment.yaml
# Edit line 43: image: your-registry/mlops-inference:v1.0

# 2. Apply all manifests
kubectl apply -f k8s/

# 3. Verify deployment
kubectl get pods -l app=mlops-inference
kubectl get svc mlops-inference-service
kubectl get hpa mlops-inference-hpa

# 4. Check pod logs
kubectl logs -f deployment/mlops-inference

# 5. Test the service (port-forward for testing)
kubectl port-forward service/mlops-inference-service 8080:80

# 6. Make a prediction
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This movie was fantastic!"}'
```

## Design Choices

### Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5 CPU cores
    memory: "1Gi"    # 1GB RAM
  limits:
    cpu: "2000m"     # 2 CPU cores
    memory: "2Gi"    # 2GB RAM
```

**Rationale:**
- **Requests**: Based on baseline resource usage during model loading and inference
  - Model size: ~500MB
  - Overhead: ~300MB (Python runtime, dependencies)
  - CPU: 0.5 cores handles ~10-20 requests/second
- **Limits**: Allow bursting for traffic spikes
  - 2x CPU for handling concurrent requests
  - 2GB memory prevents OOM during batch predictions

**How to tune:**
```bash
# Monitor actual usage
kubectl top pods

# Adjust based on metrics
# - If CPU consistently >80% → Increase request
# - If memory near limit → Increase limit
# - If pods throttled → Increase CPU limit
```

### Health Probes

**Liveness Probe** (Is the container alive?)
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30  # Wait for model loading
  periodSeconds: 10
  failureThreshold: 3      # Restart after 3 failures
```

**Readiness Probe** (Can the container serve traffic?)
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10  # Check sooner
  periodSeconds: 5         # Check more frequently
  failureThreshold: 2      # Remove from service faster
```

**Rationale:**
- **Liveness**: Detects deadlocks/hangs and restarts the container
- **Readiness**: Prevents traffic to pods during startup or temporary issues
- **Different timing**: Readiness checks faster to quickly remove unhealthy pods from service

### Scaling Strategy

**Horizontal Pod Autoscaler (HPA):**
```yaml
minReplicas: 2   # High availability
maxReplicas: 10  # Cost control
metrics:
  - cpu: 70%     # Scale when average CPU > 70%
  - memory: 80%  # Scale when average memory > 80%
```

**Scaling behavior:**
- **Scale up**: Fast (60s stabilization, +50% pods)
- **Scale down**: Slow (5min stabilization, -1 pod at a time)

**Rationale:**
- **Min 2 replicas**: Ensures availability during pod failures or updates
- **Max 10 replicas**: Prevents runaway scaling and cost overruns
- **70% CPU target**: Leaves headroom for traffic spikes
- **Asymmetric scaling**: Quick scale-up for responsiveness, slow scale-down for stability

**Alternative scaling approaches:**
1. **Custom metrics** (requests per second):
   ```yaml
   metrics:
   - type: Pods
     pods:
       metric:
         name: http_requests_per_second
       target:
         averageValue: "1000"
   ```

2. **Vertical Pod Autoscaler (VPA)**: Automatically adjust resource requests
3. **Cluster Autoscaler**: Add nodes when pods can't be scheduled

## GPU Support

### Enabling GPU for Inference

**1. Update Deployment:**
```yaml
# deployment.yaml
spec:
  containers:
  - name: inference-api
    image: mlops-inference:v1.0-gpu  # GPU-enabled image
    resources:
      requests:
        nvidia.com/gpu: 1  # Request 1 GPU
      limits:
        nvidia.com/gpu: 1
    
  # Ensure scheduling on GPU nodes
  nodeSelector:
    accelerator: nvidia-tesla-t4
  
  # Tolerate GPU node taints
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

**2. Update Docker Image:**
```dockerfile
# Use NVIDIA CUDA base image
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# Install Python and dependencies
RUN apt-get update && apt-get install -y python3.10 python3-pip

# Install GPU-enabled libraries
RUN pip install torch --index-url https://download.pytorch.org/whl/cu118
```

**3. Cluster Requirements:**
- NVIDIA GPU Operator installed
- GPU nodes with appropriate labels and taints
- Device plugin for GPU resource management

### GPU Considerations

**When to use GPU:**
- ✅ Large models (>1B parameters)
- ✅ High throughput requirements (>100 req/s)
- ✅ Batch inference workloads
- ✅ Real-time inference with tight latency SLAs

**When to use CPU:**
- ✅ Small models (like this text classifier)
- ✅ Low-medium traffic (<50 req/s)
- ✅ Cost-sensitive deployments
- ✅ Simpler operations (no GPU driver management)


**Optimization strategies:**
1. **Batch requests**: Accumulate requests and process in batches
2. **Model optimization**: Use TensorRT, ONNX Runtime
3. **Mixed precision**: FP16 inference for 2x speedup
4. **Multi-model serving**: Share GPU across multiple models

## Monitoring and Observability

### Recommended Metrics

**Application metrics** (expose via `/metrics` endpoint):
```python
# Prometheus metrics
predictions_total = Counter('predictions_total', 'Total predictions')
prediction_latency = Histogram('prediction_latency_seconds', 'Prediction latency')
model_load_time = Gauge('model_load_time_seconds', 'Model load time')
active_requests = Gauge('active_requests', 'Active requests')
```

**Kubernetes metrics** (via kube-state-metrics):
- Pod restart count
- CPU/memory usage
- Request rate
- Error rate

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
- name: mlops-inference
  rules:
  - alert: HighPodRestartRate
    expr: rate(kube_pod_container_status_restarts_total[1h]) > 5
    annotations:
      summary: "Pod restart rate > 5/hour"
  
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
    annotations:
      summary: "Error rate > 1%"
  
  - alert: HighLatency
    expr: histogram_quantile(0.95, prediction_latency_seconds) > 0.5
    annotations:
      summary: "P95 latency > 500ms"
```

### Logging Strategy

**Structured logging:**
```python
import json
import logging

logger.info(json.dumps({
    "event": "prediction",
    "request_id": request_id,
    "latency_ms": latency,
    "model_version": "v1.0",
    "confidence": confidence,
    "timestamp": datetime.utcnow().isoformat()
}))
```

**Log aggregation:**
- Use Fluentd/Fluent Bit to collect logs
- Send to Elasticsearch, CloudWatch, or Stackdriver
- Create dashboards for visualization

## Updating the Service

### Update Configuration

```bash
# 1. Edit configmap.yaml
vim k8s/configmap.yaml

# 2. Apply changes
kubectl apply -f k8s/configmap.yaml

# 3. Restart pods to pick up new config
kubectl rollout restart deployment/mlops-inference

# 4. Monitor rollout
kubectl rollout status deployment/mlops-inference
```

### Update Application Code

```bash
# 1. Build new image
docker build -f docker/Dockerfile -t mlops-inference:v1.1 .

# 2. Push to registry
docker push your-registry/mlops-inference:v1.1

# 3. Update deployment
kubectl set image deployment/mlops-inference \
  inference-api=your-registry/mlops-inference:v1.1

# 4. Monitor rollout (zero downtime!)
kubectl rollout status deployment/mlops-inference

# 5. Rollback if needed
kubectl rollout undo deployment/mlops-inference

## Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl get pods -l app=mlops-inference

# Describe pod for events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Common issues:
# - Image pull errors → Check image name and registry access
# - OOMKilled → Increase memory limit
# - CrashLoopBackOff → Check application logs
```

### Service not accessible

```bash
# Check service endpoints
kubectl get endpoints mlops-inference-service

# If no endpoints:
# - Check pod labels match service selector
# - Check readiness probe is passing

# Test from within cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://mlops-inference-service/health
```

### HPA not scaling

```bash
# Check HPA status
kubectl get hpa mlops-inference-hpa

# Check metrics server
kubectl top pods


