# Microservices Architecture with Istio & API Gateway

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│         Istio Service Mesh                           │
│  ┌──────────────────────────────────────────────┐   │
│  │ Istio Ingress Gateway (API Gateway)          │   │
│  │ - Routes traffic to microservices             │   │
│  │ - Circuit breaking, retries, timeouts         │   │
│  └──────────┬───────────┬────────────┬─────────┘    │
│             │           │            │               │
│  ┌──────────▼──┐ ┌──────▼───┐ ┌────▼──────┐       │
│  │ Transits    │ │  Tarot   │ │  Events   │       │
│  │ Service     │ │ Service  │ │ Service   │       │
│  │ (Port 8001) │ │(Port8002)│ │(Port8003) │       │
│  │ 2 Replicas  │ │ 2 Replicas│ 2 Replicas│       │
│  └─────────────┘ └──────────┘ └───────────┘       │
│             ▲           ▲            ▲               │
│             └─────┬─────┴────┬───────┘               │
│                   │          │                       │
│  ┌────────────────▼──────────▼──────────────────┐   │
│  │         Frontend (Vite)                      │   │
│  │  - Calls: /api/transits, /api/tarot,        │   │
│  │           /api/events via Istio gateway     │   │
│  └─────────────────────────────────────────────┘   │
│                                                      │
│  Observability:                                     │
│  - Kiali: Service topology visualization            │
│  - Prometheus: Metrics collection                   │
│  - Jaeger: Distributed tracing                      │
└─────────────────────────────────────────────────────┘
```

---

## Part 1: Install Istio

### 1.1 Install Istio (one-time setup)

```powershell
# Download Istio
curl -L https://istio.io/downloadIstio | sh
cd istio-*

# Install Istio demo profile (includes observability)
.\bin\istioctl install --set profile=demo -y

# Verify installation
kubectl get ns | findstr istio
kubectl get pods -n istio-system
```

**Expected output:**
```
istio-system            Active   10m
istiod                  1/1 Running
istio-ingressgateway    1/1 Running
istio-egressgateway     1/1 Running
prometheus              1/1 Running
grafana                 1/1 Running
jaeger                  1/1 Running
kiali                   1/1 Running
```

### 1.2 Enable sidecar injection

Istio automatically injects Envoy sidecars into pods for service mesh traffic management:

```powershell
# Label default namespace for automatic sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify
kubectl get namespace -L istio-injection
```

---

## Part 2: Deploy Microservices

### 2.1 Deploy microservices (Transits, Tarot, Events)

```powershell
cd "C:\Users\eyuel\Dropbox\My PC (DESKTOP-OJ4TQI2)\Desktop\cosmos-app"

# Apply microservices
kubectl apply -f k8s-microservices.yaml

# Verify all 3 services are running
kubectl get pods
# Expected: transits-service-xxx (2 replicas)
#           tarot-service-xxx (2 replicas)
#           events-service-xxx (2 replicas)

kubectl get svc
# Expected: transits (8001)
#           tarot (8002)
#           events (8003)
```

### 2.2 Apply Istio Gateway & Virtual Services

```powershell
# Apply Istio configuration
kubectl apply -f k8s-istio-gateway.yaml

# Verify Gateway created
kubectl get gateway
kubectl describe gateway cosmos-gateway

# Verify Virtual Services
kubectl get virtualservice
kubectl describe vs transits
```

### 2.3 Apply Network Policies (Security)

```powershell
# Apply network policies
kubectl apply -f k8s-network-policies.yaml

# Verify
kubectl get networkpolicies
```

---

## Part 3: Test Microservices

### 3.1 Get Istio Ingress Gateway External IP

```powershell
# Find ingress gateway service
kubectl get svc -n istio-system istio-ingressgateway

# Get the EXTERNAL-IP (or use localhost if minikube)
$INGRESS_IP = kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
$INGRESS_PORT = kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.ports[0].port}'

echo "Ingress Gateway: $INGRESS_IP:$INGRESS_PORT"
```

### 3.2 Test each microservice

**Test Transits Service:**
```powershell
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80 &

