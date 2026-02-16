# Production ML Inference System

> **MLOps Assessment Project**: A production-ready text classification inference service with Docker containerization and Kubernetes orchestration.


## 📋 Assessment Overview

This project demonstrates production-ready MLOps practices for deploying a machine learning inference service:

- ✅ **Task 1**: Minimal ML inference service (REST API, stateless, containerized)
- ✅ **Task 2**: Kubernetes deployment with resource management and health probes
- ✅ **Documentation**: Design decisions and operational considerations

**Key Documents:**
- 📖 [Design & Decision Document](DESIGN_DECISIONS.md) - Architecture and trade-offs
- 🚀 [Kubernetes Deployment Guide](k8s/README.md) - K8s manifests and scaling strategies

- 📡 [API Examples](docs/API_EXAMPLES.md) - Request/response examples

---

## 🎯 Features

- **FastAPI REST API** for text classification
- **Stateless design** for horizontal scalability
- **Docker containerization** with optimized image size
- **Kubernetes manifests** with production-ready configuration
- **Horizontal autoscaling** based on CPU/memory metrics
- **Zero-downtime deployments** via rolling updates
- **Health probes** for liveness and readiness checks
- **Batch and single text prediction**
- **Streamlit web app** for interactive testing

---

## 🏗️ Architecture

<img width="601" height="381" alt="mlops1 drawio" src="https://github.com/user-attachments/assets/5d54e134-02df-4d40-bab9-b3f1e7887062" />

**Design Principles:**
- Stateless service (model loaded at startup)
- Horizontal scaling via Kubernetes HPA
- Resource-efficient (CPU-based inference)
- Production-ready monitoring and logging

---

## 📁 Project Structure

```
mlops-inference/
├── src/
│   ├── api/                   # FastAPI app and schemas
│   │   ├── app.py            # Main API with health checks
│   │   └── schemas.py        # Pydantic models
│   ├── inference/            # Prediction logic
│   │   └── predictor.py      # Text classification predictor
│   ├── model/                # Model loader
│   │   └── loader.py         # Load model packages
│   └── preprocessing/        # Text preprocessing
│       └── text_processor.py # Text cleaning and vectorization
├── k8s/                      # Kubernetes manifests
│   ├── deployment.yaml       # Deployment with resource limits
│   ├── service.yaml          # ClusterIP service
│   ├── configmap.yaml        # Configuration management
│   ├── hpa.yaml              # Horizontal Pod Autoscaler
│   └── README.md             # Deployment guide
├── docker/                   # Docker configuration
│   ├── Dockerfile            # Production-ready image
│   └── docker-compose.yml    # Local development
├── docs/                     # Documentation
│   ├── API_EXAMPLES.md       # API usage examples

├── models/                   # Model packages (.pkl files)
├── streamlit_app.py          # Interactive web UI
├── config.yaml               # Application configuration
├── requirements.txt          # Python dependencies
├── DESIGN_DECISIONS.md       # Architecture decisions
└── README.md                 # This file
```

---

## 🚀 Quick Start

### Option 1: Docker (Recommended for Local Testing)

```bash
# 1. Build the Docker image
docker build -f docker/Dockerfile -t mlops-inference:latest .

# 2. Run the container
docker run -d \
  --name ml-inference \
  -p 8000:8000 \
  -v $(pwd)/models:/app/models \
  -v $(pwd)/config.yaml:/app/config.yaml \
  mlops-inference:latest

# 3. Test the API
curl http://localhost:8000/health

# 4. Make a prediction
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This movie was fantastic!"}'
```

### Option 2: Docker Compose

```bash
# Start all services
docker-compose -f docker/docker-compose.yml up -d

# View logs
docker-compose -f docker/docker-compose.yml logs -f

# Stop services
docker-compose -f docker/docker-compose.yml down
```

### Option 3: Kubernetes

```bash
# 1. Update image in k8s/deployment.yaml (line 43)
# image: your-registry/mlops-inference:v1.0

# 2. Apply manifests
kubectl apply -f k8s/

# 3. Verify deployment
kubectl get pods -l app=mlops-inference
kubectl get svc mlops-inference-service

# 4. Port-forward for testing
kubectl port-forward service/mlops-inference-service 8080:80

# 5. Test endpoint
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "Great movie!"}'
```

See [k8s/README.md](k8s/README.md) for detailed Kubernetes deployment guide.

---

## 📡 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | API information |
| `/health` | GET | Health check (liveness & readiness) |
| `/predict` | POST | Single text classification |
| `/predict/batch` | POST | Batch text classification |
| `/predict/top-k` | POST | Top-K predictions |
| `/model/info` | GET | Model metadata |
| `/docs` | GET | Interactive API documentation |

### Example Request/Response

**Request:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This movie was absolutely fantastic!"}'
```

**Response:**
```json
{
  "text": "This movie was absolutely fantastic!",
  "predicted_class": 1,
  "label": "positive",
  "confidence": 0.9234,
  "is_confident": true,
  "probability_positive": 0.9234,
  "probability_negative": 0.0766,
  "probabilities": {
    "negative": 0.0766,
    "positive": 0.9234
  }
}
```

See [docs/API_EXAMPLES.md](docs/API_EXAMPLES.md) for more examples.

---

## 🎨 Streamlit Web App

Interactive web interface for testing the model:

```bash
# Install dependencies
pip install -r requirements.txt

