# Serving 100 million people — notes from the transcript

The diagram looks terrifying because **every box is real infrastructure** and **every arrow is a decision under pressure**. The climb has **six stages**, each starting where the last **broke**.

You should be able to look at **any size** and name **which part is actually struggling**.

**Most expensive mistake:** distributing **before you need to**.

---

## Three words (keep them apart)

| Word | Meaning |
|------|--------|
| **Latency** | How long **one** request takes |
| **Throughput** | How many requests **per second** |
| **Capacity** | How much load before things **degrade** |

Google SRE **four golden signals** sit on this. They **move independently**. Buying more throughput/capacity often **costs latency**. It’s a **dial**, not a switch.

---

## Two directions only

```
VERTICAL     make THE box bigger     zero architecture, zero code
HORIZONTAL   add MORE boxes          only after the box is stateless
```

Vertical first. Boring. Linear $ per vCPU on many families (e.g. AWS M6i on-demand: **double size ≈ double cost ≈ double capacity**). Ceiling: **biggest instance**, and **one box = one failure domain** (one deploy/reboot takes the product down).

**Universal Scalability Law (Gunther):** add concurrency → throughput **rises, flattens, then falls**. Extra nodes can **make you slower** (shown on multi-node Hadoop, not just one CPU).

**Rule: scale the bottleneck, not the system.**

---

## Four axes (every scaling problem is one of these)

1. **Compute** — handling the request itself  
2. **Reads** — answering questions about data  
3. **Writes** — changing data  
4. **Async** — work **after** you’ve already responded  

Split **by what it does** first (upload / feed / notify) — **reversible**.  
Split **by which data** (key ranges / shards) later — **nearly permanent**.

```
compute ──► bigger box → stateless → many boxes + LB
reads    ──► index/query shape → cache → replicas → CDN
writes   ──► pool + batch → split by capability → shard
async    ──► queue/topic → idempotent workers → scale on depth
then always: fail safely (timeouts, breakers, shed, degrade)
```

---

## Watching the system

**RED** (Wilkie, services): **R**ate, **E**rrors, **D**uration.  
**USE** (Gregg, hardware): **U**tilization, **S**aturation, **E**rrors.  
You need **both**.

**Averages lie** (*The Tail at Scale*). Same mean, different user pain. Track **P90** (typical) and **P99** (the complaint). Track **cost per 1k requests** from day one.

Dashboards say *something* is slow. **Traces** say *which hop*: tag at the edge, one bar per service. Long bar = answer; short bars prove they’re fine.

Load-test until something gives **before** production is 100× yesterday.

---

# The climb

## ~100 users — monolith (correct, not a phase to skip)

One box, one Postgres, nginx in front. Fast, debuggable, one head. **Do not distribute yet.** Build dashboards in the calm.

## ~10k — compute red → bigger box

CPU-bound → **resize + restart**, no code. Hardware isn’t always the bug:

- Datadog: **use the index you already had** → 22,000 ms → 200 ms (~110×).  
- Sentry: **N+1 → one join** → 7s → &lt;2s; typical 3s → 275 ms.  

Missing index **beats** “2× instance for 2× money.”

Vertical dies: no bigger SKU, and single failure domain.

## ~100k — no bigger box → horizontal **after** emptying state

Must leave the app:

| In-process (unsafe to clone) | Move to |
|------------------------------|---------|
| Sessions in RAM | Redis or **signed tokens** |
| Files on local disk | Object storage |
| In-process cache | Shared cache |
| Cron inside the app | Own process |

**Sticky sessions:** 12-factor **violation**. Uneven load, scary deploys. Works well enough to be dangerous.

### Traffic finds a box

```
DNS     nearest region
L4      fast: which connection (no HTTP peek)
L7      path/headers, routing, throttling
```

TLS termination ≠ L4 vs L7. **Content-based routing is L7.** AWS NLB can terminate TLS at L4.

**Health checks:**  
- **Liveness** = process alive (**shallow**).  
- **Deep readiness** hitting **shared DB/queue** → all instances fail together → LB dumps **entire fleet** = self-inflicted outage.  
A health check **can be** the outage.

**Autoscaling:** CPU lies when threads **wait** on a slow dependency (AWS documents this). Scale on **backlog per instance**. New boxes need **boot + warm**. Pokémon Go: 50× target in ~15 min — **automation alone couldn’t absorb it**. Autoscaling saves **money**, not a surprise spike.

### Pagination (same DB, 15 servers)

**OFFSET** walks and **throws away** skipped rows → deep pages expensive.  
**Cursor / keyset:** “after last row I saw” → every page same cost.