Start-Sleep -Seconds 2

Invoke-WebRequest http://localhost:8080/api/transits -UseBasicParsing
# Expected: Array of transit data (Mercury, Venus, Earth, Sun, Moon, Saturn)

Invoke-WebRequest http://localhost:8080/api/transits/Mercury -UseBasicParsing
# Expected: Single transit object
```

**Test Tarot Service:**
```powershell
Invoke-WebRequest http://localhost:8080/api/tarot -UseBasicParsing
# Expected: Array of tarot cards

Invoke-WebRequest http://localhost:8080/api/tarot/draw -UseBasicParsing
# Expected: Random tarot card
```

**Test Events Service:**
```powershell
Invoke-WebRequest http://localhost:8080/api/events -UseBasicParsing
# Expected: Array of astro events (Lunar Eclipse, Meteor Shower, etc.)
```

### 3.3 Test service-to-service communication

```powershell
# Exec into a transits pod and call tarot service
$TRANSITS_POD = kubectl get pod -l app=transits -o jsonpath='{.items[0].metadata.name}'

kubectl exec -it $TRANSITS_POD -- curl -s http://tarot:8002/api/tarot | head -20
# Expected: Tarot cards data (proves service-to-service works)
```

---

## Part 4: Istio Observability

### 4.1 Open Kiali Dashboard (Service Topology)

```powershell
# Port-forward Kiali
kubectl port-forward -n istio-system svc/kiali 20000:20000 &

# Open browser: http://localhost:20000/kiali
# Default credentials: admin / admin
```

**In Kiali:**
- Go to **Graph** tab
- Select **default** namespace
- Watch live traffic flowing between services
- See error rates, latencies, throughput

### 4.2 Open Grafana (Metrics Dashboard)

```powershell
# Port-forward Grafana
kubectl port-forward -n istio-system svc/grafana 3000:3000 &

# Open browser: http://localhost:3000
# Default credentials: admin / admin
```

**Dashboards:**
- Istio Mesh Dashboard (overall metrics)
- Istio Service Dashboard (per-service metrics)
- Istio Workload Dashboard (per-pod metrics)

### 4.3 Open Jaeger (Distributed Tracing)

```powershell
# Port-forward Jaeger
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686 &

# Open browser: http://localhost:16686
```

**In Jaeger:**
- Select service from dropdown
- View end-to-end request traces
- See latency breakdown across services

### 4.4 View Prometheus Metrics

```powershell
# Port-forward Prometheus
kubectl port-forward -n istio-system svc/prometheus 9090:9090 &

# Open browser: http://localhost:9090
```

**Sample queries:**
```
# Request rate (requests/sec)
rate(istio_request_total[1m])

# Request latency (p95)
histogram_quantile(0.95, rate(istio_request_duration_milliseconds_bucket[1m]))

# Error rate
rate(istio_request_total{response_code="500"}[1m])
```

---

## Part 5: Advanced Istio Features

### 5.1 Circuit Breaker (Automatic Failover)

Already configured in **k8s-istio-gateway.yaml** via **DestinationRules**:
- **Outlier Detection**: Removes unhealthy pods from load balancing
- **Max Connections**: Limits concurrent connections (100)
- **Retries**: Auto-retries failed requests (3 attempts)
- **Timeout**: Kills slow requests (10s timeout)

### 5.2 Traffic Weighting (Canary Deployments)

Update **VirtualService** to route 90% to stable, 10% to canary:

```yaml
http:
- match:
  - uri:
      prefix: "/api/transits"
  route:
  - destination:
      host: transits
      subset: v1  # stable version
    weight: 90
  - destination:
      host: transits
      subset: v2  # canary version
    weight: 10
```

### 5.3 Request Mirroring (Shadow Traffic)

Route duplicate traffic to test version without affecting users:

```yaml
http:
- route:
  - destination:
      host: transits
      subset: v1
    weight: 100
  mirror:
    host: transits
    subset: v2
  mirrorPercent: 50  # mirror 50% of traffic