# Run Streamlit app
streamlit run streamlit_app.py
```

Features:
- Single text prediction with confidence visualization
- Batch prediction from CSV/TXT files
- Threshold analysis and tuning
- Interactive charts and metrics

---

## ⚙️ Configuration

Edit `config.yaml` to customize:

```yaml
model:
  package_path: "models/cnn_model_package.pkl"
  threshold: 0.5

preprocessing:
  lowercase: true
  remove_special_chars: false
  remove_stopwords: false

api:
  host: "0.0.0.0"
  port: 8000
  log_level: "info"

inference:
  batch_size: 100
  confidence_threshold: 0.5
```

---

## 📊 Production Considerations

### Resource Management

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5 CPU cores
    memory: "1Gi"    # 1GB RAM
  limits:
    cpu: "2000m"     # 2 CPU cores
    memory: "2Gi"    # 2GB RAM
```

**Rationale:** Based on model size (~500MB) and typical inference workload (~10-20 req/s per pod).

### Scaling Strategy

- **Horizontal Pod Autoscaler**: Scales from 2 to 10 pods based on CPU (target: 70%)
- **Handles**: 10 req/s (2 pods) to 200 req/s (10 pods)
- **Cost-effective**: CPU-based inference for small models

### GPU Support

For larger models or higher throughput:
- Update Dockerfile to use CUDA base image
- Add GPU resource requests in deployment
- Use node selectors for GPU nodes

See [k8s/README.md](k8s/README.md#gpu-support) for details.

---

## 🔍 Monitoring & Observability

**Metrics to track:**
- Predictions per second
- Latency (p50, p95, p99)
- Error rate
- CPU/memory usage
- Pod restart count

**Recommended tools:**
- Prometheus for metrics collection
- Grafana for visualization
- ELK/EFK stack for log aggregation

---

## 🧪 Testing

```bash
# Run unit tests
pytest

# Test API locally
python test.py

# Load testing
ab -n 1000 -c 10 -p payload.json -T application/json \
  http://localhost:8000/predict
```

---

## 🔄 Deployment Workflow

```bash
# 1. Build new image
docker build -f docker/Dockerfile -t mlops-inference:v1.1 .

# 2. Push to registry
docker push your-registry/mlops-inference:v1.1

# 3. Update Kubernetes deployment
kubectl set image deployment/mlops-inference \
  inference-api=your-registry/mlops-inference:v1.1

# 4. Monitor rollout (zero downtime!)
kubectl rollout status deployment/mlops-inference

# 5. Rollback if needed
kubectl rollout undo deployment/mlops-inference
```

---

## 📚 Documentation

- **[DESIGN_DECISIONS.md](DESIGN_DECISIONS.md)**: Architecture, technology choices, and trade-offs
- **[k8s/README.md](k8s/README.md)**: Kubernetes deployment guide with GPU and scaling strategies
- **[docs/API_EXAMPLES.md](docs/API_EXAMPLES.md)**: API usage examples with curl and Python

---

## 🛠️ Technology Stack

- **API Framework**: FastAPI (async, auto-docs, type-safe)
- **ML Libraries**: TensorFlow/Keras, scikit-learn
- **Containerization**: Docker
- **Orchestration**: Kubernetes
- **Web UI**: Streamlit
- **Text Processing**: NLTK

---

## 🎯 Key Design Decisions

1. **Stateless Design**: Model loaded at startup, no persistent state
   - ✅ Simple, scalable, fast
   - ❌ Startup time ~30s (mitigated by readiness probes)

2. **CPU Inference**: Cost-effective for small models
   - ✅ $0.05/hour vs $0.50/hour for GPU
   - ❌ Lower throughput (acceptable for current scale)

3. **Rolling Updates**: Zero-downtime deployments
   - ✅ Gradual rollout with health checks
   - ❌ Slower than blue-green (acceptable trade-off)

See [DESIGN_DECISIONS.md](DESIGN_DECISIONS.md) for full analysis.

---

## 🚦 Production Checklist

- [x] Docker image optimized (slim base, layer caching)
- [x] Kubernetes manifests with resource limits
- [x] Health probes (liveness & readiness)
- [x] Horizontal autoscaling configured
- [x] Zero-downtime deployment strategy
- [x] API documentation (OpenAPI/Swagger)
- [x] Design document with trade-offs
- [x] Monitoring and alerting (Prometheus/Grafana)
- [ ] Log aggregation (ELK/EFK)
- [ ] Security scanning (Trivy, Snyk)
- [ ] CI/CD pipeline (GitHub Actions, GitLab CI)

---

## 📝 License

This project is for assessment purposes.

---

## 🤝 Contributing

This is an assessment project, but feedback is welcome!

---

## 📧 Contact

For questions about this assessment project, please reach out via the submission platform.
