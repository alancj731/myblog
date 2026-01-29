---
title: Create a Low Cost Cluster
date: "2026-01-29T13:14:10.440Z"
description: Deploy a cluster with one medium size node
---

# Create a Low Cost GKE Cluster

### This document describes how to create a minimal-cost GKE cluster and deploy to it using GitHub Actions.

---

## Create a Low Cost Cluster

```bash
gcloud container clusters create "medium-cost-cluster" \
    --zone "us-central1-a" \
    --num-nodes=1 \
    --machine-type "e2-medium" \
    --spot \
    --disk-type "pd-standard" \
    --disk-size "30GB" \
    --release-channel "stable"
```

## Create a New Artifact Registry Repository

```bash
gcloud artifacts repositories create my-simple-repo \
    --repository-format=docker \
    --location=us-central1
```

## Create a Workload Identity Pool

```bash
gcloud iam workload-identity-pools create "github-pool" --location="global"
```

### Create a Workload Identity Provider

```bash
gcloud iam workload-identity-pools providers create-oidc "my-repo-provider" \
    --location="global" \
    --workload-identity-pool="github-pool" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
    --attribute-condition="attribute.repository_owner == 'alancj731'"
```

## Create a Service Account

```bash
gcloud iam service-accounts create "github-actions-sa" \
    --display-name="GitHub Actions Service Account"
```
<br>

```bash
gcloud iam workload-identity-pools list \
    --project="plasma-set-485717-s0" \
    --location="global"
```

## GitHub Actions Secrets and Variables
#### Set the following values in:
##### **Repository → Settings → Secrets and variables → Actions**

#### GCP_PROJECT_ID:
use this commmand to show your project id and number
```bash
gcloud projects list
```

#### GCP_SA_EMAIL:
###### Format: 
**{SERVICE_ACCOUNT_NAME}@{PROJECT_ID}.iam.gserviceaccount.com**

#### GCP_WORKLOAD_IDENTITY_PROVIDER:
###### Format: 
**projects/{PROJECT_NUMBER}/locations/global/workloadIdentityPools/{POOL_ID}/providers/{PROVIDER_ID}**

###### You can also use this command to show your providers
```bash
gcloud iam workload-identity-pools providers list \
    --location="global" \
    --workload-identity-pool="YOUR_POOL_ID" \
    --format="value(name)"
```

## GitHub Actions Workflow 
**File:**.github/workflows/deployment.yaml

```yaml
name: Deploy to GKE
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: "read"
      id-token: "write"

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # 1. Authenticate to Google Cloud
      - id: "auth"
        uses: "google-github-actions/auth@v2"
        with:
          workload_identity_provider: "${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}"
          service_account: "${{ secrets.GCP_SA_EMAIL }}"

      # 2. Get GKE Credentials
      - name: Set up GKE credentials
        uses: "google-github-actions/get-gke-credentials@v2"
        with:
          cluster_name: "medium-cost-cluster"
          location: "us-central1-a"

      # 3. Build and Push Image (Using Artifact Registry)
      - name: Build and Push Docker Image
        run: |
          gcloud auth configure-docker us-central1-docker.pkg.dev
          docker build -t us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-simple-repo/my-app:${{ github.sha }} .
          docker push us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-simple-repo/my-app:${{ github.sha }}

      # 4. Deploy to Cluster
      - name: Deploy to GKE
        run: |
          # This replaces the 'IMAGE_PLACEHOLDER' in your deployment.yaml with the real image path
          sed -i "s|IMAGE_PLACEHOLDER|us-central1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/my-simple-repo/my-app:${{ github.sha }}|g" k8s/deployment.yaml
          kubectl apply -f k8s/
```

## Kubernetes Deployment
**File:** k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: web-container
        image: IMAGE_PLACEHOLDER  # The CI/CD script will replace this
        resources:
          requests:
            cpu: "10m"      # Minimal reservation to fit on cheap nodes
            memory: "32Mi"
          limits:
            cpu: "500m"     # High limit so it doesn't get slow
            memory: "256Mi" # High limit so it doesn't crash
---
apiVersion: v1
kind: Service
metadata:
  name: my-web-app-service
spec:
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```

## Kubernetes Service (LoadBalancer) 
**File:** k8s/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-app-service
  annotations:
    # This line tells GKE to look for a Standard Tier IP
    cloud.google.com/network-tier: "Standard"
spec:
  type: LoadBalancer
  loadBalancerIP: "35.208.88.209"
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

## Kubernetes Ingress
**File:** k8s/ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-web-app-ingress
  annotations:
    # Use the NAME of your GLOBAL static IP
    kubernetes.io/ingress.global-static-ip-name: "my-cheap-ip"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-web-app-service
            port:
              number: 80
```