## Reads: replicas, then cache, then CDN

**Replicas:** primary writes, replicas read. **Async lag** (GitHub: &lt;600 ms 95% — good, **not zero**).

**Read-your-own-writes** (Vogels): upload → profile on replica → **own clip missing**. Fix: route **that user** to **primary** for a short window. You traded **consistency**.

**Cache-aside:** cache → miss → DB → fill. Facebook memcached: **92%–99.9%** hits → DB sees ~8% even at the low end. **Single-flight** refill of a cold key → ~**10×** lower peak DB.

Highest-leverage lever in the talk.

**Invalidation:**

| Strategy | Trade |
|----------|--------|
| **TTL** | Expires even if unchanged |
| **Write-through** | Never stale; every write hits two systems |
| **Explicit bust** | Precise; **one forgotten path** → stale **forever** |

**Materialized view:** cache the **answer** (join + sort + page), not a row. Work moves **write-time**. Two copies; freshness = last rebuild.

**Fan-out on write** vs **read:** feeds read more than write → pay once at write. **Celebrity (10M followers)** = 10M inserts = **hot partition**. Real systems: fan-out write for normals, **merge celebs at read**.

**Thundering herd / stampede:** popular key expires → all miss together. Facebook: 17k QPS → **one filler, others wait** → 1.3k QPS (**92% cut**). **Jitter** TTLs.

**Hot key:** one viral clip saturates **one shard**; others idle. Twitter 2018 (~1h partial outage): hashing piled keys; **adding a shard wouldn’t help**. Fix: **local cache** in front of shared cache.

**Load-bearing cache:** if cache vanishes, do you **degrade** or **die**? If die, cache is an **undocumented dependency**.

**CDN:** fiber ~⅔ c. Sydney–Virginia ≈ **157 ms floor** before routers; Azure ~**200 ms** US East–Australia, contracts ~210 ms. **Distance, not CPU.** Same coast can be single-digit ms.

## ~5M — writes: connections, indexes, split, then shard

15 apps × 100 conns → thousands. Postgres default **max_connections = 100**. AWS: idle conn ~**1.5 MB** + CPU; 2,000 idle: small instance **1% → 8% CPU** with **zero queries**.

**PgBouncer / RDS Proxy:** many app conns → few real ones. DB conn count **flat** as fleet grows.

**Batch** writes. **Every index is a write tax** (Winand: first index can crush insert rate by ~**100×**). Read optimization **is** the write problem.

**Functional partition first** (reversible): uploads DB, feed DB, notify DB. Cost: **no cross-table join**, **no single transaction**.

Then **shard by key** (bookshelf A–F, G–L…). DynamoDB partition caps (**3k reads / 1k writes per partition**) regardless of table size. Popular creator = **one hot shelf**. Adaptive capacity (on by default since 2019) exists because traffic **drifts**.

**Reshard:** naive hash → **almost every key moves**. **Consistent hashing** (Dynamo paper): join/leave hits **neighbors only**. Cassandra: **virtual nodes**. Cost: count/scan **every shard**; distributed tx hard.

**Right store:** known access patterns → table; full text → search engine; metrics → time series; follows → graph.

**CQRS:** different **write model vs read model**, not just a replica. Fowler: default use **causes more problems**. Read lag + sync is **ongoing** work (Microsoft).

## ~20M — async: what the user needs *now*

Upload: user needs **file durable**. Transcode, thumb, moderation, notify, feed fan-out, search index → **after** response.

```
before: 45s, 6 jobs on the request path
after:  400ms  [store file]     rest → queue "later"
```

**Queue:** each item to **exactly one** worker (transcode once).  
**Topic:** **every** subscriber gets a copy (notify + feed + search).

Cleanest autoscale signal: **queue depth**, not CPU.

**Idempotency:** SQS standard **may deliver twice**. FIFO/Kafka “exactly once” = **broker won’t invent dupes**. Worker that **does the side effect then crashes before ack** still **redelivers**. Honest: **at-least-once + idempotent app = effectively once**. Exactly-once is **marketing**.

**Ordering:** parallelism **gave it up**. Order only **inside a partition**. SQS FIFO ~**3k msg/s batched** vs unordered firehose. Price is **per order group**.

**Silent loss:** visibility timeout **too short** = duplicate (OK-ish). **Ack then crash** = **gone**. DLQ isolates poison; **unmonitored DLQ fills forever** (alarm is optional in AWS docs).

