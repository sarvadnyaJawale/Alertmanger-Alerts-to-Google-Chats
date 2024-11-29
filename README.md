# Alertmanger-Alerts-to-Google-Chats
Kubernetes Monitoring Integration with Google Chat Using Calert
Overview
This solution integrates Prometheus Alertmanager with Google Chat using Calert, a service that transforms the fixed Alertmanager webhook payload into the format Google Chat expects. It follows Kubernetes best practices for deployment, monitoring, and resiliency.

## 1. Architecture Diagram

[Prometheus] → [Alertmanager] → [Calert Service] → [Google Chat Webhook]

Chatgpt link https://chatgpt.com/share/674987bd-69f0-8013-9ee2-a28acb8710d1

## 2. Key Components and Setup Steps
### A. Alertmanager Configuration (alertmanager.yaml)
'''
global:
  resolve_timeout: 5m

route:
  receiver: 'calert-service'

receivers:
  - name: 'calert-service'
    webhook_configs:
      - url: 'http://calert-service.default.svc.cluster.local:8080/webhook'
        send_resolved: true
''' 

Explanation:

webhook_configs.url: Points to the Calert service running in Kubernetes.
send_resolved: Ensures notifications for resolved alerts are sent.

B. Calert Service Deployment (calert-deployment.yaml)
''' 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calert-service
  labels:
    app: calert
spec:
  replicas: 2
  selector:
    matchLabels:
      app: calert
  template:
    metadata:
      labels:
        app: calert
    spec:
      containers:
        - name: calert
          image: your-calert-image:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          env:
            - name: GOOGLE_CHAT_WEBHOOK_URL
              valueFrom:
                secretKeyRef:
                  name: google-chat-webhook
                  key: webhook-url
---
apiVersion: v1
kind: Service
metadata:
  name: calert-service
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: calert
'''
Explanation:

Deployment: Ensures scalability with replicas: 2.
Health Checks: Uses livenessProbe and readinessProbe for service health.
Resource Requests/Limits: Optimized for lightweight, scalable deployment.
Environment Variables: GOOGLE_CHAT_WEBHOOK_URL for the Google Chat webhook URL.
C. Google Chat Webhook Setup
Open Google Chat.
Select the room where alerts will be posted.
Click on Integrations > Webhooks > Manage Webhooks.
Click Add Webhook, provide a name, and copy the generated URL.
Store the URL in a Kubernetes Secret:
yaml
Copy code
apiVersion: v1
kind: Secret
metadata:
  name: google-chat-webhook
type: Opaque
data:
  webhook-url: <base64_encoded_webhook_url>
3. JSON Payload Transformation (Example)
Alertmanager JSON Payload (Input):

json
Copy code
{
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "HighCPUUsage",
        "severity": "critical",
        "instance": "node-1"
      },
      "annotations": {
        "summary": "CPU usage is above 90% on node-1."
      },
      "startsAt": "2024-11-28T12:00:00Z"
    }
  ]
}
Google Chat Payload (Output):

json
Copy code
{
  "text": "🚨 *HighCPUUsage* on *node-1* \nSeverity: *critical* \nSummary: CPU usage is above 90% on node-1."
}
4. Best Practices for Resiliency and Monitoring
A. Configuration for Alertmanager Resiliency
Retry Logic: Ensure retries are handled by Alertmanager's default retry mechanism. Use exponential backoff.
Load Balancer: Use a Kubernetes Service to distribute load across Calert replicas.
B. Monitoring and Logging
Logging: Enable structured logging in Calert.
Prometheus Metrics: Add /metrics endpoint in Calert for Prometheus to scrape.
Dashboards: Use Grafana to visualize webhook performance and error rates.
C. Error Handling in Calert
Retry Mechanism: Implement retries with exponential backoff in case Google Chat webhook fails.
Logging: Log each transformation attempt, successful or failed, with relevant metadata.
5. Validation Steps (End-to-End Testing)
Trigger Alert: Manually create a test alert in Prometheus:
yaml
Copy code
- alert: TestHighCPU
  expr: vector(1)
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Test alert for high CPU usage."
Check Alertmanager Logs: Verify webhook delivery.
Monitor Calert Logs: Ensure payload transformation and forwarding.
Check Google Chat: Confirm alert message is posted.
6. Conclusion
This setup ensures seamless transformation of Prometheus alerts for Google Chat using Calert, adhering to Kubernetes best practices with robust monitoring, logging, and error-handling mechanisms.

