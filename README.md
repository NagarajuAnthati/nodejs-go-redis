# nodejs-go-redis
# Kubernetes Monitoring, Performance Optimization, and Alerting

## **Overview**
This document provides details on setting up a centralized monitoring solution, performing backend load testing, and configuring alerts in a Kubernetes environment using Prometheus, Grafana, and Alertmanager.

---

## **1. Kubernetes Deployments & Configurations**

### **1.1 Backend Deployment (Falcon API)**
The Falcon backend was deployed using a Kubernetes Deployment and Service.

- **File:** [backend-deployment.yaml](backend-deployment.yaml)
- **Replicas:** 2
- **Exposed Port:** 4000 (mapped to service port 80)
- **Environment Variables:** Configured via ConfigMap

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falcon
spec:
  replicas: 2
  selector:
    matchLabels:
      app: falcon
  template:
    metadata:
      labels:
        app: falcon
    spec:
      containers:
        - name: falcon
          image: docker.io/nagarajuanthati/falcon:latest
          ports:
            - containerPort: 4000
          env:
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: REDIS_HOST
            - name: REDIS_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: REDIS_PORT
```

### **1.2 ConfigMap for Redis Configuration**
- **File:** [configmap.yaml](configmap.yaml)
- **Purpose:** Stores Redis service connection details

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6339"
```

---

## **2. Monitoring Setup (Prometheus & Grafana)**
### **2.1 Prometheus Configuration**
Prometheus was set up to monitor:
- **Kubernetes Nodes:** Scrapes Node Exporter metrics
- **Kubernetes Pods:** Scrapes container metrics
- **Redis Performance:** Tracks memory usage and hit/miss rate

**Prometheus Scrape Configuration:**
```yaml
scrape_configs:
  - job_name: 'kubernetes-nodes'
    static_configs:
      - targets: ['192.168.1.100:9100']  # Replace with your node's IP

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

### **2.2 Grafana Dashboard**
- **File:** [grafana-dashboard.json](grafana-dashboard.json)
- **Panels Included:**
  - **CPU Usage**: Tracks container CPU utilization
  - **Memory Usage**: Monitors container memory consumption
  - **HTTP Request Rate**: Observes API request traffic
  - **Redis Hit/Miss Rate**: Checks Redis cache efficiency

```json
{
  "dashboard": {
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)"
          }
        ]
      }
    ]
  }
}
```

### **2.3 Grafana Slack Integration**
To receive alerts in Slack:
1. **Go to Grafana → Alerting → Notification Channels**
2. **Select Type:** `Slack`
3. **Enter Webhook URL** (Generated from Slack API)
4. **Set Channel:** `#alerts`
5. **Test & Save**

---

## **3. Load Testing (Falcon API)**
We performed load testing using **k6** to simulate 100 users over 5 minutes.

### **Load Test Script (`k6-load-test.js`)**
```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export let options = {
  stages: [
    { duration: "1m", target: 20 },
    { duration: "2m", target: 50 },
    { duration: "2m", target: 100 }
  ]
};

export default function () {
  let res = http.get("http://falcon-service:80");
  check(res, {
    "status is 200": (r) => r.status === 200,
    "response time < 200ms": (r) => r.timings.duration < 200
  });
  sleep(1);
}
```

#### **Execution Steps**:
```sh
k6 run k6-load-test.js
```

---

## **4. Alerting Configuration (Prometheus + Alertmanager)**

### **4.1 Prometheus Alert Rules**
```yaml
groups:
- name: kubernetes-alerts
  rules:
  - alert: HighCpuUsage
    expr: (sum(rate(container_cpu_usage_seconds_total[1m])) by (container) / sum(machine_cpu_cores) by (node)) > 0.8
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High CPU usage detected on container {{ $labels.container }}"
```

### **4.2 Alertmanager Slack Integration**
```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  receiver: 'slack-notifications'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
    channel: '#alerts'
```

### **4.3 Apply Alert Rules**
```sh
kubectl apply -f prometheus-alerting-rules.yaml -n monitoring
```

### **4.4 Apply Alertmanager Configuration**
```sh
kubectl create secret generic alertmanager-secret --from-file=alertmanager.yaml=alertmanager-config.yaml -n monitoring
```

---