```

### 5.4 Rate Limiting

Apply rate limiting via **RequestAuthentication** + **AuthorizationPolicy**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Telemetry
metadata:
  name: rate-limit
spec:
  metrics:
  - providers:
    - name: prometheus
  dimensions:
  - request.path
  - response.code
```

---

## Part 6: Production Deployment

### 6.1 Update Frontend to use Istio Gateway

**In cosmos-app frontend (JavaScript/Vite):**

```javascript
// Before: Direct API calls
fetch('http://cosmos-api:5000/api/transits')

// After: Via Istio Gateway
const GATEWAY_URL = process.env.VITE_GATEWAY_URL || 'http://localhost:8080'

fetch(`${GATEWAY_URL}/api/transits`)
fetch(`${GATEWAY_URL}/api/tarot/draw`)
fetch(`${GATEWAY_URL}/api/events`)
```

### 6.2 Configure custom domain with TLS

Update **k8s-istio-gateway.yaml**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cosmos-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: cosmos-tls  # Created from cert-manager
    hosts:
    - "cosmos.yourdomain.com"
```

### 6.3 PodDisruptionBudget (High Availability)

Ensure at least 1 pod always running:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: transits-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: transits
```

---

## Part 7: Monitoring & Alerting

### 7.1 Create Prometheus Alert Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cosmos-alerts
spec:
  groups:
  - name: cosmos-microservices
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: rate(istio_request_total{response_code="500"}[5m]) > 0.05
      for: 5m
      annotations:
        summary: "High error rate detected"
    
    - alert: HighLatency
      expr: histogram_quantile(0.95, istio_request_duration_milliseconds_bucket) > 1000
      for: 5m
      annotations:
        summary: "P95 latency above 1 second"
```

### 7.2 Set Grafana Alerts

1. Open Grafana dashboard
2. Create **Alert** on metric
3. Set threshold (e.g., error rate > 5%)
4. Configure notification channel (Slack, PagerDuty, etc.)

---

## Part 8: Troubleshooting

### Issue: Sidecar not injected

```powershell
# Check pod for envoy sidecar
kubectl describe pod <pod-name>

# Look for: "istio-proxy" container
# If missing: Check namespace label
kubectl get namespace -L istio-injection
```

### Issue: Traffic not reaching service

```powershell
# Check VirtualService routing rules
kubectl describe vs transits

# Check DestinationRule
kubectl describe dr transits

# Test sidecar logs
kubectl logs <pod-name> -c istio-proxy

# Test connectivity
kubectl exec -it <pod-name> -- curl -v http://tarot:8002/api/tarot
```

### Issue: High latency

```powershell
# Check Jaeger traces for bottleneck
# Check Grafana for CPU/memory usage
kubectl top pods

# Check network policies not blocking
kubectl get networkpolicies
kubectl describe networkpolicies allow-frontend-to-services
```

---

## Part 9: Commands Cheat Sheet

```powershell
# Istio Management
istioctl install --set profile=demo -y
istioctl uninstall --purge
kubectl label namespace default istio-injection=enabled

# Istio Objects
kubectl get gateway
kubectl get virtualservice
kubectl get destinationrule
kubectl describe vs transits

# Port-forward
kubectl port-forward -n istio-system svc/kiali 20000:20000
kubectl port-forward -n istio-system svc/grafana 3000:3000
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686
kubectl port-forward -n istio-system svc/prometheus 9090:9090
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80

# Debugging
istioctl analyze
kubectl logs <pod-name> -c istio-proxy
kubectl exec -it <pod-name> -- curl http://tarot:8002/api/tarot
```

---

## 📚 Resources

- Istio Docs: https://istio.io/latest/docs/
- Kiali: https://kiali.io/
- Jaeger: https://www.jaegertracing.io/
- Prometheus: https://prometheus.io/

---

**Status**: ✅ Microservices + Istio ready for deployment

**Last Updated**: September 2026