**Dual write:** cannot atomically **DB + broker**. **Outbox:** event row in **same transaction** as business write; publisher (or **Debezium** tailing WAL) later. Commit and publish **together or not at all**.

**Backpressure:** SRE book — queue ≤ **half thread pool**; reject early. Unbounded queue **delays** overload while latency is **wait in line**. Reactive manifesto: don’t fail badly or drop silently — **signal upstream**. AWS: **backlog per instance**.

## ~100M — why launch day still dies

**Retry storm:** slow service → retries **triple** load. Timeout too low → **complete outage**. Retries are **selfish**. AWS Oct 2025 account: **congestive collapse**, retry queue outgrew processing, hours, dozens of services.

**Jitter** on backoff (AWS sim: **&gt;50% fewer** total calls vs plain backoff). Google: **3 attempts**, retries **&lt;10% of traffic**. **Never blindly retry non-idempotent writes** (timeout ≠ “it didn’t happen”).

**Cascading failure:** C slow → B threads pile (no timeout) → A piles. Google: 5% slow requests need 5,000 threads vs 1,000 → front **fails 80% of everything**, including work that **never touched C**.

**Nygard *Release It!*:** timeouts **everywhere**; **bulkheads** per dependency (Netflix Hystrix). **No timeout = resource leak.**

**Deadline propagation:** three generous hop timeouts **sum** past the user’s patience. Give the **request a budget**; subtract and pass remainder; last hop **fails immediately** rather than start work it can’t finish.

**Circuit breaker:** closed (count failures) → **open** (fail fast) → **half-open** (trial) → close or reopen. Hystrix / Resilience4j.

**Load shedding:** accept everything past capacity → **goodput → 0**. Shed 10%, keep 90% healthy. AWS: shed 60% and **median looks amazing** because fails are **fast** — **goodput** is the real number. Google: **four priority** levels, drop least important first.

**Rate limit:** **token bucket** (Stripe: spare tokens = real bursts). **Sliding window** smoother, more state; Cloudflare: approximation wrong on **0.0003%** of 400M requests. Bucket simpler for bursts.

**Graceful degradation (decide before load does):** uploads **never** break; feed **stale OK**; recommendations **off**.

---

# Global

**Multi-AZ** ≠ **multi-region**. Zones = buildings, own power/net; **region event takes all AZs**. Second region is **far** → **~200 ms** floor returns. Survival decision, not latency win.

**DR ladder (AWS WA):** backup/restore → pilot light → warm standby → **active-active** (most operationally complex). **Compute copies easy; state is the whole problem.** Two regions writing one record → **conflict**.

**CAP (Brewer; Gilbert & Lynch proof):** during **partition**, reject writes (consistency) **or** accept both and reconcile (availability). DynamoDB global tables: **last-write-wins by timestamp** — older write **quietly gone**. Fine for some data; **not** for money/identity.

**Cells:** identical slices of customers. 16 cells, one dies → **~94% never notice**. **Blast radius**, not speed.

---

# Cost

Cost/1k requests from act one. Flexera 2026: **~29% wasted** cloud spend (up). McKinsey ~4k servers: typical box delivers **5–15%** of capability.

5× cost per user is **spending**, not scaling.

| Shape | Lever |
|-------|--------|
| Steady baseline | Reserved / Savings Plans up to **72%** (1–3 yr) |
| Stoppable async | Spot up to **90%** (can vanish) |
| Spikes | On-demand |
| Ops time | Managed services |

**Data tiering:** clip hot days 1–N, then cold. Lifecycle is a **guess** — check real access. Colder = slower restore.

**Egress:** inbound often free; **out bills**. Same AZ free-ish; **cross-AZ**; **internet worst**. Chatty pair in **two AZs** surprises people. Price the **arrows**.

**Failover:** DNS **TTL** bounds how fast you can steer; health checks decide “dead.” **Untested runbook = untested.** Rehearse.

---

# 2 a.m. decision tree

1. Which of **four axes** is saturated?  
2. **Compute:** vertical → drain state → horizontal → LB (right health/autoscale signals).  
3. **Reads:** **reshape query/index first** → cache → replicas → CDN.  
4. **Writes:** pool + batch → functional split → shard.  
5. **Async:** queue → **idempotent** workers → scale on **depth**.  
6. Always: timeouts, breakers, shedding, **what may degrade**.

```
100 → 10k → 100k → 1M → 5M → 20M → 100M → global
each rung = different bottleneck = different fix
```

Name every box: compute / data / writes / async / multi-region. Say **what problem it solves** and **what it costs**. Don’t scale the thing that isn’t the constraint.