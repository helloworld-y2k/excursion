# Observability crash course — notes

**Payoff:** see *inside* a running system. What logs, metrics, and traces each tell you; why they break when one service becomes many; how OpenTelemetry stitches them; how to know you’re healthy **before users do**.

App: ride-share backend. One booking: match → price → pay → confirm.

---

## The silent failure (why dashboards lie)

One booking fails at confirmation. Rider waits. **CPU fine, memory fine, request count normal.** Green charts + real failure.

That is **monitoring**: you collect what you **decided in advance**. Charts answer questions you already thought to ask.

| | |
|--|--|
| **Known unknowns** | Failures you set a threshold for → monitoring |
| **Unknown unknowns** | Broken booking nobody charted → need **observability** |

**Observability:** understand the system **from the outside**; ask **new** questions **without shipping new code**. If you can pull events for **that booking** and ask *why* **now**, you have it. If you must add a dashboard and wait, you only had monitoring.

---

## Logs

**Log** = timestamped event, one line per thing that happened.

Plain text at thousands of lines/s → regex hell.

**Structured logging** (usually JSON): key/value fields you **query**.

```json
{"ts":"...","booking_id":"b_19","status":"failed","duration_ms":4120,"step":"confirm"}
```

Filter: `status = failed` AND `booking_id = …` → that request.

A **single event** still cannot tell you **how often** or **how slow overall**.

---

## Metrics

**Metric** = number **aggregated over time**. Cheapest signal.

| Type | Behavior | Examples |
|------|----------|----------|
| **Counter** | Only **up** (odometer). Resets on restart; **rate()** expects that. Never force a counter down. | bookings_total |
| **Gauge** | **Up and down** (current level) | in-flight bookings, queue depth, memory |
| **Histogram** | Observation → **buckets**; sum buckets across instances → fleet **P95** | latency |

Histogram P95 is only as honest as **bucket boundaries**.

**Summary** computes percentiles **per machine** — **cannot combine** (you cannot average percentiles). **Fleet latency → histograms.**

**Four golden signals (Google):** latency, traffic, errors, **saturation**.

| Shorthand | View | Fields |
|-----------|------|--------|
| **RED** (Wilkie) | Request | Rate, Errors, Duration |
| **USE** (Gregg) | Resource (CPU, disk…) | Utilization, Saturation, Errors |

Not rivals — **both**. Duration = **distribution**, never a lone average (average hides the tail that pages people).

---

## Traces and spans

Logs = events. Metrics = aggregates. **Trace** = **one request’s path**.

```
trace_id  (same for whole booking)
  └─ root span  POST /book
       ├─ span  matching
       ├─ span  pricing
       ├─ span  payment
       └─ span  confirmation   ← long RED bar (error recorded)
```

**Span:** name, start/end, attributes, status, **span_id**. Tree via **parent span id**. Root = where the request began.

Bar turns red only if code **records the error**. Swallow exception + return default → **slow green bar** (hides for weeks).

---

## Many services: context propagation

Monolith → rides, matching, pricing, payments, notifications. Each hop only sees **its** piece.

**Context propagation:** caller puts trace context on the outgoing request; callee starts a **child** span.

**W3C Trace Context** — HTTP headers everyone speaks.

**`traceparent`:** same **trace-id** every hop + caller **span-id** + **sampled flag** (keep vs drop). All services must make the **same** keep/drop call or you save **half a trace**.

Drop headers on **one hop** → trail **snaps**; downstream spans **orphan**.

OpenTelemetry does this **by default**.

### Queues (not HTTP)

Notifications: message on a queue; payments **don’t wait**. By consume time, producer span has **ended** → cannot be parent.

OTel: **producer span** + **consumer span** joined by a **span link** (same booking, not parent/child). Confirmation-you-never-got still on the trace.

---

## “Three pillars” — the point is correlation

Originally **overlapping circles**, not three towers. Value is **moving between them for the same request**.

```
error log  --trace_id-->  full waterfall (5 services)
latency spike  --exemplar-->  same trace
logger copies active span ids onto every log line  (wire once)
```

**Do not** put `trace_id` on metric labels → **millions of series**, wrecks the bill. Link = **exemplar** (pointer to one real request in that bucket).

**Observability = correlation.** One `trace_id` = one story.

---

## OpenTelemetry (OTel)

**Is:** CNCF vendor-neutral standard to **generate and ship** traces, metrics, logs the same way. Merge of two earlier projects; tools speak it.

**Is not:** a backend. Does **not** store or draw dashboards. Prometheus / Grafana / vendor **store and show**. Instrument once; **repoint** later.

```
app: API (you call) + SDK (does work; no SDK → API is a no-op)
        │  OTLP  (HTTP or gRPC)
        ▼
   Collector  receive → processors (batch, strip PII, fan-out) → exporters
```

**Semantic conventions:** same names (`http.method`, …) everywhere.

**Attributes** = this operation. **Resource attributes** = who emitted. **`service.name` is mandatory.** Skip it → `unknown_service` and a blob on the service map.

**Auto-instrumentation:** HTTP, DB, … no code. **Manual:** business spans auto cannot see. **Use both.**

---

## Sampling (traces)

Cannot store every trace.

| | Head | Tail |
|--|------|------|
| When | **Start**, before anything happens | **After** whole trace finishes |
| Catch errors/slow? | **No** — interesting one may be dropped | **Yes** |
| Cost | Cheap | Buffer **all spans** of a trace on **one collector** (route by **trace_id**); memory + seconds of delay |

