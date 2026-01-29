---
title: Get a Persistent Static IP for AWS Spot Instances
date: "2026-01-28T23:24:12-05:00"
description: Is it possible to maitian a consistent ip address when using AWS Spot Instances
---

### 💡 Create a static ip address
```
gcloud compute addresses create my-cheap-ip \
    --region=us-central1 \
    --network-tier=STANDARD
```
output the reserved static IP address
```
gcloud compute addresses describe my-cheap-ip \
--region=us-central1 --format="get(address)"
35.208.88.209
```

### 🌐 Assign Static IP to LoadBalancer
in service.yaml, set like this:
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

### 🚀 Benefits of Static IPs for Spot Clusters

#### 1. Persistent Access Points
Spot instances are ephemeral and can be reclaimed at any time. A static IP ensures your users always have a **consistent entry point**, even if the underlying instance is replaced.

#### 2. DNS Stability
Avoid the hassle of updating DNS records every time an instance rotates.
* **No Propagation Lag:** Your A-record stays the same.
* **Reliability:** No "down time" caused by DNS caching.

#### 3. Cost Optimization
Combining **Spot Instances** (up to 90% savings) with a **Standard Tier IP** provides a production-grade entry point at a hobbyist price.