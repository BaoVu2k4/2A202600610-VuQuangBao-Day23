# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Vu Quang Bao
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/BaoVu2k4/2A202600610-VuQuangBao-Day23

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 7.43 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: D:\AI_THUCCHIEN\NGAY23\2A202600610-VuQuangBao-Day23\00-setup\setup-report.json
```

(Ports show as BOUND because the stack is already running — this is expected, not a failure.)

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` with `docker stop day23-app` | — |
| T0+90s | `ServiceDown` fired (severity=critical, receiver=slack-critical) | screenshot `alertmanager-firing.png` |
| T1 | restored app with `docker start day23-app` | — |
| T1+60s | alert resolved (Alertmanager shows Resolved state) | — |

### One thing surprised me about Prometheus / Grafana

The datasource UID mismatch was a hidden trap: Grafana auto-generates a random UID like `PBFA97CFB590B2093` when no `uid:` field is set in the provisioning YAML, but the dashboard JSON hard-codes `uid: "prometheus"`. Every panel silently showed "No data" until I added explicit `uid: prometheus` to `datasources.yml` and restarted Grafana. The fix was one line — but finding the root cause required comparing the provisioned datasource config against the dashboard JSON panel by panel.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

Trace ID: `2feae2ab9c2e45cd9bc3cb2be279946f` — 4 spans, depth 2, duration 167.64ms.

### Log line correlated to trace

```
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 62, "quality": 0.794, "duration_seconds": 0.1746, "trace_id": "b521526de9e94511b711a6e981756d98", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T03:26:18.198644Z"}
```

The `trace_id` field in the structured log links directly to the corresponding trace in Jaeger. Loki's derived field config (`matcherRegex: '"trace_id":"([a-fA-F0-9]+)"'`) creates a clickable link from each log line to the Jaeger trace detail view.

### Tail-sampling math

The OTel collector ran at roughly 200 spans/second during the 60-second load test (≈11 000 spans received total / 55s ≈ 200 spans/s). With 4–5 spans per trace, that is about 40–50 traces/sec.

Sampling math:
- `keep-errors` policy: keeps all traces where any span has status ERROR → ~1% of production traffic (rare errors)
- `keep-slow` policy: keeps traces with latency > 2 s → negligible during normal load
- `probabilistic-1pct` policy: keeps 1 % of remaining healthy traces → 0.01 × 50 = 0.5 traces/sec

Net result: ~1–2% of traces survive. Of 11 951 spans received, 122 spans were forwarded to Jaeger — a 99% reduction, exactly as designed for cost control.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Chosen test | Reason |
|---|---|---|
| `prompt_length` | **PSI** | Continuous, unbounded distribution. PSI gives a single interpretable score (>0.2 = significant drift). The mean shift from 50→85 characters is large and PSI=3.46 confirms severe drift clearly. |
| `embedding_norm` | **KS (Kolmogorov-Smirnov)** | Embedding norm is bounded near 1.0 with small variance. KS is sensitive to subtle shifts in the CDF without requiring binning, making it ideal for tight, well-behaved distributions. |
| `response_length` | **KL divergence** | Response length has a roughly Gaussian shape. KL captures asymmetric divergence well and is natural when you care more about the reference distribution being "heavier" than the current one. |
| `response_quality` | **PSI** | Quality score is a bounded [0,1] proportion from a Beta distribution. The distribution shifts from high-quality (Beta(8,2)) to low-quality (Beta(2,6)) — PSI=8.85 exposes this extreme shift immediately. |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest metric to expose would be **Day 20 — llama.cpp token throughput** (`monitor-day20-llama-cpp.py`). Unlike cloud-hosted services that expose standard Prometheus endpoints, llama.cpp runs as a local process that must be patched to emit `/metrics`. The monitor script must both start a sidecar exporter and handle the fact that llama.cpp's GPU offload configuration varies per machine — making the metric pipeline fragile unless you control the exact model and hardware. The cross-day dashboard panel correctly shows "No Data (start monitor-day20...)" because that service was not running in this isolated Day 23 environment.

---

## 6. The single change that mattered most

**Adding explicit datasource UIDs to the Grafana provisioning YAML** was the single change that transformed the stack from "technically running but showing nothing" to fully observable.

Every Grafana dashboard panel uses a `datasource` reference like `{"type": "prometheus", "uid": "prometheus"}`. When no `uid:` is set in `provisioning/datasources/datasources.yml`, Grafana auto-generates a random hash (e.g. `PBFA97CFB590B2093`) on first start. All six dashboard panels silently displayed "No data" because the UID in the provisioned datasource didn't match the hardcoded `"prometheus"` UID in the dashboard JSON. Adding two lines — `uid: prometheus` and `uid: loki` — to the YAML and restarting Grafana made every panel instantly light up with live metrics.

This connects directly to the **dashboards-as-code** principle from the lecture: provisioning dashboards via JSON/YAML only works reliably when every reference is deterministic. Random auto-generated IDs break the contract between the datasource definition and the dashboard that consumes it. In a real production environment, this same bug would silently invalidate all your observability tooling during a Grafana upgrade or pod restart — exactly when you need dashboards most. The fix proves that "observable" is not a property of your data pipeline alone; it requires the entire chain from metric emitter → scraper → datasource → dashboard panel to share a consistent naming contract.
