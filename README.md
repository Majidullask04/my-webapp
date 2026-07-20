# my-webapp

A simple Flask web application containerized with Docker and deployable to Kubernetes.

## Features
- Flask REST API
- Docker containerization
- Kubernetes deployment configuration
- Kubernetes service configuration

## Prerequisites
- Python 3.9+
- Docker
- Kubernetes cluster (for k8s deployment)
- kubectl CLI (for k8s deployment)

## Local Development

### Install dependencies
```bash
pip install -r requirements.txt
```

### Run the app
```bash
python app.py
```

The app will be available at `http://localhost:5000`

## Docker

### Build the image
```bash
docker build -t my-webapp .
```

### Run the container
```bash
docker run -p 5000:5000 my-webapp
```

## Kubernetes Deployment

### Deploy to cluster
```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### Check status
```bash
kubectl get deployments
kubectl get services
```

### Access the app
```bash
kubectl port-forward svc/my-webapp-service 5000:80
```

## Project Structure
```
.
├── app.py                 # Flask application
├── requirements.txt       # Python dependencies
├── Dockerfile            # Docker configuration
├── README.md             # This file
└── k8s/
    ├── deployment.yaml   # Kubernetes deployment config
    └── service.yaml      # Kubernetes service config
```

## Author
Majid Ullah
