# Production Incident: Checkout API Latency Investigation

## Incident Overview

Users reported significant performance degradation while placing orders through the website.

### Impact

- **Affected Service:** Checkout API
- **Severity:** High
- **Customer Impact:** Slow order confirmation during checkout.
- **Traffic Volume:** Normal (Prometheus confirmed there was no traffic spike.)

---

## Symptoms

| Metric | Value |
|---------|-------|
| P95 Latency | **4.8 seconds** |
| Normal Latency | **400–600 ms** |
| Error Rate | **0%** |
| Traffic | **Normal** |

---

## Architecture Flow

```text
User
   │
   ▼
Checkout Service
   │
   ▼
Inventory Service
   │
   ▼
Payment Service
   │
   ▼
Database
```

---

## Distributed Trace Analysis

Using **Grafana Tempo**, the request trace showed:

```text
checkout-service      0.5s
        │
        ▼
inventory-service     0.1s
        │
        ▼
payment-service       4.2s
```

The **payment-service** was responsible for most of the request latency.

---

# Environment Setup

## Step 1: Create Observability Namespace

```bash
kubectl create namespace observability
```

---

## Step 2: Install Tempo

```bash
helm install tempo grafana/tempo \
  --namespace observability \
  --set tempo.storage.trace.backend=local
```

---

## Step 3: Install Grafana

```bash
helm install grafana grafana/grafana \
  --namespace observability \
  --set adminPassword=admin \
  --set persistence.enabled=false
```

---

## Step 4: Deploy the Application

```bash
kubectl apply -f website-app.yaml
```

The application is instrumented using **OpenTelemetry** for automatic distributed tracing.

---

## Step 5: Generate Test Traffic

```bash
kubectl run tracer \
--rm -it \
--image=curlimages/curl \
-- http://checkout-service:8081/
```

Output

```text
Checkout Complete!

Total Time: 4.82s
```

---

# Investigation

## Prometheus Metrics

Infrastructure metrics were healthy.

- CPU Usage ✅ Normal
- Memory Usage ✅ Normal
- Network Traffic ✅ Normal
- Request Rate ✅ Normal

No infrastructure bottleneck was detected.

---

## Grafana Tempo

The trace clearly showed that the delay occurred inside the **payment-service**.

---

# Root Cause

The payment service waited for a slow response from an external payment gateway before returning the result.

Possible production causes include:

- Slow third-party payment gateway (Stripe, Razorpay, PayPal, etc.)
- Database connection pool exhaustion
- Slow SQL queries
- Missing database indexes
- Blocking synchronous HTTP calls
- Thread pool starvation
- High network latency

---

# Resolution

## Immediate Fix

Configured HTTP client timeout.

```text
Connection Timeout : 2 seconds

Read Timeout : 2 seconds
```

This prevents requests from waiting indefinitely.

---

## Long-Term Solution

Implemented asynchronous payment processing using a message broker.

```text
User
   │
   ▼
Checkout Service
   │
   ▼
RabbitMQ / Kafka
   │
   ▼
Payment Service
   │
   ▼
Database
```

Workflow:

1. Customer submits an order.
2. Checkout Service stores the order.
3. Payment event is published to RabbitMQ/Kafka.
4. API immediately returns **HTTP 202 Accepted**.
5. Payment Service processes the request asynchronously.
6. Order status is updated after payment completion.

---

# Monitoring Stack

| Tool | Purpose |
|------|---------|
| Kubernetes | Container orchestration |
| Prometheus | Metrics collection |
| Grafana | Dashboard visualization |
| Grafana Tempo | Distributed tracing |
| OpenTelemetry | Automatic instrumentation |
| Helm | Monitoring stack deployment |

---

# Outcome

- Identified the bottleneck using distributed tracing.
- Eliminated long-running synchronous requests.
- Reduced checkout latency.
- Improved application scalability.
- Increased system resilience against slow external services.

---

# Key Learnings

- Distributed tracing significantly reduces troubleshooting time.
- Infrastructure metrics alone cannot identify application-level bottlenecks.
- Configure aggressive timeouts for downstream services.
- Prefer asynchronous communication for long-running operations.
- Combine **Metrics + Logs + Traces** for complete observability.
