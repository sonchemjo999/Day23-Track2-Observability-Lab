# Day 23 Lab Reflection

**Student:** Đào Hồng Sơn  
**Submission date:** 2026-05-11  
**Lab repo URL:** https://github.com/sonchemjo999/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Output/report from `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.1"
  },
  "ram_gb_available": 15.44,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "all_ports_free": false
}
```

The ports were already bound because the Day 23 Compose stack was running when the setup report was captured. Docker, Compose v2, and RAM checks passed.

---

## 2. Track 02 - Dashboards & Alerts

### 6 essential panels (screenshot)

Evidence: `submission/screenshots/track02-1.png`

This screenshot shows the AI Service Overview dashboard with request rate, latency P50/P95/P99, error rate, GPU utilization, token throughput, and in-flight requests.

### Burn-rate panel

Evidence: `submission/screenshots/track02-3.png`

The SLO Burn Rate dashboard renders the multi-window burn-rate panel for 5m, 30m, 1h, and 6h windows. The error budget panel had no data at capture time, but the burn-rate query and alert panels loaded.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` through `make alert` / `scripts/trigger-alert.sh` | `submission/screenshots/track02-5.png` |
| T0+90s | waited for `ServiceDown` to fire | `submission/screenshots/track02-5.png` |
| T1 | restarted `day23-app` | `submission/screenshots/track02-5.png` |
| T1+60s | alert resolved / no active Alertmanager alerts | `submission/screenshots/track02-4.png`, `submission/screenshots/track02-5.png` |

### One thing surprised me about Prometheus / Grafana

The biggest surprise was how sensitive the dashboards were to scrape timing and label choices. A panel can be technically correct but still show `No data` if the service has not emitted enough samples in the selected time window, so I had to treat dashboard validation as a time-series workflow rather than a static UI check.

---

## 3. Track 03 - Tracing & Logs

### One trace screenshot from Jaeger

Evidence: `submission/screenshots/track03-1.png`, `submission/screenshots/track03-2.png`, `submission/screenshots/track03-3.png`

The Jaeger screenshots show `inference-api` traces for `predict` and generated spans such as `generate-tokens`. The span attributes include GenAI-style fields such as `gen_ai.request.model`.

### Log line correlated to trace

Structured JSON log line from `docker compose logs app --tail=50 | Select-String "trace_id"`:

```json
{"model": "llama3-mock", "input_tokens": 10, "output_tokens": 14, "quality": 0.746, "duration_seconds": 0.2464, "trace_id": "1599742a0a9f13f7f7fe4e48d13c2990", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T06:02:19.916980Z"}
```

Trace ID: `1599742a0a9f13f7f7fe4e48d13c2990`

### Tail-sampling math

The collector policy keeps 100% of error traces, 100% of slow traces over 2 seconds, and 1% of healthy traces:

```text
sampled = N * (P(error) * 1.0 + P(slow and not error) * 1.0 + P(healthy) * 0.01)
```

For a typical case with 1% errors, 1% slow non-error traces, and 98% healthy traces:

```text
sampled = N * (0.01 + 0.01 + 0.98 * 0.01)
        = N * 0.0298
        = 2.98% of traces retained
```

For an all-healthy workload, the policy keeps `N * 0.01`, or 1% of traces. The useful part is that the traces most needed during incidents, errors and slow tails, are retained at 100% while normal traffic is aggressively reduced.

---

## 4. Track 04 - Drift Detection

### PSI scores

Contents of `04-drift-detection/reports/drift-summary.json`:

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

HTML report evidence: `submission/screenshots/track04-2.png`

### Which test fits which feature?

For `prompt_length`, I would use PSI in production because it is a binned, easy-to-explain measure for distribution shifts in request shape. Prompt length is operationally important because a shift toward longer prompts changes latency, token cost, and context-window pressure.

For `embedding_norm`, I would use KS for a quick univariate check and MMD if I monitor the full embedding vector instead of only its norm. The norm is a continuous numeric feature, so KS is lightweight, but embedding drift is often geometric and multivariate, where MMD is a better fit.

For `response_length`, I would use PSI or KS. PSI is easier to dashboard and alert on because the score is stable and interpretable by bins, while KS is useful for confirming whether the continuous output-token distribution changed significantly.

For `response_quality`, I would use KS plus PSI. Quality score is continuous and directly tied to model behavior, so KS catches distribution movement; PSI gives an SRE-friendly score for trend dashboards and threshold-based alerts.

---

## 5. Track 05 - Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day metric was the Day 19 vector-store metric from Qdrant. It depends on another local service being reachable through the correct host and port, and the real `/metrics` endpoint can disappear if Day 19 is not running, so the integration script needs a stub path to keep the dashboard rendering.

The cross-day dashboard evidence is `submission/screenshots/track05-1.png`. The fail-soft design is important: panels can show data or `No data`, but the dashboard itself still loads and gives one place to check cloud, pipeline, lakehouse, vector-store, model-serving, and alignment signals.

---

## 6. The single change that mattered most

The single change that mattered most was keeping metric labels low-cardinality and moving request-specific details into logs and traces instead. For Prometheus metrics I kept labels such as `model`, `status`, and `direction`, but avoided labels like `prompt`, `request_id`, or `trace_id`. That made the dashboards stable enough to answer operational questions like request rate, latency, token throughput, and quality score without creating a time-series explosion.

This connects directly to the deck's Prometheus cardinality and three-pillars sections. Metrics should describe aggregate system health, traces should explain one request path, and logs should carry detailed event context. Once I separated those responsibilities, the stack moved from "it emits telemetry" to "it is useful during debugging": Grafana shows where the problem is, Jaeger shows the request path, and the JSON log gives the exact `trace_id` and model context.
