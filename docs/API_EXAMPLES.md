# API Examples

This document provides example requests and responses for all API endpoints.

## Base URL

```
http://localhost:8000
```

For Kubernetes deployment:
```
http://mlops-inference-service
```

---

## Endpoints

### 1. Root Endpoint

**GET /**

Returns API information and available endpoints.

**Request:**
```bash
curl http://localhost:8000/
```

**Response:**
```json
{
  "message": "Text Classification API",
  "version": "1.0.0",
  "status": "healthy",
  "endpoints": {
    "health": "/health",
    "predict": "/predict",
    "predict_batch": "/predict/batch",
    "predict_top_k": "/predict/top-k",
    "model_info": "/model/info",
    "docs": "/docs"
  }
}
```

---

### 2. Health Check

**GET /health**

Check if the service is healthy.

**Request:**
```bash
curl http://localhost:8000/health
```

**Response:**
```json
{
  "status": "healthy",
  "model_loaded": true,
  "device": "cpu"
}
```

---

### 3. Single Prediction

**POST /predict**

Classify a single text input.

**Request:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "text": "This movie was absolutely fantastic! The acting was superb."
  }'
```

**Python example:**
```python
import requests

url = "http://localhost:8000/predict"
payload = {
    "text": "This movie was absolutely fantastic! The acting was superb."
}

response = requests.post(url, json=payload)
print(response.json())
```

**Response:**
```json
{
  "text": "This movie was absolutely fantastic! The acting was superb.",
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

**Negative example:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Terrible movie. Complete waste of time and money."
  }'
```

**Response:**
```json
{
  "text": "Terrible movie. Complete waste of time and money.",
  "predicted_class": 0,
  "label": "negative",
  "confidence": 0.8912,
  "is_confident": true,
  "probability_positive": 0.1088,
  "probability_negative": 0.8912,
  "probabilities": {
    "negative": 0.8912,
    "positive": 0.1088
  }
}
```

---

### 4. Batch Prediction

**POST /predict/batch**

Classify multiple texts in a single request.

**Request:**
```bash
curl -X POST http://localhost:8000/predict/batch \
  -H "Content-Type: application/json" \
  -d '{
    "texts": [
      "This movie was fantastic!",
      "Terrible film, waste of time.",
      "Pretty good, worth watching."
    ],
    "return_probabilities": true
  }'
```

**Python example:**
```python
import requests

url = "http://localhost:8000/predict/batch"
payload = {
    "texts": [
        "This movie was fantastic!",
        "Terrible film, waste of time.",
        "Pretty good, worth watching."
    ],
    "return_probabilities": True
}

response = requests.post(url, json=payload)
result = response.json()

print(f"Processed {result['total_processed']} texts")
for pred in result['predictions']:
    print(f"{pred['label']}: {pred['text'][:50]}... (confidence: {pred['confidence']:.2f})")
```

**Response:**
```json
{
  "predictions": [
    {
      "text": "This movie was fantastic!",
      "predicted_class": 1,
      "label": "positive",
      "confidence": 0.9123,
      "is_confident": true,
      "probability_positive": 0.9123,
      "probability_negative": 0.0877,
      "probabilities": {
        "negative": 0.0877,
        "positive": 0.9123
      }
    },
    {
      "text": "Terrible film, waste of time.",
      "predicted_class": 0,
      "label": "negative",
      "confidence": 0.8756,
      "is_confident": true,
      "probability_positive": 0.1244,
      "probability_negative": 0.8756,
      "probabilities": {
        "negative": 0.8756,
        "positive": 0.1244
      }
    },
    {
      "text": "Pretty good, worth watching.",
      "predicted_class": 1,
      "label": "positive",
      "confidence": 0.7234,
      "is_confident": true,
      "probability_positive": 0.7234,
      "probability_negative": 0.2766,
      "probabilities": {
        "negative": 0.2766,
        "positive": 0.7234
      }
    }
  ],
  "total_processed": 3
}
```

---

### 5. Top-K Predictions

**POST /predict/top-k?k=2**

Get the top K most likely classes for a text.

**Request:**
```bash
curl -X POST "http://localhost:8000/predict/top-k?k=2" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "This movie had some good moments but overall disappointing."
  }'
```

**Response:**
```json
{
  "text": "This movie had some good moments but overall disappointing.",
  "top_predictions": [
    {
      "label": "negative",
      "probability": 0.5623
    },
    {
      "label": "positive",
      "probability": 0.4377
    }
  ]
}
```

---

### 6. Model Information

**GET /model/info**

Get information about the loaded model.

**Request:**
```bash
curl http://localhost:8000/model/info
```

**Response:**
```json
{
  "model_type": "Sequential",
  "num_classes": 2,
  "classes": [
    "negative",
    "positive"
  ],
  "device": "cpu",
  "max_input_length": null
}
```

---

## Error Responses

### 503 Service Unavailable

Model not loaded yet.

**Response:**
```json
{
  "detail": "Model not loaded. Please check server logs."
}
```

### 500 Internal Server Error

Prediction failed.

**Response:**
```json
{
  "detail": "Prediction failed: <error message>"
}
```

### 422 Validation Error

Invalid request format.

**Request:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "invalid_field": "value"
  }'
```

**Response:**
```json
{
  "detail": [
    {
      "loc": ["body", "text"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
