# Capstone Project: Robot Shop SRE & Observability Implementation

## 1. Deploy and Watch
* **Deployment**: Robot Shop microservices deployed via Kubernetes Helm charts and Docker Compose.
* **Load Generation**: Locust load generator (`load-gen/robot-shop.py`) simulating realistic user behavior (login, browsing catalogue, adding items to cart, updating quantities, calculating shipping, placing orders, and generating payment errors).
* **Pod Health**: Monitored via Kubernetes health checks ensuring all services (`web`, `cart`, `catalogue`, `payment`, `shipping`, `user`, `ratings`, `dispatch`, `mongodb`, `mysql`, `redis`, `rabbitmq`, `load`) are healthy and operational.

---

## 2. See the Metrics (Prometheus & Grafana)
* **Metrics Stack**: Installed `kube-prometheus-stack` via Helm (`monitoring` namespace).
* **Grafana Dashboard PromQL Queries**:
  * **Requests (QPS)**:
    ```promql
    sum(rate(http_requests_total{job=~"robot-shop.*"}[5m])) by (service)
    ```
  * **Errors (5xx Rate)**:
    ```promql
    sum(rate(http_requests_total{job=~"robot-shop.*", status=~"5.."}[5m])) by (service)
    ```
  * **Latency (P95 / P99)**:
    ```promql
    histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job=~"robot-shop.*"}[5m])) by (le, service))
    ```

---

## 3. Collect the Logs (Fluent Bit & Loki)
* **Log Pipeline**: Fluent Bit daemonset ingests container stdout/stderr logs from the `robot-shop` namespace and forwards them to Grafana Loki.
* **Searchability (LogQL)**:
  * By Service: `{service="cart"}`
  * By Error Level: `{service="payment"} |= "error"` or `{level="error"}`

---

## 4. Follow One Trace (OpenTelemetry & Jaeger)
* **Tracing Architecture**: OpenTelemetry SDK configured across microservices (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`).
* **Trace-to-Log Links**: Configured in Grafana data source settings so clicking a trace span ID instantly jumps to the corresponding log lines in Loki (`${__span.traceId}`).

---

## 5. Set One SLO, Error Budget Policy & Slack Alerts
* **SLO Definition**: **99.9% Availability** over a 30-day rolling window for API requests.
* **Error Budget Policy**:
  * **Budget**: 0.1% of total requests over 30 days.
  * **Action Thresholds**: If 50% of budget is consumed in 7 days, feature freeze. If 100% is consumed, all engineering efforts pivot to reliability remediation.
* **Burn Rate Alerts & Runbook**:
  * **Alert Rule (Prometheus / Alertmanager)**: Multi-window burn rate alert (e.g., burning 2% of budget in 1 hour).
  * **Slack Webhook Integration**: Alerts route to `#sre-alerts` with severity, burn rate, and a direct markdown link to the [Runbook](#).

---

## 6. Incident Drill, RCA & Postmortem
* **Drill Scenario**: Simulated database/payment gateway failure.
* **Mitigation**: Alert fired in Slack with runbook link -> engineer executed failover / restart procedure.
* **Root Cause Analysis (RCA)**: Connection pool exhaustion during traffic spike combined with unoptimized DB queries.
* **Postmortem**: Documented timeline, customer impact, root cause, and preventative action items (rate limiting, connection pooling tuning).

---

### Definition of Done Checklist
- [x] The alert fires during the drill
- [x] One request is traced
- [x] Logs are searchable by service and level
- [x] A runbook is linked from the alert
- [x] Burn rate alerts
- [x] Trace to log links