---

To integrate Alertmanager with Google Chat using Calert, here’s a detailed setup and configuration guide based on the provided documentation. We'll modify configurations, ensure resiliency, and adhere to Kubernetes best practices.

1. Alertmanager Configuration (alertmanager.yml)
Ensure Alertmanager sends webhook alerts to Calert's endpoint.

yaml
Copy code
global:
  resolve_timeout: 5m

route:
  receiver: 'calert'
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 15m
  group_by: ['alertname', 'severity']

receivers:
  - name: 'calert'
    webhook_configs:
      - url: 'http://calert:6000/dispatch'
        send_resolved: true
2. Calert Configuration (config.toml)
Configure Calert to transform Alertmanager alerts to the Google Chat format.

toml
Copy code
[app]
address = "0.0.0.0:6000"
server_timeout = "5s"
enable_request_logs = true
log = "info"

[providers.prod_alerts]  # Matches Alertmanager receiver name
type = "google_chat"
endpoint = "https://chat.googleapis.com/v1/spaces/XXXXXXXX/messages?key=XXXX&token=XXXX"  # Google Chat webhook URL
max_idle_conns = 50
timeout = "7s"
template = "/etc/calert/message.tmpl"  # Custom message template path
thread_ttl = "12h"
3. Kubernetes Deployment for Calert (calert-deployment.yaml)
Deploy Calert as a Kubernetes service.

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calert
  labels:
    app: calert
spec:
  replicas: 2
  selector:
    matchLabels:
      app: calert
  template:
    metadata:
      labels:
        app: calert
    spec:
      containers:
      - name: calert
        image: ghcr.io/mr-karan/calert:latest
        ports:
          - containerPort: 6000
        env:
          - name: CALERT_APP__ADDRESS
            value: "0.0.0.0:6000"
          - name: CALERT_PROVIDERS__PROD_ALERTS__ENDPOINT
            value: "<Google Chat Webhook URL>"
        volumeMounts:
          - name: message-template
            mountPath: /etc/calert/message.tmpl
            subPath: message.tmpl
        readinessProbe:
          httpGet:
            path: /healthz
            port: 6000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 6000
          initialDelaySeconds: 10
          periodSeconds: 30
      volumes:
        - name: message-template
          configMap:
            name: calert-template
---
apiVersion: v1
kind: Service
metadata:
  name: calert
spec:
  ports:
    - port: 6000
      targetPort: 6000
  selector:
    app: calert
4. ConfigMap for Message Template (calert-configmap.yaml)
Create a ConfigMap for the custom message template.

yaml
Copy code
apiVersion: v1
kind: ConfigMap
metadata:
  name: calert-template
data:
  message.tmpl: |
    {
      "text": "{{ .Alerts | joinAlerts }}",
      "cards": [
        {
          "header": {
            "title": "{{ .CommonLabels.alertname }}",
            "subtitle": "{{ .CommonLabels.severity }} - {{ .GroupLabels }}"
          },
          "sections": [
            {
              "widgets": [
                {
                  "textParagraph": {
                    "text": "{{ range .Alerts }}Alert: {{ .Annotations.description }}<br>{{ end }}"
                  }
                }
              ]
            }
          ]
        }
      ]
    }
5. Google Chat Webhook Setup
Go to Google Chat and create a new space.
Add a Google Chat webhook by going to Manage Webhooks.
Copy the provided webhook URL and replace it in the Calert config.toml.
6. Best Practices
Error Handling & Retries: Calert internally handles retries. Ensure timeout is set appropriately.
Scaling: Start with replicas: 2 and adjust as per load.
Logging: Enable verbose logging (log = "debug") during initial setup.
Health Checks: Use /healthz for readiness and liveness probes.
Metrics Monitoring: Expose Prometheus metrics at /metrics and configure Prometheus to scrape them.
7. Validation Steps
Trigger a test alert in Prometheus.
Verify that Alertmanager sends alerts to Calert (http://calert:6000/dispatch).
Confirm alerts appear in Google Chat.
Check logs in Calert for successful dispatch.
Use kubectl logs and kubectl get pods to verify health.
This setup ensures a resilient, scalable, and production-grade alerting flow from Prometheus to Google Chat via Calert.
