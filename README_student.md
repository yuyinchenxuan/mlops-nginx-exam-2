# MLOps Nginx Exam Project

## Project Structure
```
mlops-nginx-exam-2/
├── deployments/
│   ├── nginx/
│   │   ├── certs/
│   │   │   ├── nginx.crt        # SSL certificate
│   │   │   └── nginx.key        # SSL private key
│   │   ├── .htpasswd            # Basic auth credentials
│   │   ├── Dockerfile           # Nginx Docker image
│   │   └── nginx.conf           # Nginx configuration
│   └── prometheus/
│       └── prometheus.yml       # Prometheus configuration
├── model/
│   └── model.joblib             # Trained ML model
├── src/
│   └── api/
│       ├── v1/
│       │   ├── Dockerfile       # api-v1 Docker image
│       │   └── main.py          # Standard API
│       └── v2/
│           ├── Dockerfile       # api-v2 Docker image
│           ├── main.py          # Debug API
│           └── requirements.txt # Python dependencies
├── tests/
│   └── run_tests.sh             # Test script
├── docker-compose.yml           # Docker Compose configuration
├── Makefile                     # Makefile for common commands
└── README.md                    # This file
```


---

## Architecture Overview

                    ┌─────────────────────────────────────┐
                    │              Nginx                   │
                    │         (Reverse Proxy)              │
                    │                                      │
                    │  - HTTPS (port 443)                  │
                    │  - HTTP → HTTPS redirect (port 80)   │
                    │  - Basic Authentication              │
                    │  - Rate Limiting                     │
                    │  - A/B Testing                       │
                    └─────────────────────────────────────┘
                                    │
                ┌───────────────────┴───────────────────┐
                │                                       │
                ▼                                       ▼
    ┌───────────────────────┐             ┌────────────────────┐
    │       api-v1          │             │      api-v2         │
    │   (Load Balanced)     │             │   (Debug Version)   │
    │                       │             │                     │
    │  - mlops-api-v1-1     │             │  - Extra debug info │
    │  - mlops-api-v1-2     │             │  - Triggered by     │
    │  - mlops-api-v1-3     │             │    X-Experiment-    │
    └───────────────────────┘             │    Group: debug     │
                                          └────────────────────┘

    ┌───────────────────────┐             ┌────────────────────┐
    │      Prometheus       │────────────▶│      Grafana        │
    │  (Metrics Collection) │             │  (Visualization)    │
    └───────────────────────┘             └────────────────────┘

---

## Features

### 1. Reverse Proxy
- Nginx acts as the **single point of entry**
- All traffic goes through Nginx before reaching the API

### 2. Load Balancing
- `api-v1` runs with **3 replicas**
- Nginx distributes traffic across all replicas
- Ensures **high availability**

### 3. HTTPS Security
- All traffic is encrypted via **HTTPS (port 443)**
- HTTP traffic on port 80 is **automatically redirected** to HTTPS
- Uses **self-signed certificates**

### 4. Access Control
- `/predict` endpoint is protected by **Basic Authentication**
- Unauthorized requests return `HTTP 401`

### 5. Rate Limiting
- `/predict` endpoint is limited to **10 requests/second per IP**
- Excess requests return `HTTP 429`

### 6. A/B Testing
- Requests **without** `X-Experiment-Group` header → routed to `api-v1`
- Requests **with** `X-Experiment-Group: debug` header → routed to `api-v2`

### 7. Monitoring (Bonus)
- **Prometheus** collects Nginx metrics
- **Grafana** visualizes metrics on dashboards

---

## Prerequisites
- Docker
- Docker Compose
- curl
- openssl

---

## Setup & Usage

### 1. Start Services
make start-project
### 2. Run a test curl request
make test-api
### 3. Validate the project features
make test
### 4. Stop Services
make stop-project

## Example Request:
### Standard request → api-v1
curl -X POST "https://localhost/predict" \
     -H "Content-Type: application/json" \
     -d '{"sentence": "You are so cool!"}' \
     --user admin:admin \
     --cacert ./deployments/nginx/certs/nginx.crt

### Debug request → api-v2
curl -X POST "https://localhost/predict" \
     -H "Content-Type: application/json" \
     -H "X-Experiment-Group: debug" \
     -d '{"sentence": "You are so cool!"}' \
     --user admin:admin \
     --cacert ./deployments/nginx/certs/nginx.crt

## Example Response:
### Example Response (api-v1):
{
    "prediction value": "happiness"
}
### Example Response (api-v2):
{
    "prediction_value": "happiness",
    "prediction_proba_dict": {
        "anger":       0.0029  (0.29%),
        "boredom":     0.0039  (0.39%),
        "empty":       0.0067  (0.67%),
        "enthusiasm":  0.0195  (1.95%),
        "fun":         0.0750  (7.50%),
        "happiness":   0.4039  (40.39%),
        "hate":        0.0133  (1.33%),
        "love":        0.1461  (14.61%),
        "neutral":     0.1478  (14.78%),
        "relief":      0.0246  (2.46%),
        "sadness":     0.0356  (3.56%),
        "surprise":    0.0720  (7.20%),
        "worry":       0.0487  (4.87%)
    }
}


## Monitoring
| Service | URL | Credentials |
|---------|-----|-------------|
| Grafana | http://localhost:3000 | admin/admin |
| Prometheus | http://localhost:9090 | - |
