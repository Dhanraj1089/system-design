# Availability in System Design

> **Availability = the fraction of time a system is usable when users need it.**
> High availability (HA) = eliminate single points of failure (SPOFs), add redundancy, detect failure fast, recover fast.

## Contents

1. [Formula and the nines](#1-formula-and-the-nines)
2. [Composite availability math](#2-composite-availability-math)
3. [MTBF, MTTR and why recovery speed wins](#3-mtbf-mttr-and-why-recovery-speed-wins)
4. [Techniques to achieve HA](#4-techniques-to-achieve-ha)
5. [RTO and RPO](#5-rto-and-rpo)
6. [SLI, SLO, SLA and error budgets](#6-sli-slo-sla-and-error-budgets)
7. [Availability vs related concepts](#7-availability-vs-related-concepts)
8. [Interview playbook](#8-interview-playbook)
9. [Cheat sheet](#9-cheat-sheet)

---

## 1. Formula and the nines

```text
Availability = Uptime / (Uptime + Downtime) × 100
```

Worked example (99.9% over one year):

```text
Minutes/year = 365 × 24 × 60 = 525,600
Downtime     = 0.1% × 525,600 = 525.6 min ≈ 8 h 46 min
```

| Availability | Per year   | Per 30 days | Per day  |
| -----------: | ---------: | ----------: | -------: |
| 99%          | 3.65 days  | 7.2 hours   | 14.4 min |
| 99.9%        | 8.76 hours | 43.2 min    | 86.4 sec |
| 99.95%       | 4.38 hours | 21.6 min    | 43.2 sec |
| 99.99%       | 52.6 min   | 4.32 min    | 8.64 sec |
| 99.999%      | 5.26 min   | 25.9 sec    | 0.86 sec |

Memorize: **99.9% ≈ 43 min/month, 99.99% ≈ 4.3 min/month.**

Each extra nine cuts allowed downtime 10x and typically costs far more than 10x:
more redundancy, multi-region, automation, and operational maturity. At 99.99%+,
a human cannot respond in time, so recovery must be automated.

---

## 2. Composite availability math

**Components in series** (every one must work): multiply.

```text
A_total = A1 × A2 × A3
```

```text
LB 99.99% × App 99.9% × DB 99.9% = 99.79%  (~18.4 h/year)
```

Chained dependencies are worse than their weakest link. Each hard dependency lowers your ceiling.

**Components in parallel** (redundant, any one suffices): multiply the failure probabilities.

```text
A_total = 1 - (1 - A)^n
```

```text
2 servers at 99%   → 1 - (0.01)^2 = 99.99%
3 servers at 99%   → 1 - (0.01)^3 = 99.9999%
```

**Caveat:** this assumes failures are independent. Real outages are often
correlated (shared deploy, shared config, same AZ, same bad dependency, same
certificate expiry). Redundancy defends against random hardware failure, not
against a bad release pushed everywhere. This is why canary rollouts and
failure-domain isolation matter as much as replica count.

---

## 3. MTBF, MTTR and why recovery speed wins

```text
Availability = MTBF / (MTBF + MTTR)
```

| Term | Meaning                                          |
| ---- | ------------------------------------------------ |
| MTBF | Mean Time Between Failures (how often it breaks) |
| MTTR | Mean Time To Recovery (how long to fix/restore)  |

Two levers: **fail less often** (raise MTBF) or **recover faster** (cut MTTR).
Cutting MTTR is usually cheaper and more reliable than trying to prevent all failures.

```text
System A: fails 1x/month, 2 h manual recovery   → ~99.7%
System B: fails 1x/month, 30 s auto failover    → ~99.999%
```

MTTR breaks down into: **detect → diagnose → mitigate → verify.** Monitoring and
alerting attack detection; runbooks and observability attack diagnosis; automated
failover, rollback, and traffic shifting attack mitigation.

---

## 4. Techniques to achieve HA

### 4.1 Redundancy

Run multiple instances of every critical component. Remove every SPOF: single
server, single DB, single LB, single AZ, single DNS provider, single cert, single
person who knows how it works.

```text
Users → [Server 1 | Server 2 | Server 3]
```

### 4.2 Load balancing and health checks

```text
                 ┌──→ Server 1
Users → LB ──────┼──→ Server 2
                 └──→ Server 3
```

- Distributes traffic, so servers can fail independently.
- **Health checks** remove unhealthy backends automatically. Check real readiness
  (can it serve?), not just "process is alive."
- The LB itself must be redundant (managed LB, or an active/standby or anycast pair).

### 4.3 Failover

Traffic shifts automatically to a backup when the primary fails.

| Model              | How it works                                     | Trade-off                                        |
| ------------------ | ------------------------------------------------ | ------------------------------------------------ |
| **Active-active**  | All nodes serve traffic                          | Better utilization, near-instant failover; harder state/consistency |
| **Active-passive** | Standby idle until primary fails                 | Simpler; failover delay, idle cost, standby may be untested |

An untested failover path is a hypothesis, not a feature. Exercise it regularly.

### 4.4 Replication

```text
Primary DB ──replicate──→ Replica DB   (promote on failure)
```

- **Synchronous:** no data loss on failover (RPO ≈ 0), but higher write latency
  and writes can stall if the replica is down.
- **Asynchronous:** low latency, but recent writes can be lost on failover (RPO > 0).
- Watch for split-brain (two nodes both believing they are primary). Use quorum/consensus
  or fencing.

### 4.5 Multi-AZ and multi-region

```text
Region
 ├── AZ-1: app + DB node
 └── AZ-2: app + DB node
```

- **Multi-AZ:** protects against data-center-level failure. Baseline for production.
- **Multi-region:** protects against regional outage; costs more and adds data
  consistency and latency complexity. Justify it with the SLO.

### 4.6 Disaster recovery (DR)

For large-scale loss (region outage, major corruption, disaster). Strategies, cheapest to fastest:

| Strategy         | Description                                   | Recovery speed |
| ---------------- | --------------------------------------------- | -------------- |
| Backup & restore | Restore from backups in another region        | Hours+         |
| Pilot light      | Minimal core running, scale up on disaster    | Tens of minutes|
| Warm standby     | Scaled-down full copy running                 | Minutes        |
| Multi-site active| Full capacity in multiple regions             | Seconds–none   |

### 4.7 Resilience patterns in software

- **Timeouts** on every network call (no timeout = unbounded hang).
- **Retries with exponential backoff and jitter** (naive retries cause retry storms).
- **Circuit breakers** to stop hammering a failing dependency.
- **Bulkheads** to isolate resource pools so one failure doesn't consume everything.
- **Load shedding / rate limiting** to protect the system under overload.
- **Graceful degradation:** serve a reduced experience (cached or stale data) instead of failing.
- **Idempotency** so retries and failovers are safe.

### 4.8 Safe change management

Most outages are self-inflicted by changes. Use canary and progressive rollouts,
feature flags, automated rollback, and staggered deploys across failure domains.

### 4.9 Detection

Monitor user-facing symptoms (SLIs), not only host metrics. Alert on SLO burn rate.
Fast detection directly cuts MTTR.

---

## 5. RTO and RPO

| Term    | Question                                        | Drives                               |
| ------- | ----------------------------------------------- | ------------------------------------ |
| **RTO** | How long can we be down? (Recovery Time Objective)  | Failover/DR strategy, automation level |
| **RPO** | How much data can we lose? (Recovery Point Objective) | Replication mode, backup frequency     |

```text
RPO = 0   → synchronous replication
RPO = 5m  → async replication or frequent snapshots
RTO = 30s → automated failover, warm/active standby
RTO = 4h  → backup & restore may be acceptable
```

---

## 6. SLI, SLO, SLA and error budgets

| Term       | Meaning                                                | Example                              |
| ---------- | ------------------------------------------------------ | ------------------------------------ |
| **SLI**    | Measured indicator of service behavior                 | % of requests returning non-5xx under 300 ms |
| **SLO**    | Internal target for an SLI                             | 99.9% over a rolling 30 days         |
| **SLA**    | External contract with consequences (credits/penalties)| 99.5% or customer gets credits       |
| **Error budget** | Allowed unreliability = 100% - SLO               | 0.1% = ~43 min/month at 99.9%        |

Practical points:

- Set the SLA looser than the SLO, so you get warned before you owe money.
- Define "available" per request (success ratio) instead of binary up/down:
  partial outages and brownouts count.
- Spending the error budget is allowed. When it is exhausted, prioritize
  reliability work over feature launches.
- Exclude or include planned maintenance explicitly in the definition.
- Your SLO cannot exceed the SLOs of your hard dependencies (see series math).

---

## 7. Availability vs related concepts

| Concept             | Question it answers                                  |
| ------------------- | ---------------------------------------------------- |
| **Availability**    | What fraction of time is the system usable?          |
| **Reliability**     | How long does it run correctly without failing? (MTBF) |
| **Fault tolerance** | Does it keep working through failure with no visible impact? |
| **Resilience**      | How well does it absorb and recover from disruption? |
| **Scalability**     | Can it handle growing load?                          |
| **Durability**      | Will stored data survive? (distinct from being reachable) |

Key distinctions:

- **Reliability vs availability:** a system that fails often but recovers in
  seconds can be highly available yet not very reliable. Availability includes recovery time.
- **Fault tolerance vs HA:** fault tolerance aims for *zero* visible interruption
  and is a mechanism that supports HA. HA accepts minimal downtime and is the broader goal.
- **Scalability vs availability:** independent. A system can scale to millions of
  users and still go down from one SPOF, or be highly available at tiny scale.
  They interact though: overload is a common cause of outages.
- **Durability vs availability:** data can be safe (durable) while the service is
  unreachable. S3 is a good example of the two being quoted separately.

---

## 8. Interview playbook

**Q: How would you design a highly available web application?**

```text
Users → DNS → LB (redundant, health-checked)
                ↓
   ┌────────────┬────────────┬────────────┐
   │ App (AZ-1) │ App (AZ-2) │ App (AZ-3) │   stateless, autoscaled
   └────────────┴────────────┴────────────┘
                ↓
   DB primary (AZ-1) ⇄ sync replica (AZ-2) → async replica / DR region
                ↓
   Cache, queues: also replicated across AZs
```

Answer framework (say it in this order):

1. **Clarify the target:** what SLO? what RTO/RPO? cost tolerance? Don't over-build.
2. **Remove SPOFs at every layer:** DNS, LB, app, cache, DB, queue, network.
3. **Stateless app tier** behind an LB so any instance is replaceable; autoscale.
4. **Spread across AZs**; consider multi-region only if the SLO demands it.
5. **Data tier:** replication mode chosen by RPO; automated failover; guard against split-brain.
6. **Detect fast:** health checks, SLI-based alerts, burn-rate alerting.
7. **Degrade gracefully:** timeouts, retries with backoff, circuit breakers, load shedding.
8. **Deploy safely:** canary, feature flags, fast rollback.
9. **Prove it:** failover drills, chaos testing, game days, backup restore tests.
10. **Call out trade-offs:** cost, consistency vs availability (CAP), complexity.

Follow-ups worth preparing:

- Why is multiplying nines across dependencies dangerous?
- Sync vs async replication: what do you give up each way?
- How do you avoid split-brain in a failover?
- Why can adding redundancy *reduce* availability? (Complexity, failover bugs,
  correlated failures, config drift.)
- What is the CAP trade-off during a partition, and which would you choose for payments vs a feed?

---

## 9. Cheat sheet

| Concept             | One-liner                                           |
| ------------------- | --------------------------------------------------- |
| Availability        | Fraction of time usable                             |
| Nines               | 99.9% ≈ 8.76 h/yr ≈ 43 min/mo; 99.99% ≈ 52.6 min/yr |
| Series              | Multiply availabilities (worse than weakest)        |
| Parallel            | 1 - (1 - A)^n (assumes independent failures)        |
| MTBF / MTTR         | A = MTBF / (MTBF + MTTR); shrink MTTR first         |
| Redundancy          | No SPOFs                                            |
| Load balancer       | Spread traffic, health-check backends               |
| Failover            | Auto-switch to standby; active-active vs active-passive |
| Replication         | Sync = no loss, slower; async = fast, may lose data |
| Multi-AZ / Region   | AZ for baseline HA; region for DR-level resilience  |
| RTO / RPO           | Max downtime / max data loss                        |
| SLO / error budget  | Target and the allowed unreliability                |
| Fault tolerance     | Works through failure with no visible impact        |
| Graceful degradation| Reduced service beats no service                    |

> **Rule:** High availability comes from eliminating single points of failure,
> adding independent redundancy, detecting failure quickly, and recovering
> automatically, then proving it works by testing the failure paths.