# Throughput in System Design

> Scope: definition, relationship to latency and concurrency, limits and collapse, how to raise it, capacity math, measurement, overload behavior, interview prep.
> All numbers in examples are illustrative. Validate with load tests on your own workload.
> Companion to [scalability.md](scalability.md).

## Contents

1. [What throughput is](#1-what-throughput-is)
2. [Throughput vs latency, bandwidth, capacity](#2-throughput-vs-latency-bandwidth-capacity)
3. [Little's Law: throughput, concurrency, latency](#3-littles-law-throughput-concurrency-latency)
4. [The throughput curve: knee and collapse](#4-the-throughput-curve-knee-and-collapse)
5. [Bottlenecks set the maximum](#5-bottlenecks-set-the-maximum)
6. [How to increase throughput](#6-how-to-increase-throughput)
7. [Batching: throughput vs latency trade-off](#7-batching-throughput-vs-latency-trade-off)
8. [Throughput by tier](#8-throughput-by-tier)
9. [Capacity math](#9-capacity-math)
10. [Measuring throughput](#10-measuring-throughput)
11. [Protecting throughput under overload](#11-protecting-throughput-under-overload)
12. [Common mistakes](#12-common-mistakes)
13. [Interview questions](#13-interview-questions)
14. [Key takeaways](#14-key-takeaways)
15. [References](#references)

---

## 1. What throughput is

Throughput is the amount of work a system **completes** per unit of time.

Units depend on the system:

| System | Typical throughput unit |
|---|---|
| Web / API | requests per second (RPS) |
| Database | transactions or queries per second (TPS / QPS) |
| Message queue / stream | messages per second, MB/s |
| Storage | IOPS, MB/s |
| Network | bits per second, packets per second |
| Batch pipeline | records or jobs per hour |

Three terms to keep straight:

- **Offered load:** work arriving at the system.
- **Throughput:** work completed (any outcome).
- **Goodput:** work completed **successfully** and usefully. Failed requests, timeouts, and duplicate retries inflate throughput but not goodput.

Healthy: offered ≈ throughput ≈ goodput.
Overloaded: offered keeps rising while goodput flattens or **falls**.

**Example.** API receives 1,200 RPS, answers 1,000 RPS, of which 900 succeed within the timeout. Throughput = 1,000 RPS. Goodput = 900 RPS. 200 RPS is queued or dropped.

---

## 2. Throughput vs latency, bandwidth, capacity

| Term | Question it answers | Unit |
|---|---|---|
| Throughput | How much work finishes per second? | req/s, MB/s |
| Latency | How long does one unit of work take? | ms |
| Bandwidth | Maximum data rate of a link (a ceiling, not an achieved rate) | Mbps, Gbps |
| Capacity | Maximum sustainable throughput within targets | req/s at SLO |
| Concurrency | How much work is in flight right now? | count |
| Utilization | Fraction of a resource's capacity in use | % |

Key distinctions:

- High bandwidth does not imply high throughput. Round trips, protocol overhead, and small payloads can leave a fat pipe mostly idle.
- Low latency does not imply high throughput. A single-threaded service with 1 ms latency tops out near 1,000 RPS.
- Capacity is meaningful only with a latency and error target attached: "5,000 RPS at p99 < 200 ms and < 0.1% errors".

---

## 3. Little's Law: throughput, concurrency, latency

```text
L = lambda x W
concurrency = throughput x average time in system
```

Rearranged:

```text
throughput = concurrency / latency
```

This is the single most useful relationship for reasoning about throughput. It holds for any stable system, regardless of arrival pattern.

**Examples**

- 1,000 RPS x 0.1 s = **100 requests in flight**. Size thread pools, connection pools, and queues accordingly.
- A client with 50 connections and 200 ms per call: max throughput = 50 / 0.2 = **250 RPS**, no matter how fast the server is.
- A worker pool of 200 threads, each request holding a thread for 100 ms: ceiling = 200 / 0.1 = **2,000 RPS**.

**Why this matters in incidents.** If a dependency slows from 100 ms to 1 s at the same arrival rate, in-flight work grows 10x (100 to 1,000). Pools exhaust, queues build, and the slowdown spreads upstream. Throughput falls even though nothing "crashed".

**Levers:** raise throughput by raising concurrency (more workers, async I/O, more connections) **or** by cutting latency (less work per request). Concurrency is capped by the narrowest pool or resource in the path.

---

## 4. The throughput curve: knee and collapse

As offered load rises, a typical system moves through three regions:

```text
Throughput
   ^
   |            ______ plateau (saturated)
   |          /        \
   |        /           \___ collapse (overload)
   |      /
   |    /   linear region
   |  /
   +------------------------------> Offered load
          ^         ^
          |         knee: latency starts climbing
          comfortable operating point (below the knee)
```

1. **Linear region:** throughput tracks offered load; latency is flat.
2. **Knee:** a resource nears saturation; queues form; latency climbs sharply.
3. **Plateau:** throughput stops growing; extra load only adds queueing delay.
4. **Collapse:** past some point throughput **drops**. Causes: timeouts followed by retries (wasted work), thread and connection exhaustion, memory pressure and GC thrash, context-switch overhead, lock contention, queues full of requests whose clients already gave up.

### Why latency explodes near full utilization

For a simple single-server queue (M/M/1, idealized), response time ≈ service time / (1 - utilization):

| Utilization | Response time (multiple of service time) |
|---:|---:|
| 50% | 2x |
| 70% | 3.3x |
| 80% | 5x |
| 90% | 10x |
| 95% | 20x |
| 99% | 100x |

Real systems differ, but the shape holds. Conclusion: **never plan to run hot**. A utilization target (commonly 60-75% at peak, depending on burstiness and the cost of failure) is what keeps latency inside the SLO.

---

## 5. Bottlenecks set the maximum

End-to-end throughput is limited by the slowest stage on the path:

```text
max throughput = min(stage_1, stage_2, ..., stage_n)
```

Example (per-request view):

| Stage | Capacity | Notes |
|---|---:|---|
| Load balancer | 20,000 RPS | |
| App tier (8 instances x 1,500) | 12,000 RPS | |
| Cache | 50,000 ops/s | 95% hit ratio |
| Database | 5,000 queries/s | 5 queries per cache-missing request |

Convert everything to the same unit before taking the minimum. If 5% of 12,000 RPS misses cache and each miss costs 5 queries: DB load = 12,000 x 0.05 x 5 = 3,000 queries/s, under the 5,000 limit, so the app tier is the bottleneck at 12,000 RPS. Drop the hit ratio to 80% and DB load becomes 12,000 x 0.2 x 5 = 12,000 queries/s, which overwhelms the database. **A small change in hit ratio moves the bottleneck.**

**Per-request cost view.** If each request needs 5 DB queries and the DB sustains 5,000 queries/s, the ceiling is 5,000 / 5 = **1,000 RPS** regardless of app instances.

Fix the first saturated resource, then re-measure; the next bottleneck is waiting.

---

## 6. How to increase throughput

Two broad strategies: **do less work per request** and **do more work in parallel**.

| Strategy | Examples | Watch out for |
|---|---|---|
| Reduce work per request | Optimize hot paths, fix N+1 queries, add indexes, cache results, trim payloads, compress | Cache invalidation, stale data |
| Cut round trips | Batch calls, connection reuse (keep-alive, HTTP/2), pipelining, fewer chatty service hops | Larger blast radius per batch |
| Increase parallelism | More instances, worker pools, async/non-blocking I/O, partitioned queues | Shared dependencies cap the gain |
| Remove contention | Shorter transactions, finer-grained locks, avoid hot rows or keys, sharded counters | Complexity, consistency |
| Decouple with queues | Offload non-critical work to async workers, absorb bursts | Queue growth, ordering, at-least-once semantics |
| Better data layout | Partition by access pattern, avoid hot partitions, right indexes | Rebalancing, cross-partition queries |
| Right-size resources | Faster disks (IOPS), more memory to avoid swapping, network tuning | Cost |
| Scale out | Add instances behind a load balancer | See [scalability.md](scalability.md); only helps if the bottleneck is the scaled tier |

Order of operations: measure, find the bottleneck, remove unnecessary work first (cheapest), then add parallelism, then add hardware.

---

## 7. Batching: throughput vs latency trade-off

Batching amortizes fixed per-operation costs (network round trip, syscall, disk seek, commit) over many items. It raises throughput and usually raises per-item latency.

```text
No batching:   1 item  / 1 round trip   -> low latency, low throughput
Batching 100:  100 items / 1 round trip -> higher throughput, items wait for the batch to fill
```

Typical knobs:

- **Size trigger:** flush when N items accumulated.
- **Time trigger:** flush after T ms even if not full (caps added latency).
- **Compression:** larger batches compress better, saving bandwidth.

Kafka producer example: `batch.size`, `linger.ms`, and `compression.type` trade producer latency for throughput. On the consumer side, parallelism within a consumer group is bounded by the number of partitions, and `max.poll.records` and processing time per poll shape throughput and rebalance risk.

Database equivalents: multi-row inserts, group commit, bulk loaders.

Rule of thumb: pick the smallest batch that reaches your throughput target while keeping tail latency within the SLO.

---

## 8. Throughput by tier

| Tier | What limits throughput | Typical levers |
|---|---|---|
| App / API | CPU per request, thread or event-loop capacity, GC pauses, downstream waits | Optimize, async I/O, more instances, pool sizing via Little's Law |
| Cache | Network ops/s, single-key hot spots, memory | Replication for reads, key spreading, local near-caches |
| Relational DB | Query cost, locks, connection limits, disk IOPS, WAL/commit rate | Indexes, shorter transactions, pooling, replicas (reads only), partitioning (writes) |
| Queue / stream | Partition count, broker disk and network, consumer processing rate | More partitions, bigger batches, compression, more consumers (up to partition count) |
| Storage | IOPS, MB/s, queue depth, block size, throttling by volume type | Faster volume class, striping, larger sequential I/O |
| Network | Bandwidth, packets per second, connection setup cost, TLS handshakes | Keep-alive, HTTP/2, compression, CDN, regional placement |

Remember that reads and writes scale differently: replicas raise read throughput but do nothing for write throughput.

---

## 9. Capacity math

### 9.1 Request rate

- 100,000 DAU x 20 requests/day = 2,000,000 requests/day
- Average: 2,000,000 / 86,400 ≈ 23 RPS
- Peak at 5x: ≈ 116 RPS (the multiplier is an assumption; use real traffic data when available)

### 9.2 Network throughput

```text
bandwidth = RPS x average payload size
```

1,000 RPS x 50 KB = 50 MB/s = **400 Mbps** (8 bits per byte), before protocol overhead. Check against NIC, load balancer, and egress limits, and consider peak, not average.

### 9.3 Storage write throughput

2,000 writes/s x 1 KB = 2 MB/s ≈ 173 GB/day (2 MB/s x 86,400 s). Multiply by replication factor and index or write amplification for real disk load.

### 9.4 Instances needed

One instance sustains 100 RPS within SLO; expected peak 1,000 RPS.

- Naive: 1,000 / 100 = 10.
- At 70% target utilization: 1,000 / (100 x 0.7) ≈ 14.3, so 15.
- Add N+1 (or survive loss of one of three zones: 15 x 3 / 2 ≈ 23).

### 9.5 Pool sizing with Little's Law

Target 500 RPS to a dependency with 80 ms average latency: in-flight = 500 x 0.08 = 40. A pool of 40 would run at 100% and queue constantly; size with headroom (for example 60-80) and set timeouts so slow calls cannot hold slots indefinitely.

---

## 10. Measuring throughput

### 10.1 What to record

- Request rate (traffic), per endpoint and per status class
- Goodput (successful responses within the latency target)
- Latency percentiles (p50, p95, p99), successes and errors separately
- Saturation of the most constrained resource: CPU, memory, I/O, connection pools, thread pools, queue depth and age, consumer lag
- Cost per request

Averages hide tails. At 1,000 RPS with a 100 ms mean, 1% of requests can still take seconds. Prefer latency histograms over mean values.

### 10.2 Example Prometheus queries

Metric names are examples; adapt to your instrumentation.

```promql
# Throughput (requests per second, 1m window)
sum(rate(http_requests_total[1m]))

# Goodput: successful responses per second
sum(rate(http_requests_total{status=~"2..|3.."}[1m]))

# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# p99 latency from a histogram
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Kafka consumer lag trend (metric name depends on exporter)
sum(kafka_consumergroup_lag) by (consumergroup, topic)
```

Aggregate percentiles from histogram buckets; never average per-instance percentiles.

### 10.3 Load testing for throughput

1. **Ramp test:** increase offered load in steps; record throughput, latency, errors, and saturation per step. Find the knee and the plateau.
2. **Soak test:** hold near expected peak for hours; catches leaks, GC drift, slow degradation.
3. **Spike test:** sudden jump; shows autoscaling lag and overload behavior.
4. **Stress test:** push past the plateau; shows how it fails and whether it recovers when load drops.

Pitfalls:

- **Closed vs open load models.** A closed-loop generator (fixed users, each waits for a response) slows down when the system slows, hiding overload. Real internet traffic is mostly open-loop: arrivals continue regardless. Use an arrival-rate (open) model to find real limits.
- **Coordinated omission.** If the generator pauses while waiting on a slow response, it under-samples exactly the periods that matter, understating latency percentiles. Use tools and settings that schedule requests by intended time.
- **Unrepresentative data and mix.** Cache-warm tests with uniform keys overstate throughput; production has skewed keys, cold caches, and varied request cost.
- **Generator as bottleneck.** Verify the load generator itself is not saturated.

Illustrative ramp result:

| Offered | Throughput | Goodput | p99 | Note |
|---:|---:|---:|---:|---|
| 500 | 500 | 500 | 90 ms | linear |
| 1,000 | 995 | 995 | 120 ms | linear |
| 1,500 | 1,380 | 1,370 | 260 ms | knee |
| 2,000 | 1,450 | 1,300 | 1.8 s | plateau; errors begin |
| 3,000 | 900 | 600 | timeouts | collapse; retries amplifying |

Operating point here: about 1,000-1,200 RPS, safely below the knee.

---

## 11. Protecting throughput under overload

When offered load exceeds capacity, the goal is to keep **goodput** high by refusing excess work early rather than letting everything degrade.

| Mechanism | What it does |
|---|---|
| Rate limiting | Caps per-client or global request rate |
| Load shedding | Drops low-priority or excess requests when saturated, preferably cheaply and early |
| Admission control / concurrency limits | Bounds in-flight work (Little's Law: bounded L bounds latency) |
| Backpressure | Slows or blocks producers when consumers fall behind |
| Timeouts and deadlines | Stops spending resources on requests whose caller has given up; propagate deadlines downstream |
| Retry discipline | Exponential backoff with jitter and retry budgets; unbounded retries multiply load |
| Circuit breakers | Fail fast when a dependency is unhealthy so threads are not tied up |
| Bounded queues | Unbounded queues turn overload into huge latency and memory growth |
| Graceful degradation | Serve cheaper responses (cached, partial) under pressure |

Failure pattern to recognize: dependency slows, in-flight work balloons, pools exhaust, clients time out and retry, load amplifies, throughput collapses. Breaking the loop needs shedding and bounded retries, not just more instances.

---

## 12. Common mistakes

1. **Confusing throughput with bandwidth or capacity.**
2. **Reporting throughput without latency and error context.** "10k RPS" at what p99 and error rate?
3. **Measuring only averages.** Tails and goodput tell the real story.
4. **Testing with closed-loop clients only.** Overload stays hidden.
5. **Ignoring the cost per request.** Different endpoints cost different amounts; RPS alone is a poor capacity unit.
6. **Scaling the wrong tier.** More app instances do nothing if the database is the limit.
7. **Running near 100% utilization.** Latency explodes; no headroom for failure or bursts.
8. **Unbounded queues and retries.** They hide overload, then amplify it.
9. **Over-batching.** Throughput target met, latency SLO missed.
10. **Forgetting downstream limits:** connection caps, API rate limits, partition counts, IOPS throttles.
11. **Testing with warm caches and uniform keys.** Production is colder and more skewed.

---

## 13. Interview questions

**Beginner**
1. What is throughput?
2. Throughput vs latency?
3. Throughput vs bandwidth?
4. What is goodput?
5. What is capacity?

**Intermediate**
6. State Little's Law and use it to size a thread pool.
7. A service has 100 ms latency and 50 concurrent workers. What is its maximum throughput?
8. Why can adding app servers fail to raise throughput?
9. How does batching affect throughput and latency?
10. How do you find the maximum sustainable throughput of a service?
11. Why does latency spike as utilization approaches 100%?
12. Open vs closed workload models in load testing?

**Advanced**
13. Why can throughput drop under overload, and how do you prevent collapse?
14. How do retries affect throughput during partial failure?
15. How do you raise write throughput on a database?
16. How do you scale a Kafka pipeline's throughput, and what limits consumer parallelism?
17. How do you attribute end-to-end throughput limits across a multi-tier path?
18. What is coordinated omission, and how does it distort results?
19. How do you choose a utilization target for a latency-sensitive service?
20. How would you set load-shedding thresholds?

**SRE-flavored**
21. Which SLIs show that throughput is approaching a limit before users are affected?
22. How do you alert on saturation without paging on harmless high utilization?
23. How would you forecast throughput needs for the next quarter?
24. Walk through an incident where a slow dependency caused a throughput collapse.

---

## 14. Key takeaways

1. Throughput is completed work per unit time; goodput is the successful part of it.
2. Throughput, latency, and concurrency are tied by Little's Law: throughput = concurrency / latency.
3. Capacity only means something with a latency and error target attached.
4. End-to-end throughput equals the slowest stage; convert all stages to one unit first.
5. Latency climbs steeply as utilization approaches 100%; plan for headroom.
6. Past the knee, throughput plateaus and can collapse; retries and unbounded queues make it worse.
7. Raise throughput by cutting work per request first, then adding parallelism, then hardware.
8. Batching trades latency for throughput; tune it against the SLO.
9. Measure goodput, tail latency, and saturation, using open-loop load tests.
10. Under overload, shed and limit early to protect goodput.

---

## References

- [Google SRE Book, Ch. 6: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) (traffic and saturation as golden signals, utilization targets, latency as a leading indicator of saturation, tail latency vs mean). Content read and confirmed.
- Further reading from the same book, titles confirmed in its table of contents but not read for this document: [Handling Overload](https://sre.google/sre-book/handling-overload/), [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/).

Concepts not tied to a single source here (Little's Law, M/M/1 queueing response time, open vs closed workload models, coordinated omission) are standard results; consult a queueing theory text and the load-testing tool's documentation before quoting exact figures in a design review.