Standard trade, not a reason to skip tail sampling.

---

## Cardinality (metrics bill)

Cardinality = **how many distinct values**.

`status` handful, `region` dozens, **`user_id` millions**.

High cardinality is **the point of observability** (slice to **one booking**). **Trap:** put it on a **metric label** → every combo = a **time series**. Same reason: never label metrics with `trace_id`.

Messy unique fields → **traces/logs**. Metric labels **small**. Exemplars link.

---

## Retention (don’t keep everything forever)

| Signal | Typical |
|--------|---------|
| Metrics | Cheap; **~1 year** if you roll up older buckets |
| Traces | Already sampled; **~weeks** |
| Logs | Often **biggest bill** — keep **all errors**; sample success; drop noise **in the collector** |

---

## Healthy: SLI / SLO / SLA

| | |
|--|--|
| **SLI** | What you measure (e.g. % bookings succeeded) |
| **SLO** | Target (e.g. 99.9%) |
| **SLA** | Same target in a **contract**. Test: **break it → money/credits** → SLA |

**Error budget** = `100% − SLO` = allowed failure before users hurt.

30-day month (approx.):

| SLO | Budget |
|-----|--------|
| 99% | **7.2 hours** |
| 99.9% | **43 minutes** |
| 99.99% | **~4 minutes** |

Every extra nine is expensive. Budget **left** → ship features. **Spent** → slow/stop releases until earned back. Policy agreed **ahead**, not a 2 a.m. fight.

---

## Alerting

Page on **symptoms users feel**, not causes.

Kitchen: hot burner = cause; **late plates** = dinner. Burner can run hot all night with plates on time → paging CPU 81% / pool 90% / disk half full → **alert fatigue** → miss the one that matters.

If the right reaction is a **shrug**, it should **never page**. CPU/memory = **debug after you’re awake**.

### Burn-rate alerting

Burn rate = how fast you spend the budget. **1×** → 30-day budget lasts 30 days.

**14.4× for 1 hour** ≈ **2%** of a 30-day budget → worth waking (common recipe, not magic).

Second tier e.g. **6× over 6 hours**; slower **3-day** → ticket.

**Long window AND short window must agree** → real + sustained. Short window recovers → alert **clears minutes** after errors, not nags for an hour.

---

## Service map

Hand-drawn architecture is **wrong the day after**.

Traces already have caller + callee **`service.name`**. Pair of spans → edge. Thickness = traffic; red = failing. Collector can emit **spanmetrics** with no custom code.

---

## eBPF

Tiny **sandboxed** programs in the **kernel**; watch every process **without changing app code**. Kernel **verifier** rejects unsafe; needs **privilege**.

Grafana Beyla, Odigos, etc. → OTel metrics/traces automatically.

**Limit:** sees syscalls (`connect`, write 200 bytes) — **not** “charged the rider $18”. Still add **manual** business spans.

**Hybrid:** eBPF for coverage; handwritten spans for meaning.

---

## Continuous profiling

Metrics: pricing slow. Trace: **which span**. Neither: **which line**.

Profiler samples production (~**few %** overhead). Output: **flame graph** (hot loop / slow distance calc). Grafana Pyroscope, Parca. Jump **slow span → profile**. Sometimes called a “fourth pillar” — more **depth** than settled doctrine.

---

## The user (RUM)

Backend green + SLOs met ≠ happy human. Experience is **browser, device, network**.

**RUM** = real users. **Core Web Vitals** (good at **75th percentile** of real loads):

- **LCP** — main content: **&lt; 2.5s**
- **INP** — tap responsiveness  
- **CLS** — layout jump  

Lab test from a fast DC **lies**. **Synthetic** = safety net; **RUM** = ground truth.

Browser can **start the trace** (page load span) and pass context on API calls → backend spans **children of the click**.

---

## 3 a.m. incident

Page is **not** “CPU high.” It is **booking success dropping, budget burning**.

1. Runbook (checklist, not improvisation).  
2. Confirm burn (same rate query).  
3. **What changed?** Deploy markers on the graph.  
4. Exemplar → trace → failing span → log via **trace_id** → bad deploy to **notifications**.  
5. Rollback → success up, budget stops draining.  

Bigger: **incident commander** vs people fixing vs people updating.

**Blameless review:** timeline + contributing causes; assume people did their best with what they knew. Output = **fixes so it can’t break the same way**, not a name.

---

## Open-source shape

```
apps ──OTLP──► OTel Collector ──push──► Tempo   (traces)
                              ──push──► Loki    (logs)
                              ◄─scrape─ Prometheus (metrics; collector exposes endpoint)

Grafana = one window; jump trace ↔ logs ↔ metrics
Grafana Alloy = packaged OTel collector
```

Self-host at scale = real people (Prometheus needs extra pieces to grow). Small team or huge fleet: **managed** often cheaper. OTel → Grafana Cloud / Datadog / Honeycomb = **repoint collector**, no app rewrite.

---

## Recap

One service → six. Add observability when failure **forces** it: logs → metrics (RED/USE/golden) → traces → **correlation** → OTel → sampling, budgets, alerts, eBPF, profiles, RUM.

The first invisible failed booking is now followable in seconds.

**Keep this:** observability is **not** three separate pillars. It is **correlating them for a single request**.