# Fallacies of Distributed Computing

> **Topic:** System Design Fundamentals (Distributed Systems)
> **Level:** Beginner to Intermediate
> **Focus:** Network failure, latency, bandwidth, security, topology, ownership, cost, heterogeneity, SRE practice
> **Companion guides:** [Reliability](reliability.md) · [Latency](latency.md) · [Scalability](scalability.md) · [CAP Theorem](cap-theorem.md)
> **Last updated:** 20 October 2026

## Table of Contents

1. [What Are the Fallacies of Distributed Computing?](#1-what-are-the-fallacies-of-distributed-computing)
2. [History and Origin](#2-history-and-origin)
3. [Why Do These Fallacies Matter?](#3-why-do-these-fallacies-matter)
4. [The Eight Fallacies Explained](#4-the-eight-fallacies-explained)
5. [A Resilient Remote Call (Worked Example)](#5-a-resilient-remote-call-worked-example)
6. [Summary of the Eight Fallacies](#6-summary-of-the-eight-fallacies)
7. [Beyond the Eight: Commonly Proposed Additions](#7-beyond-the-eight-commonly-proposed-additions)
8. [How the Fallacies Apply to Microservices](#8-how-the-fallacies-apply-to-microservices)
9. [How the Fallacies Apply to AWS](#9-how-the-fallacies-apply-to-aws)
10. [How the Fallacies Apply to Kubernetes and SRE](#10-how-the-fallacies-apply-to-kubernetes-and-sre)
11. [Testing for the Fallacies](#11-testing-for-the-fallacies)
12. [Remote Call Design Checklist](#12-remote-call-design-checklist)
13. [Connection to Other System Design Concepts](#13-connection-to-other-system-design-concepts)
14. [Common Mistakes to Avoid](#14-common-mistakes-to-avoid)
15. [Interview Questions and Answers](#15-interview-questions-and-answers)
16. [Cheat Sheet](#16-cheat-sheet)
17. [Key Takeaways](#17-key-takeaways)
18. [References](#18-references)

---

## 1. What Are the Fallacies of Distributed Computing?

The Fallacies of Distributed Computing are a set of assumptions that developers often make when building distributed systems, but which turn out to be false in real-world environments.

A distributed system consists of multiple computers that communicate over a network to provide a service. Unlike a single computer, its components can fail independently, network communication can be delayed or lost, and different machines may have different views of the system's state.

The classic list contains eight fallacies:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

**Core principle:** Design distributed systems assuming that networks, machines, configurations, and dependencies can fail or behave unexpectedly.

## 2. History and Origin

The list evolved over several years at Sun Microsystems:

- **Bill Joy and Tom Lyon** had already identified the first four as "The Fallacies of Networked Computing."
- **L. Peter Deutsch** extended the list to seven around 1994.
- **James Gosling** added the eighth (the network is homogeneous) in 1997.

The list is commonly attributed to Deutsch, though it was a collaborative effort. Arnon Rotem-Gal-Oz later wrote a widely read essay, *Fallacies of Distributed Computing Explained*, that expanded on each item.

The list has aged remarkably well. The technology has changed enormously (cloud, containers, service meshes), but each fallacy still describes a way real systems fail today.

## 3. Why Do These Fallacies Matter?

Imagine an e-commerce application built from separate services:

- User Service
- Product Catalog Service
- Order Service
- Payment Service
- Notification Service

The services communicate through HTTP APIs and message queues. A developer might assume that when the Order Service calls the Payment Service, the request will always succeed, quickly and securely.

In production, however:

- The Payment Service may be unavailable.
- The network may delay a response.
- A request may succeed even though the response is lost.
- A retry may accidentally initiate a duplicate payment.
- DNS or routing changes may temporarily prevent communication.
- A cloud network configuration change may block traffic.

Each of these problems causes failures when the system relies on unrealistic assumptions. Understanding the fallacies helps engineers **design for failure instead of assuming ideal conditions**.

## 4. The Eight Fallacies Explained

### Fallacy 1: The Network Is Reliable

**False assumption:** Network communication always succeeds.

**Reality:** Requests can time out, packets can be dropped, connections can reset, and services can become unreachable. Even cloud networks experience connectivity problems from infrastructure failures, misconfiguration, routing issues, or overloaded components. (See Bailis and Kingsbury, *The Network Is Reliable*, ACM Queue, for a survey of real network failures.)

#### Example: Payment processing

A customer submits an order worth ₹2,000.

1. The Order Service sends a payment request.
2. The Payment Service processes the payment.
3. The response is lost because of a network problem.
4. The Order Service times out.

The Order Service does not know whether the payment succeeded. Blindly retrying could charge the customer twice if the provider processes both requests independently.

#### Why it cannot be fully solved: the Two Generals Problem

Two parties communicating over an unreliable channel can never be **certain** they agree. If A sends a message and B acknowledges it, B does not know whether the acknowledgment arrived; if A acknowledges the acknowledgment, the same problem repeats. No finite number of messages removes the uncertainty.

In practice this means you cannot get certainty from the network. You get it from **design**: idempotent operations, durable state, and reconciliation.

#### How to design for it

- Set connection and request timeouts.
- Use bounded retries with exponential backoff and jitter.
- Make operations idempotent where possible.
- Use circuit breakers where appropriate.
- Monitor network errors and dependency health.
- Design reconciliation workflows for uncertain outcomes.

See [A Resilient Remote Call](#5-a-resilient-remote-call-worked-example) for code.

**SRE lesson:** A timeout does not necessarily mean the operation failed. It means the caller did not receive a response within the expected time. (See also [CAP, Section 2.4](cap-theorem.md#24-a-timeout-is-a-partition-decision).)

### Fallacy 2: Latency Is Zero

**False assumption:** Communication between services takes no time.

**Reality:** Every network request introduces latency. The delay depends on physical distance, routing, congestion, processing, and the number of network round trips.

A call within the same data center is generally faster than one between distant regions, but neither is instantaneous. Distance sets a hard floor: light in fiber covers about 1,000 km in 5 ms one way (see [Latency, Section 5](latency.md#5-latency-numbers-worth-knowing)).

#### Example: A slow checkout flow

Checkout performs five sequential service calls:

| Service call | Example latency |
|---|---:|
| Validate user | 20 ms |
| Retrieve product | 30 ms |
| Check inventory | 40 ms |
| Authorize payment | 120 ms |
| Create order | 35 ms |
| **Total** | **245 ms** |

Sequentially, latencies add: `20 + 30 + 40 + 120 + 35 = 245 ms`, excluding extra processing, retries, and network overhead.

If the first three calls are independent and can run in parallel, the total becomes `max(20, 30, 40) + 120 + 35 = 195 ms`. Parallelism helps, but only on the critical path, and it raises exposure to the slowest branch (tail latency; see [Latency, Section 9](latency.md#9-tail-latency-in-distributed-systems)).

#### Latency vs bandwidth

They are different resources. Latency is how long a message takes to arrive; bandwidth is how much data moves per second. A high-bandwidth link does not make a request-response exchange faster: a tiny request over a 100 ms round trip still costs 100 ms, regardless of link capacity. Many small sequential calls are latency-bound; large transfers are bandwidth-bound.

#### How to design for it

- Measure p50, p95, and p99 latency.
- Avoid unnecessary network round trips.
- Parallelize independent operations when safe.
- Cache frequently requested data where appropriate.
- Place workloads close to their dependencies when practical.
- Set end-to-end latency budgets and propagate deadlines.
- Avoid synchronous dependency chains that are unnecessarily long.

**SRE lesson:** Optimize the entire request path, not just the response time of an individual service.

### Fallacy 3: Bandwidth Is Infinite

**False assumption:** A network can transfer unlimited amounts of data.

**Reality:** Networks have finite capacity. Large payloads, excessive logging, replication traffic, and high request volumes can consume available bandwidth.

#### Example: Large API responses

A product API returns 2 MB of JSON per request. At 500 requests per second:

`2 MB × 500 = 1,000 MB/s ≈ 1 GB/s`

That is about **8 Gbps** of payload (`1,000 MB/s × 8 bits`), before protocol overhead and other traffic. On a 10 Gbps link, this one API would consume roughly 80% of capacity.

This can create substantial load, especially when traffic crosses regions or passes through shared infrastructure.

#### Hidden bandwidth consumers

- Log and metric shipping (often forgotten until the bill or the saturation arrives).
- Replication and backups.
- Retries and duplicated requests.
- Service mesh sidecars and telemetry.
- Large container image pulls during scale-out or restarts.

#### How to design for it

- Reduce unnecessary response fields.
- Use pagination for large result sets.
- Compress payloads when beneficial (it trades CPU for bandwidth).
- Avoid transferring large files through application APIs unnecessarily (use pre-signed URLs and object storage).
- Cache static assets and repeated responses (CDNs).
- Monitor network throughput, saturation, and packet drops.
- Use streaming or asynchronous processing for suitable workloads.

**SRE lesson:** Optimize the amount of data transferred, not just the number of requests.

### Fallacy 4: The Network Is Secure

**False assumption:** Anything reachable over the network can be trusted.

**Reality:** Networks may be exposed to unauthorized access, compromised credentials, malicious traffic, misconfiguration, and lateral movement after a breach. Private networks and cloud security groups reduce exposure, but they do not establish trust.

#### Example: Internal API access

An attacker compromises one application service and calls the internal payment API. If the payment API trusts every request from the internal network, the attacker can use that trust to reach sensitive operations.

#### How to design for it

- Use TLS to protect data in transit (mutual TLS between services where appropriate).
- Authenticate services and users.
- Apply authorization checks to individual operations.
- Follow least-privilege access.
- Rotate credentials and manage secrets securely.
- Segment networks and restrict unnecessary traffic (for example, Kubernetes NetworkPolicies).
- Audit sensitive actions.
- Monitor unusual access patterns.

A **zero-trust** approach (see NIST SP 800-207) assumes network location alone is not sufficient proof of identity or authorization.

**SRE lesson:** Treat internal services as security boundaries, not as automatically trusted components. Also remember that security controls can themselves cause outages (expired certificates, misconfigured policies), so monitor and rotate them deliberately.

### Fallacy 5: Topology Doesn't Change

**False assumption:** The network structure and the location of services remain constant.

**Reality:** Infrastructure changes continually. Machines are replaced, IP addresses change, containers restart, workloads move, and routes or DNS records are updated. In cloud-native systems, instances are ephemeral.

#### Example: Kubernetes

A service runs several pods. One crashes and Kubernetes replaces it with a pod that has a different IP address. If another application hardcoded the old IP, its requests fail.

```yaml
# Fragile: depends on a specific pod IP
PAYMENTS_URL: http://10.244.3.17:8080

# Better: a stable service identity resolved by DNS
PAYMENTS_URL: http://payments.shop.svc.cluster.local:8080
```

#### Gotchas even with service discovery

- **DNS caching:** clients and runtimes may cache DNS results longer than expected, so they keep using stale addresses. Respect TTLs and re-resolve on connection errors.
- **Long-lived connections:** HTTP/2 and gRPC reuse one connection for many requests. Behind a standard ClusterIP Service, load balancing happens per **connection**, so one pod can receive most of the traffic and new pods receive none. Use a headless Service with client-side balancing, or an L7 proxy or service mesh that balances per request.
- **Connection draining:** during rollouts, pods terminate while requests are in flight; use readiness probes and graceful shutdown (see [Reliability, Section 23.6](reliability.md#236-graceful-shutdown)).

#### How to design for it

- Use Kubernetes Services instead of hardcoded pod IPs.
- Use service discovery and DNS appropriately.
- Use load balancers for distributing requests.
- Automate configuration and infrastructure management.
- Handle connection resets and DNS changes.
- Avoid depending on fixed instance identities unless required.
- Monitor routing, discovery, and endpoint health.

**SRE lesson:** Design around stable service identities and discovery mechanisms rather than assuming individual instances will remain unchanged.

### Fallacy 6: There Is One Administrator

**False assumption:** A single person or team controls every part of the environment and can coordinate every change.

**Reality:** Large distributed systems span multiple teams, cloud accounts, organizations, vendors, and regions. Each has different permissions, change processes, and priorities. Third-party dependencies (payment providers, identity providers, CDNs) are administered by people you will never meet, on schedules you do not control.

#### Example: Production outage

An application suddenly cannot connect to its database. The application team sees no recent code changes. The database team sees no database failure. A networking team recently modified a firewall rule. No single team has visibility into the whole request path.

#### How to design for it

- Define clear service ownership.
- Maintain documented escalation paths.
- Use centralized observability where appropriate.
- Establish change-management practices (and a searchable change log so responders can ask "what changed?").
- Automate configuration validation.
- Use infrastructure as code.
- Coordinate changes across dependent teams.
- Maintain clear incident command and communication procedures.
- Track third-party dependency SLOs (a hard dependency at 99.9% caps your own availability; see [Reliability, Section 15.1](reliability.md#151-series-system)).

**SRE lesson:** Observability and incident response must work across organizational boundaries. See also [Conway's Law](conways-law.md) and [Reliability, Section 28](reliability.md#28-incident-management-and-root-cause-analysis).

### Fallacy 7: Transport Cost Is Zero

**False assumption:** Sending data between components has no meaningful cost beyond the code that performs the operation.

**Reality:** Network communication consumes time, compute, infrastructure capacity, and often money.

Transport cost includes more than bandwidth charges: serialization and deserialization, encryption, connection management, load balancing, retries, and the engineering effort to operate it all.

#### Example: Chatty microservices

An Order Service makes 20 sequential requests to assemble one checkout response. Even if each call takes only 15 ms, network latency alone contributes:

`20 × 15 ms = 300 ms`

This excludes application processing and other overhead.

#### Money matters too

Cloud providers commonly charge for data moving between availability zones, between regions, out to the internet, and through managed gateways. Chatty cross-zone or cross-region designs can produce surprisingly large bills even when performance is acceptable. Check your provider's current pricing, and attribute transfer cost to the services that generate it.

#### How to design for it

- Reduce unnecessary service-to-service calls.
- Avoid overly chatty APIs (prefer coarser-grained calls or batch endpoints).
- Use batching when appropriate.
- Cache suitable data.
- Consider asynchronous communication for non-blocking work.
- Keep chatty components in the same zone or region where reliability allows.
- Use efficient serialization on hot paths (for example, Protocol Buffers instead of verbose JSON).
- Monitor request volume and data transfer costs.

**SRE lesson:** A design with many small services can still perform poorly (and cost a lot) if its communication patterns are inefficient.

### Fallacy 8: The Network Is Homogeneous

**False assumption:** Every machine, network, operating system, protocol, and software version behaves identically.

**Reality:** Distributed systems commonly combine different hardware, operating systems, runtimes, regions, network configurations (MTU, proxies, firewalls), and service versions. These differences create compatibility and interoperability problems.

#### Example: API version mismatch

The Order Service expects a response field named `customer_id`. The Customer Service is updated and renames the field or changes its type. The Order Service has not yet been updated. Requests fail even though both services are individually healthy.

#### How to design for it

- Use documented API contracts and contract tests.
- Introduce backward-compatible changes (add fields; do not rename or repurpose them).
- Validate schemas and data formats.
- Be a **tolerant reader**: ignore unknown fields rather than failing on them.
- Test interoperability between supported versions.
- Use staged rollouts (see [Reliability, Section 25](reliability.md#25-safe-change-management-and-deployments)).
- Monitor version-specific errors.
- Define compatibility and deprecation policies.

**SRE lesson:** Deployments are not simultaneous. During a rollout (and rollback), old and new versions coexist, so every change must be safe in mixed-version operation.

## 5. A Resilient Remote Call (Worked Example)

The fallacies converge in one pattern: calling another service safely. This sketch combines a timeout, bounded retries with exponential backoff and jitter, a total deadline, an idempotency key, and honest handling of an unknown outcome.

```python
import random
import time

import requests


def charge(amount_inr, order_id, *, attempts=4, base=0.2, cap=5.0, deadline_s=10.0):
    key = f"order-{order_id}"  # same key on every retry of this logical operation
    start = time.monotonic()

    for attempt in range(attempts):
        remaining = deadline_s - (time.monotonic() - start)
        if remaining <= 0:
            break
        try:
            resp = requests.post(
                "https://payments.internal/v1/charges",
                json={"amount": amount_inr, "order_id": order_id},
                headers={"Idempotency-Key": key},
                timeout=(1.0, min(3.0, remaining)),  # (connect, read)
            )
            if resp.status_code < 500 and resp.status_code != 429:
                return resp  # success, or a client error that retrying cannot fix
        except (requests.ConnectionError, requests.Timeout):
            pass  # outcome unknown, so retry safely using the idempotency key

        backoff = random.uniform(0, min(cap, base * 2**attempt))  # "full jitter"
        time.sleep(min(backoff, max(0.0, deadline_s - (time.monotonic() - start))))

    # Do NOT assume the charge failed: it may have succeeded with a lost response.
    raise RuntimeError("payment outcome unknown: mark order PAYMENT_UNKNOWN and reconcile")
```

What each part addresses:

| Element | Fallacy addressed |
|---|---|
| Connect and read timeouts | Reliable network, zero latency |
| Total deadline across retries | Zero latency (bounded waiting) |
| Exponential backoff with jitter | Reliable network (avoids synchronized retry storms) |
| Bounded attempts | Reliable network (retries amplify overload) |
| Idempotency key | Reliable network (makes retry safe) |
| Not retrying `4xx` (except `429`) | Reliable network (do not hammer permanent errors) |
| `PAYMENT_UNKNOWN` and reconciliation | Reliable network (ambiguous outcomes) |

This is a teaching sketch. In production, also use a circuit breaker, a retry budget (cap retries to a small share of traffic), metrics on retries and unknown outcomes, and a reconciliation job that resolves unknown orders by querying the provider with the idempotency key. See [Reliability, Section 17](reliability.md#17-timeouts-retries-circuit-breakers-and-backpressure) and [Reliability, Section 30.4](reliability.md#304-model-order-state-explicitly).

## 6. Summary of the Eight Fallacies

| Fallacy | Real-world problem | Common mitigation |
|---|---|---|
| Network is reliable | Requests fail or responses are lost | Timeouts, retries, idempotency, reconciliation |
| Latency is zero | Slow end-to-end requests | Latency budgets, caching, fewer round trips |
| Bandwidth is infinite | Network saturation | Payload optimization, pagination, compression |
| Network is secure | Unauthorized access or interception | Authentication, authorization, TLS, segmentation |
| Topology doesn't change | Stale addresses and broken connections | Service discovery, load balancing, re-resolution |
| One administrator | Cross-team coordination failures | Ownership, observability, incident processes |
| Transport cost is zero | Excessive latency and infrastructure cost | Batching, fewer calls, efficient communication |
| Network is homogeneous | Compatibility failures | Contracts, versioning, compatibility testing |

## 7. Beyond the Eight: Commonly Proposed Additions

The original list covers the network. Practitioners often add further false assumptions. These are **unofficial** community additions, but each causes real incidents.

| Unofficial fallacy | Reality | Mitigation |
|---|---|---|
| **Clocks are synchronized** | Clocks drift and jump; timestamps from different machines cannot be trusted for ordering | Use logical clocks or version vectors for ordering; avoid last-write-wins on wall-clock time; monitor clock skew |
| **Failures are independent** | Replicas share power, zones, software releases, configuration, and dependencies | Design for correlated failure and common-mode faults ([Reliability, Section 8](reliability.md#8-failure-domains-and-blast-radius)) |
| **Retries are harmless** | Retries multiply load and can cause duplicates or metastable failure | Budgets, backoff, idempotency ([Reliability, Section 18](reliability.md#18-overload-and-cascading-failures)) |
| **Messages arrive once and in order** | Networks and brokers reorder, duplicate, and delay messages | Idempotent consumers, per-key ordering, deduplication |
| **A slow node is a dead node (and vice versa)** | A node cannot tell slow from dead; failure detectors are imperfect | Timeouts as explicit policy; fencing; quorum decisions |
| **If it responds, it is healthy** | A node can answer health checks while failing real requests (gray failure) | Measure real request outcomes; passive outlier detection |
| **DNS always works** | DNS fails, caches stale data, and adds latency | Caching with sensible TTLs; fallbacks; monitoring DNS errors |
| **State is consistent everywhere** | Replicas lag and diverge; consistency is a trade-off | Choose consistency per operation ([CAP](cap-theorem.md)) |
| **The test environment matches production** | Scale, topology, data volume, and failure modes differ | Load tests, staging with realistic data, game days, chaos experiments |
| **Observability is free** | Telemetry costs CPU, bandwidth, storage, and money, and can overload the system it watches | Sampling, retention policies, monitoring the monitoring |

## 8. How the Fallacies Apply to Microservices

Microservices communicate over a network, so the eight fallacies apply with particular force.

Consider these dependencies:

`Frontend → Order Service → Inventory Service → Payment Service → Notification Service`

If every operation is synchronous, one slow or unavailable dependency can affect the whole request.

1. The Inventory Service becomes slow.
2. The Order Service waits for its response.
3. Requests begin to accumulate.
4. Threads, connections, or other resources become exhausted.
5. The Order Service becomes slow even though its own code works.
6. The problem spreads to other dependent services.

This is a **cascading failure**.

Also note the arithmetic: with many hard synchronous dependencies, availability multiplies down and tail latency multiplies up (see [Reliability, Section 15.1](reliability.md#151-series-system) and [Latency, Section 9.1](latency.md#91-the-fan-out-problem)).

### Recommended design practices

- Use explicit timeouts for remote calls, and propagate deadlines.
- Limit retries and apply exponential backoff with jitter.
- Use circuit breakers where they help isolate failing dependencies.
- Bound queues and concurrent requests (bulkheads).
- Apply backpressure when downstream systems cannot keep up.
- Use asynchronous messaging when synchronous completion is unnecessary.
- Make message processing idempotent.
- Track distributed traces across service boundaries.
- Set service-level objectives for critical user journeys.

These practices address several fallacies together rather than treating each as an isolated problem.

## 9. How the Fallacies Apply to AWS

A distributed application on AWS faces the same fundamental problems.

| AWS component or pattern | Relevant fallacy | Design consideration |
|---|---|---|
| Application Load Balancer | Topology changes | Register healthy targets, use health checks, and handle changing instances |
| Amazon SQS | Network reliability | Standard queues deliver at least once, so design consumers for retries and duplicates; FIFO queues add deduplication within a time window |
| Amazon DynamoDB | Network reliability and latency | Choose consistency settings and handle throttling and transient errors with retries and backoff |
| AWS SDKs | Network reliability | SDKs include retry logic with backoff; understand the defaults, and avoid stacking your own retries on top |
| Amazon EKS | Topology changes | Use Services and service discovery instead of pod IPs |
| Multi-AZ and multi-Region architecture | Latency and transport cost | Consider cross-zone and cross-region latency, replication lag, and data-transfer charges |
| NAT gateways and managed gateways | Transport cost | Per-GB processing charges can dominate for chatty or high-volume traffic |
| IAM and security groups | Network security | Apply identity-based permissions and least privilege; do not rely on network location alone |
| API Gateway and microservices | Bandwidth and latency | Control payload sizes, request rates, and dependency chains |

Specific guarantees and behaviors depend on the service and configuration. Check the relevant AWS documentation when selecting a service or defining failure handling.

## 10. How the Fallacies Apply to Kubernetes and SRE

For an SRE, the fallacies turn abstract principles into operational practice.

### Example: A production incident

A Kubernetes application sees increased HTTP 5xx errors. A useful investigation might proceed like this:

1. **Check network reliability:** connection errors, timeouts, resets, DNS failures.
2. **Check latency:** compare p50, p95, and p99 against the SLO.
3. **Check bandwidth:** network throughput, packet drops, saturation.
4. **Check security:** network policies, service identities, credentials, recent access-control changes, certificate expiry.
5. **Check topology:** pod restarts, endpoint changes, DNS resolution, load-balancer health.
6. **Check administration and ownership:** recent changes, and engage the relevant platform or network team.
7. **Check transport costs:** excessive requests, payload sizes, cross-region traffic, retry volume.
8. **Check heterogeneity:** version mismatches, incompatible configurations, differences between nodes or regions.

Not every incident involves all eight. The list is a checklist for identifying possible failure mechanisms.

### Example signals

Metric names vary by setup. These are common starting points in a Prometheus-based Kubernetes environment:

| Fallacy | Example signal |
|---|---|
| Reliable network | `node_netstat_Tcp_RetransSegs` (TCP retransmissions), connection reset and timeout counts from your HTTP client metrics |
| Latency | Request duration histograms (p95 and p99), dependency latency per hop from traces |
| Bandwidth | `node_network_transmit_bytes_total` and `node_network_receive_drop_total` (node exporter), `container_network_*` (cAdvisor) |
| Network security | TLS handshake failures, certificate-expiry metrics, authorization-denied counts |
| Topology | `kube_pod_container_status_restarts_total` (kube-state-metrics), endpoint count changes, `coredns_dns_responses_total` by response code |
| One administrator | Change-event annotations on dashboards, deployment and config-change logs |
| Transport cost | Request and retry rates per caller, bytes per request, cross-zone traffic reports |
| Homogeneity | Errors grouped by client or server version, by region, and by node pool |

## 11. Testing for the Fallacies

You cannot confirm a system tolerates a fallacy by reading the code. Inject the failure and watch the behavior.

| Fallacy | Experiment | Example tooling |
|---|---|---|
| Network is reliable | Drop packets, reset connections, black-hole a dependency | `tc netem`, Toxiproxy, Chaos Mesh, service mesh fault injection |
| Latency is zero | Add delay and jitter to a dependency | `tc netem`, Toxiproxy latency toxic |
| Bandwidth is infinite | Throttle bandwidth, saturate a link | `tc` rate limiting, Toxiproxy bandwidth toxic |
| Network is secure | Attempt unauthenticated internal calls; test that network policies block what they should; expire a certificate in staging | Security scanners, policy tests, certificate rotation drills |
| Topology doesn't change | Kill pods and nodes, rotate IPs, flip DNS | Chaos Mesh, Litmus, scripted node drains |
| One administrator | Run game days that require coordination across teams | Incident simulations |
| Transport cost is zero | Load-test with realistic fan-out and measure calls per request and bytes per request | Load generators, tracing |
| Network is homogeneous | Run mixed old and new versions together during a rollout simulation | Canary deployments, contract tests |

A simple local latency and loss experiment on a Linux test host (never on production):

```bash
# Add 100 ms delay (with 20 ms jitter) and 1% packet loss on eth0
sudo tc qdisc add dev eth0 root netem delay 100ms 20ms loss 1%

# ... run your load test and observe timeouts, retries, error rates ...

# Remove the rule when finished
sudo tc qdisc del dev eth0 root netem
```

For each experiment, state a hypothesis first ("with 1% packet loss, checkout success stays above the SLO and no duplicate charges occur"), define an abort condition, and limit the blast radius. See [Reliability, Section 29](reliability.md#29-reliability-testing).

## 12. Remote Call Design Checklist

Before shipping any call across a network boundary, check:

- [ ] A connect timeout and a request timeout are set (no defaults of "infinite").
- [ ] A total deadline is propagated, not just per-hop timeouts.
- [ ] Retries are bounded, use backoff and jitter, and happen at one layer only.
- [ ] The operation is idempotent, or uses an idempotency key.
- [ ] Only retryable errors are retried (not permanent `4xx` errors).
- [ ] There is a defined behavior for an **unknown** outcome (reconciliation).
- [ ] A circuit breaker or concurrency limit protects the caller from a slow dependency.
- [ ] Payload size and call count are reasonable (pagination, batching).
- [ ] The call is authenticated and authorized, and traffic is encrypted.
- [ ] The target is addressed by a stable name (no hardcoded IPs); DNS and connection reuse behavior are understood.
- [ ] The API contract is versioned, and the caller tolerates unknown fields.
- [ ] Latency, errors, retries, and timeouts are measured and traceable.
- [ ] The failure of this dependency has a defined degraded behavior.
- [ ] The dependency's availability and ownership are known.

## 13. Connection to Other System Design Concepts

### Fallacies and the CAP Theorem

CAP explains the trade-off between consistency and availability during a network partition. The fallacies explain why partitions and communication failures must be planned for in the first place. See [CAP Theorem](cap-theorem.md).

### Fallacies and Latency

Fallacy 2 is the starting point for [latency engineering](latency.md): percentiles, tail latency, latency budgets, and deadline propagation.

### Fallacies and Scalability

A system may scale its compute while the network becomes the bottleneck. Bandwidth limits, excessive remote calls, and communication overhead prevent proportional performance gains (see [Scalability, Section 26](scalability.md#26-bottlenecks-and-the-laws-of-scalability)).

### Fallacies and Reliability

Reliable systems do not assume every request succeeds. They use retries, timeouts, idempotency, fault isolation, and recovery mechanisms. [Reliability, Section 5.4](reliability.md#54-the-fallacies-of-distributed-computing) summarizes the fallacies from that perspective.

### Fallacies and Brooks's Law

[Brooks's Law](brooks-law.md) concerns coordination overhead that can make a delayed project later when people are added. The fallacies concern incorrect assumptions about infrastructure and communication. Both show that adding resources does not automatically remove bottlenecks, and both stem from the cost of communication.

### Fallacies and Conway's Law

[Conway's Law](conways-law.md) explains how organizational communication shapes architecture. The "one administrator" fallacy reminds us that large systems cross ownership boundaries, so clear responsibilities and operational coordination are essential.

## 14. Common Mistakes to Avoid

1. **Retrying every failed request indefinitely.** This can amplify an outage. Use bounded retries, backoff, and budgets.
2. **Assuming a timeout means the operation did not happen.** The remote service may have completed it but lost the response.
3. **Using hardcoded IP addresses for dynamic workloads.** Prefer service discovery and stable endpoints.
4. **Creating too many synchronous dependencies.** Each adds latency and failure propagation.
5. **Trusting internal network traffic automatically.** Authenticate and authorize sensitive operations.
6. **Deploying incompatible API changes.** Use backward-compatible changes and staged rollouts.
7. **Ignoring network and data-transfer costs.** Track throughput, payload size, and cross-zone and cross-region traffic.
8. **Treating a healthy service as proof of a healthy user journey.** A service can be healthy in isolation while a dependency fails.
9. **Stacking retries at every layer.** Three layers retrying three times each produces up to 27 attempts.
10. **Relying on default timeouts.** Many libraries default to very long or infinite timeouts.
11. **Trusting wall-clock timestamps across machines** for ordering or conflict resolution.
12. **Testing only the happy path.** Failure behavior that has never been exercised usually does not work.
13. **Assuming load balancing is even.** Long-lived connections can concentrate traffic on a few instances.

## 15. Interview Questions and Answers

### Q1. What are the Fallacies of Distributed Computing?

Eight common assumptions that are false in real distributed systems: reliable network, zero latency, infinite bandwidth, secure network, unchanging topology, one administrator, zero transport cost, and homogeneous infrastructure.

### Q2. Who came up with them?

They grew out of Sun Microsystems: Bill Joy and Tom Lyon identified the first four, L. Peter Deutsch extended the list to seven around 1994, and James Gosling added the eighth in 1997. The list is usually credited to Deutsch.

### Q3. Why is a timeout not proof that a request failed?

The server may have completed the operation but failed to deliver the response. The client cannot always distinguish a failed operation from a successful one with a lost response (the Two Generals Problem).

### Q4. How can you prevent duplicate payment processing?

Use an idempotency key supported by the payment system, persist operation state (including an unknown state), and reconcile uncertain outcomes. A retry must not create a second independent payment for the same logical operation.

### Q5. Is "exactly-once delivery" possible?

Not over an unreliable network in the strict sense: you can choose at-most-once or at-least-once delivery. What systems offer as "exactly once" is typically at-least-once delivery combined with deduplication or idempotent processing, within a documented scope. End-to-end business effects still need idempotent design.

### Q6. How do you prevent cascading failures in microservices?

Timeouts, bounded retries with budgets, circuit breakers, concurrency limits (bulkheads), backpressure, load shedding, sensible dependency boundaries, and asynchronous processing where appropriate.

### Q7. What is the difference between latency and bandwidth?

Latency is how long data or a request takes to travel between two points. Bandwidth is how much data can be transferred per unit of time. Many small calls are limited by latency; large transfers are limited by bandwidth.

### Q8. How does Kubernetes address changing network topology?

Services, endpoint discovery, and DNS let applications connect to stable logical identities instead of pod IPs. Caveats: DNS caching, and per-connection (not per-request) balancing for long-lived HTTP/2 or gRPC connections.

### Q9. How do you secure an API inside a private network?

Service authentication (for example, mutual TLS), per-operation authorization, TLS, least-privilege permissions, network policies, and auditing. Private placement alone is not sufficient.

### Q10. How do these fallacies affect multi-region systems?

Cross-region communication adds latency and transfer cost, and network partitions and replication lag affect consistency, availability, and recovery behavior.

### Q11. Why can you not trust timestamps from different servers?

Clocks drift and can jump, so two machines can disagree about which event came first. Use logical clocks or version vectors for ordering, and avoid last-write-wins on wall-clock time for important data.

### Q12. How would you use these fallacies during an SRE incident?

As a checklist: network errors, latency, saturation, topology changes, security controls, ownership boundaries, communication costs, and version compatibility. Then test the leading hypothesis against metrics and traces.

### Q13. A service calls a dependency with a 30-second default timeout and 3 retries. What is the risk?

A single slow dependency can hold a caller's threads or connections for up to 90 seconds or more per request, exhausting resources and triggering a cascade. Set tight timeouts based on the latency budget, bound the total deadline, limit retries, and add a circuit breaker.

### Q14. How would you test that a system tolerates network failure?

Form a hypothesis (for example, "with 1% packet loss, success rate stays above the SLO and no duplicates occur"), inject faults such as latency, loss, resets, and dependency outages in a controlled environment with an abort condition, and compare behavior to the hypothesis.

### Q15. Why does a deployment have to tolerate mixed versions?

Rollouts and rollbacks are gradual, so old and new versions run together. Changes must be backward and forward compatible (add fields, tolerate unknown ones, expand and contract migrations), or mixed-version operation will fail.

## 16. Cheat Sheet

| Concept | Remember |
|---|---|
| Fallacy 1 | The network fails; timeouts do not mean the operation failed |
| Fallacy 2 | Latency adds up across hops; distance sets a floor |
| Fallacy 3 | Bandwidth is finite; 2 MB × 500 req/s ≈ 8 Gbps |
| Fallacy 4 | Network location is not trust; authenticate and authorize |
| Fallacy 5 | Pods and IPs change; use stable names, watch DNS and long-lived connections |
| Fallacy 6 | Many owners; observability and incident process must cross teams |
| Fallacy 7 | Calls cost time, CPU, and money (including cross-zone transfer) |
| Fallacy 8 | Versions and environments differ; design for mixed-version operation |
| Two Generals | Certainty over an unreliable channel is impossible |
| Safe retry | Timeout, backoff plus jitter, bounded attempts, idempotency key |
| Unknown outcome | Model it explicitly and reconcile; never assume failure |
| Retry amplification | 3 layers × 3 attempts = 27 calls |
| Parallel vs sequential | `max()` of branches vs sum; tail risk grows with fan-out |
| Unofficial additions | Clocks, correlated failures, retries, ordering, gray failure, DNS |
| Testing | Inject latency, loss, resets, and kills; state a hypothesis first |

## 17. Key Takeaways

1. Distributed systems must be designed for partial failure.
2. Network requests can fail, be delayed, or return ambiguous outcomes, and certainty comes from design (idempotency, durable state, reconciliation), not from the network.
3. Latency and bandwidth are different resources, and both shape performance and scalability.
4. Network location does not establish trust.
5. Dynamic infrastructure requires service discovery and resilient connection handling, including DNS and long-lived-connection behavior.
6. Large systems have many administrators, so ownership, observability, and incident processes must cross boundaries.
7. Network communication costs time, resources, and money, and chatty designs pay all three.
8. Components and versions must remain compatible during deployments and rollbacks.
9. The list has aged well; related assumptions about clocks, ordering, retries, and correlated failure cause the same kinds of incidents.
10. Test failure behavior by injecting faults, because untested failure handling usually does not work.

**Remember:** A robust distributed system is not one that assumes the network always works. It is one that behaves predictably when the network, infrastructure, or dependencies do not.

## 18. References

1. L. Peter Deutsch (with contributions from Bill Joy, Tom Lyon, and James Gosling), "The Eight Fallacies of Distributed Computing," Sun Microsystems, 1994 to 1997.
2. Arnon Rotem-Gal-Oz, "Fallacies of Distributed Computing Explained," 2006.
3. Peter Bailis and Kyle Kingsbury, [The Network Is Reliable](https://queue.acm.org/detail.cfm?id=2655736), *ACM Queue*, 2014.
4. Martin Kleppmann, *Designing Data-Intensive Applications*, O'Reilly Media (chapter 8, "The Trouble with Distributed Systems").
5. Amazon Builders' Library, [Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).
6. Google SRE Book, [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/).
7. [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html).
8. [Kubernetes: Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/).
9. NIST Special Publication 800-207, *Zero Trust Architecture*.
10. [Toxiproxy](https://github.com/Shopify/toxiproxy), a TCP proxy for simulating network conditions.
11. Martin Fowler, [Microservices](https://martinfowler.com/articles/microservices.html).

---

*Last updated: 20 October 2026*

*Suggested repository path: `01-fundamentals/fallacies-of-distributed-computing.md`*
