# Scalability in System Design: An In-Depth Guide

> **Topic:** System Design Fundamentals
> **Level:** Beginner to Advanced
> **Focus:** Horizontal Scaling, Vertical Scaling, Distributed Systems, Load Balancing, Caching, Database Scaling, Cloud Architecture, Kubernetes, and SRE
> **Prerequisites:** Basic familiarity with HTTP services, databases, and networking
> **Companion guides:** [Latency in System Design](latency.md) · [Reliability in System Design](reliability.md)
> **Last updated:** October 2026

## Table of Contents

1. [What Is Scalability?](#1-what-is-scalability)
2. [Why Scalability Matters](#2-why-scalability-matters)
3. [Scalability vs Performance vs Reliability](#3-scalability-vs-performance-vs-reliability)
4. [Types of Scalability](#4-types-of-scalability)
5. [Vertical Scaling](#5-vertical-scaling)
6. [Horizontal Scaling](#6-horizontal-scaling)
7. [Horizontal vs Vertical Scaling](#7-horizontal-vs-vertical-scaling)
8. [A Typical Scaling Journey](#8-a-typical-scaling-journey)
9. [Scaling Dimensions](#9-scaling-dimensions)
10. [Stateless vs Stateful Architecture](#10-stateless-vs-stateful-architecture)
11. [Load Balancing](#11-load-balancing)
12. [Caching and Scalability](#12-caching-and-scalability)
13. [Database Scalability](#13-database-scalability)
14. [Database Replication](#14-database-replication)
15. [Database Sharding](#15-database-sharding)
16. [Partitioning Strategies](#16-partitioning-strategies)
17. [Asynchronous Processing and Message Queues](#17-asynchronous-processing-and-message-queues)
18. [Rate Limiting and Admission Control](#18-rate-limiting-and-admission-control)
19. [Auto Scaling](#19-auto-scaling)
20. [Microservices and Scalability](#20-microservices-and-scalability)
21. [Scalability in Distributed Systems](#21-scalability-in-distributed-systems)
22. [Cloud Scalability with AWS](#22-cloud-scalability-with-aws)
23. [Scalability in Kubernetes](#23-scalability-in-kubernetes)
24. [Capacity Planning and Performance Metrics](#24-capacity-planning-and-performance-metrics)
25. [Back-of-the-Envelope Estimation](#25-back-of-the-envelope-estimation)
26. [Bottlenecks and the Laws of Scalability](#26-bottlenecks-and-the-laws-of-scalability)
27. [The CAP Theorem and Scalability](#27-the-cap-theorem-and-scalability)
28. [Cost and Scalability Trade-Offs](#28-cost-and-scalability-trade-offs)
29. [Designing a Scalable E-Commerce System](#29-designing-a-scalable-e-commerce-system)
30. [Designing a Scalable URL Shortener](#30-designing-a-scalable-url-shortener)
31. [Common Scalability Mistakes](#31-common-scalability-mistakes)
32. [Practical SRE Scalability Workflow](#32-practical-sre-scalability-workflow)
33. [System Design Interview Questions](#33-system-design-interview-questions)
34. [Scalability Cheat Sheet](#34-scalability-cheat-sheet)
35. [Key Takeaways](#35-key-takeaways)
36. [References](#36-references)

---

## 1. What Is Scalability?

Scalability is the ability of a system to handle increasing workloads by adding resources while continuing to meet its performance, reliability, and business requirements.

A workload can increase in several ways:

- More users accessing an application.
- More requests per second.
- More data stored in a database.
- More background jobs waiting for processing.
- More services communicating with one another.
- More traffic arriving from different geographic locations.

A scalable system can accommodate this growth without requiring a complete architectural redesign each time demand increases.

### Example: An online shopping application

Suppose an e-commerce application initially serves 1,000 users per day. Its architecture contains one application server, one relational database, and one web frontend.

As the business grows, the application serves 100,000 users per day. The original server may show high CPU utilization, the database may become overloaded, and users may encounter slow responses.

A scalable architecture might introduce:

- Multiple application instances.
- A load balancer.
- A caching layer.
- Database read replicas.
- Asynchronous processing.
- Automatic scaling based on demand.

The objective is not merely to add servers. It is to identify which resources limit growth and scale those resources appropriately.

**Key principle:** Scalability is the ability to grow capacity efficiently as demand increases.

## 2. Why Scalability Matters

Successful applications rarely operate under a fixed workload forever.

### 2.1 Business growth

A startup might begin with hundreds of customers and eventually serve millions. The architecture must accommodate this growth without unacceptable latency or service interruptions.

### 2.2 Traffic spikes

Some systems experience sudden demand increases:

- Ticket booking during a major event.
- E-commerce traffic during a sale.
- Banking transactions during peak business hours.
- Social media traffic after a major announcement.
- Video streaming during a popular live event.

A system designed only for average traffic may fail when peak demand arrives.

### 2.3 Data growth

Applications continuously generate customer records, orders, transactions, logs, metrics, images and videos, and events and messages.

Data growth increases storage requirements, query latency, backup duration, and recovery complexity.

### 2.4 Geographic expansion

An application serving one country may later serve users across multiple continents. This introduces network latency, regional capacity, data replication, and geographic routing challenges.

### 2.5 Cost efficiency

A scalable architecture should add capacity when needed without requiring unnecessary resources at all times. Cloud autoscaling, caching, workload scheduling, and suitable storage choices help control costs.

## 3. Scalability vs Performance vs Reliability

These concepts are related but not interchangeable.

| Concept | Main question | Example |
|---|---|---|
| Scalability | Can the system handle growth? | Increase capacity from 1,000 to 100,000 requests per second |
| Performance | How quickly does it respond? | Reduce API latency from 500 ms to 100 ms |
| Reliability | Does it produce correct results consistently? | Process orders without losing or duplicating them |
| Availability | Is the service usable? | Users can access the checkout API |
| Elasticity | Can capacity follow demand up **and down**, automatically? | Scale from 4 to 40 instances for a sale, then back to 4 |

A system can be fast at low traffic but fail to scale when traffic increases.

A system can scale by adding servers but still have poor performance if every request waits on a slow database query.

**Scalability and elasticity differ.** A system can be scalable (it can be made bigger with effort) without being elastic (it adjusts automatically and releases capacity when demand falls). Elasticity is mainly a cost and operations property; scalability is the structural property that makes it possible.

A scalable system must be designed around measurable workload and service objectives.

## 4. Types of Scalability

There are two primary infrastructure scaling approaches:

1. Vertical scaling (scale up).
2. Horizontal scaling (scale out).

Combining them is sometimes called **diagonal scaling**: grow individual machines to a sensible size, then add more machines.

### 4.1 The scale cube

A broader framework from Abbott and Fisher's *The Art of Scalability* describes three independent axes along which a system can be scaled:

| Axis | Strategy | Description | Typical technique |
|---|---|---|---|
| **X-axis** | Horizontal duplication | Run multiple identical copies behind a load balancer | Cloning stateless instances, read replicas |
| **Y-axis** | Functional decomposition | Split by function or responsibility | Microservices, separating search from checkout |
| **Z-axis** | Data partitioning | Each instance handles a subset of the data | Sharding by user, tenant, or region |

Most large systems use all three: clone stateless services (X), split services by domain (Y), and shard the data that cannot fit on one node (Z).

Use the cube as a diagnostic question: *which axis is actually limiting growth?*

## 5. Vertical Scaling

Vertical scaling, or scaling up, means increasing the resources available to an existing machine: CPU cores, RAM, storage capacity and throughput, and network bandwidth.

### Example

An application runs on a server with 2 vCPUs and 8 GB RAM. As demand increases, it moves to 8 vCPUs and 32 GB RAM. It has more computing capacity without necessarily changing its architecture.

```text
Before:                      After:

Users                        Users
  |                            |
  v                            v
+--------------+             +--------------+
| Application  |             | Application  |
| 2 vCPU       |             | 8 vCPU       |
| 8 GB RAM     |             | 32 GB RAM    |
+--------------+             +--------------+
       |                            |
       v                            v
    Database                     Database
```

### Advantages

- Simple to implement.
- Often requires few architectural changes.
- Avoids distributing application state across many instances.
- Effective for databases and applications with large memory requirements.
- May reduce operational complexity for small workloads.

### Disadvantages

- Hardware and instance sizes have practical limits.
- Larger machines can become disproportionately expensive.
- Upgrading may require a restart or maintenance window.
- A single machine remains a single point of failure.
- Performance does not necessarily increase proportionally with resources (more cores do not help a single-threaded or lock-bound workload).

### When to use vertical scaling

- The workload is small or moderate.
- The application is difficult to distribute.
- A database needs additional memory or CPU.
- The workload fits efficiently on a larger machine.
- Simplicity matters more than very large-scale distribution.

Vertical scaling is often a good first step, but it may not suffice for sustained, large-scale growth. Modern machines are large, so many systems can go much further on one well-tuned node than engineers expect.

## 6. Horizontal Scaling

Horizontal scaling, or scaling out, means adding more machines or instances to distribute the workload.

```text
                  Users
                    |
                    v
              Load Balancer
                    |
         +----------+----------+
         |          |          |
         v          v          v
      Server 1   Server 2   Server 3
         |          |          |
         +------ Shared Services ------+
```

The load balancer distributes requests among available instances according to its routing algorithm and health configuration. If the application is designed correctly, adding instances increases aggregate processing capacity.

### Advantages

- Supports growth beyond the capacity of a single machine.
- Can improve availability when instances occupy independent failure domains.
- Allows capacity to be added incrementally.
- Supports automated scaling.
- Can distribute workloads across multiple availability zones or regions.

### Disadvantages

- Introduces distributed-system complexity.
- Requires load balancing or workload scheduling.
- Shared state must be handled correctly.
- Network communication adds latency and failure modes.
- Data consistency and coordination become harder.
- More instances do not guarantee proportional throughput increases ([Section 26](#26-bottlenecks-and-the-laws-of-scalability)).

### When to use horizontal scaling

- Traffic varies significantly.
- The application can run on multiple instances.
- The workload is large or growing rapidly.
- High availability is required.
- Cloud-native autoscaling is desirable.
- Different components need independent capacity management.

**Important:** Horizontal scaling works best when the architecture is designed to distribute work safely.

## 7. Horizontal vs Vertical Scaling

| Characteristic | Vertical scaling | Horizontal scaling |
|---|---|---|
| Approach | Increase machine resources | Add machines or instances |
| Architecture | Often simpler | Requires distributed coordination |
| Capacity ceiling | Limited by available machine sizes | Can grow across many machines, subject to system limits |
| Fault tolerance | Does not remove a single point of failure | Can improve resilience with proper redundancy |
| State management | Often simpler on one machine | Shared or distributed state must be managed |
| Scaling speed | May require resizing or restart | May add instances automatically |
| Cost model | Larger machines can become expensive | Can use smaller instances, but coordination adds cost |
| Operational complexity | Usually lower initially | Usually higher |
| Typical example | Increase database RAM | Add application replicas |

### Which approach is better?

Neither is universally better. A common strategy:

1. Start with a simple architecture.
2. Measure the actual bottleneck.
3. Optimize the code, queries, and configuration.
4. Scale vertically when it is the simplest effective option.
5. Scale horizontally when workload, availability, or capacity requirements justify distribution.
6. Combine both when necessary.

Do not add servers simply because the user count has increased. Measure requests per second, latency, saturation, and throughput first.

## 8. A Typical Scaling Journey

Systems usually evolve through recognizable stages. Each step is triggered by a **measured** symptom, not by anticipation.

| Stage | Architecture | Trigger to move on |
|---|---|---|
| 1. Single server | App and database on one machine | Resource contention between app and database; need for independent maintenance |
| 2. Separate database | App server and a dedicated database server | App CPU saturates; single app server is a SPOF |
| 3. Stateless app tier + load balancer | Multiple app instances behind a load balancer | Database reads dominate; repeated identical queries |
| 4. Caching and CDN | Cache for hot data, CDN for static and cacheable content | Primary database CPU or I/O saturates on reads |
| 5. Replicas and connection management | Read replicas, connection pooling or a proxy | Write throughput or storage hits single-node limits; slow work blocks requests |
| 6. Asynchronous processing | Queues and worker fleets for non-critical work | Different components need different scaling profiles |
| 7. Functional and data partitioning | Service decomposition, sharding or partitioned stores | Users in multiple regions; regional latency or resilience needs |
| 8. Multi-region | Regional deployments, geo-routing, replicated data | Continuous growth in complexity and cost |

Skipping ahead creates cost and complexity before it is needed. Staying too long creates incidents. The skill is recognizing the **next** bottleneck early and having the measurements to prove it.

## 9. Scaling Dimensions

Not all workloads grow in the same way.

### 9.1 User scaling

More users may increase login requests, API traffic, database reads, and session management activity.

### 9.2 Request scaling

A small number of users can generate enormous traffic if they make frequent requests or automate access. Requests per second (RPS) is often a better capacity measure than registered users.

### 9.3 Data scaling

More data affects storage consumption, index sizes, query plans, backup and restore times, replication lag, and retention costs.

### 9.4 Compute scaling

CPU-intensive workloads include image processing, video encoding, machine learning inference, complex calculations, and large-scale data transformations.

### 9.5 I/O scaling

I/O-intensive workloads depend on database reads and writes, disk throughput, network throughput, object storage access, and message queue operations.

### 9.6 Geographic scaling

Global applications may need to distribute compute and data closer to users. This reduces network latency but introduces replication, data-residency, and regional failure-recovery challenges.

A good architecture identifies the dimension that is **actually** limiting the system rather than applying one strategy everywhere.

## 10. Stateless vs Stateful Architecture

Statelessness is one of the most useful properties for horizontal scaling.

### 10.1 Stateless application

A stateless application instance does not depend on locally retained session state between independent requests. For example, authentication state might be a validated token, while durable business state lives in a database. Any healthy instance can process the next request.

```text
             Client
                |
                v
          Load Balancer
           /         \
          v           v
      Server A     Server B
          \           /
           \         /
              Database
```

### 10.2 Stateful application

A stateful application depends on state retained within a specific process or machine. For example, a shopping cart held only in local memory is lost, or invisible, if the next request reaches another instance.

This can require session affinity, shared storage, or a redesign of the state-management strategy.

### 10.3 Making application tiers easier to scale

- Store durable business state in an appropriate data store.
- Use a shared session store when server-side sessions are needed.
- Use signed tokens when suitable for security and revocation requirements.
- Avoid relying on local files for shared business data.
- Make instances replaceable ("cattle, not pets").
- Externalize configuration and secrets.
- Ensure background work can be distributed safely.

Statelessness does not eliminate state. It moves state into components designed to manage it.

### 10.4 Stateful systems can scale horizontally too

Databases, distributed caches, and streaming systems scale horizontally using replication, partitioning, sharding, or consensus. They are harder to scale because state must remain correct across distributed components.

## 11. Load Balancing

A load balancer distributes incoming requests across multiple backend instances, preventing one server from receiving all traffic while others sit underused.

```text
                Users
                   |
                   v
             Load Balancer
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
    App A        App B        App C
```

### 11.1 Layer 4 vs Layer 7

| Type | Operates on | Capabilities | Typical use |
|---|---|---|---|
| **L4** (transport) | TCP/UDP connections | Very fast, protocol-agnostic, no content awareness | High-throughput TCP/UDP, TLS passthrough |
| **L7** (application) | HTTP/gRPC requests | Routing by path, host, or header; TLS termination; retries; request-level metrics | Web and API traffic, canary routing |

L7 balancing enables per-request distribution, which matters for long-lived connections (HTTP/2, gRPC, WebSockets): an L4 balancer distributes **connections**, so a few long-lived connections can leave load badly skewed.

### 11.2 Common load-balancing algorithms

- **Round robin:** Requests distributed sequentially. Suits similar-capacity servers and similar-cost requests.
- **Weighted round robin:** Servers receive traffic in proportion to assigned weights. Suits mixed capacities.
- **Least connections:** Prefer the server with the fewest active connections. Useful when request durations vary.
- **Least response time:** Considers measured latency or load, depending on implementation.
- **Power of two choices:** Pick two servers at random and send the request to the less loaded one. Achieves much better balance than pure random with very little state.
- **Consistent hashing:** Keys map to nodes in a way that minimizes remapping when membership changes ([Section 15.2](#152-consistent-hashing)). Useful for caches and partitioned systems.

### 11.3 Session affinity (sticky sessions)

Sticky sessions route a client to the same backend. They can work around stateful designs, but they cause uneven load, make draining and scale-in harder, and lose state when the instance fails. Prefer externalizing state.

### 11.4 Health checks

Load balancers avoid unhealthy targets based on configured health checks. Checks should reflect whether the target can safely handle the relevant traffic. See [Reliability, Section 16](reliability.md#16-failure-detection-and-health-checks).

### 11.5 Global load balancing

Traffic can be steered across regions using DNS-based routing (latency, geolocation, weighted, failover) or anycast-based edge networks. DNS caching means routing changes propagate with a delay governed by TTLs and resolver behavior.

### 11.6 Load balancing limitations

A load balancer does not solve every bottleneck. If all application instances depend on an overloaded database, adding more application servers can make the database problem worse.

The load balancer itself, its configuration, and its dependencies must also meet the system's capacity and availability requirements.

## 12. Caching and Scalability

Caching stores reusable data in a faster or more convenient location to avoid repeatedly computing or fetching it. It is often one of the most effective ways to reduce backend load.

### 12.1 Example

A product catalog endpoint receives 10,000 requests per second. If each request queries the database, the database handles 10,000 reads per second.

If 90% of requests are served from cache, the database sees about `10,000 × 0.10 = 1,000` reads per second. Raising the hit ratio to 99% cuts that to 100.

| Hit ratio | Reads reaching the database (of 10,000/s) |
|---:|---:|
| 0% | 10,000 |
| 90% | 1,000 |
| 99% | 100 |
| 99.9% | 10 |

The last few points of hit ratio matter more than they look: going from 90% to 99% reduces database load **tenfold**. This illustration assumes hits eliminate the corresponding reads and ignores refreshes and other queries.

### 12.2 Cache locations

- Browser cache.
- CDN or edge cache.
- Application-local (in-process) cache.
- Distributed cache.
- Database buffer cache.
- Materialized views and precomputed results.

Multiple layers compound: each layer absorbs traffic before it reaches the next.

### 12.3 Cache-aside pattern

1. The application checks the cache.
2. On a hit, it returns the cached result.
3. On a miss, it queries the database.
4. It stores the result in the cache.
5. It returns the result.

```text
Client
  |
  v
Application
  |
  v
Cache ---- Cache hit ----> Return data
  |
  | Cache miss
  v
Database
  |
  v
Update cache and return data
```

### 12.4 Cache hit ratio

`Hit ratio = Cache hits / Total cache lookups × 100`

With 900,000 hits out of 1,000,000 lookups, the hit ratio is 90%.

A high hit ratio is useful only if the cached data is appropriate, sufficiently fresh, and correct for the use case.

### 12.5 Cache invalidation

Cached data becomes stale when the underlying data changes. Common approaches:

- Time-to-live (TTL).
- Explicit invalidation on write.
- Write-through caching.
- Refresh-ahead.
- Versioned cache keys.

### 12.6 Cache stampede

A cache stampede occurs when many requests simultaneously try to regenerate the same expired or missing value. Mitigations:

- Request coalescing (single-flight).
- Jittered expiration times.
- Background refresh.
- Short-lived stale responses where safe.
- Rate limiting expensive regeneration.

### 12.7 Scaling the cache tier itself

A cache is a stateful system and has its own limits.

- **Capacity and eviction:** know your working-set size and eviction policy (LRU, LFU, TTL).
- **Partitioning:** distribute keys across cache nodes with consistent hashing so adding or removing a node does not invalidate most of the cache.
- **Hot keys:** a single very popular key can saturate one node regardless of cluster size. Mitigate with local in-process caching, key replication, or request coalescing.
- **Failure behavior:** if the cache fails or is cold, all traffic falls through to the database. Capacity-plan the database for a degraded-cache scenario, or protect it with load shedding. See [Reliability, Section 18](reliability.md#18-overload-and-cascading-failures).

### 12.8 When caching helps

Caching works well for product catalogs, static content, frequently accessed reference data, read-heavy APIs, and repeated computations. It is less straightforward when data changes frequently or every request needs an authoritative, strongly consistent result.

## 13. Database Scalability

Databases are common bottlenecks because many operations depend on reading or modifying shared state.

Techniques include query optimization, indexing, vertical scaling, read replicas, sharding, partitioning, connection pooling, caching, data archival, and workload separation.

### 13.1 Optimize before distributing

Before introducing a distributed database architecture, examine:

- Slow queries.
- Missing or ineffective indexes.
- Excessive database round trips (N+1 queries).
- Unnecessary joins.
- Lock contention.
- Connection exhaustion.
- Inefficient pagination (for example, large `OFFSET` values; prefer keyset pagination).
- Large transactions.
- Poor data-retention policies.

A well-indexed query can be far more effective than adding replicas to compensate for inefficient access patterns.

### 13.2 Connection pooling

Opening a new connection per request creates significant overhead. A connection pool reuses a bounded number of connections.

The pool must be sized across the **entire fleet**. If 100 application instances each open 100 connections, the database may receive up to 10,000 connections. Scaling application instances without accounting for connection limits can cause an outage.

Remedies: smaller per-instance pools, a connection proxy or pooler that multiplexes many client connections onto fewer database connections, and autoscaling limits (`maxReplicas`) that respect the database's connection budget.

### 13.3 Choosing and combining data stores

No single store is best for every access pattern. Choose by access pattern and required guarantees:

| Need | Common fit | Scaling note |
|---|---|---|
| Transactions, relational integrity | Relational database | Scales up well; scale out via replicas and, eventually, sharding |
| Key-based lookups at very high volume | Key-value or wide-column store | Scales out by partition key; design access patterns up front |
| Flexible documents | Document store | Scales by sharding; avoid cross-document transactions at scale |
| Full-text and faceted search | Search engine | Scale queries with replicas and indexing with shards |
| Time-series metrics and logs | Time-series or columnar store | Partition by time; apply retention and downsampling |
| Large blobs and media | Object storage | Effectively unbounded; front with a CDN |
| Analytics over large datasets | Columnar warehouse | Keep analytical load off the transactional database |

A single system often uses several (polyglot persistence). The cost is more operational surface and consistency work between stores.

### 13.4 Separating reads from writes (CQRS)

Command Query Responsibility Segregation separates the write model from one or more read models optimized for queries. Reads are served from denormalized, precomputed views kept up to date from change events.

This can scale reads independently and simplify hot read paths, at the price of **eventual consistency** between the write and read sides and more moving parts. Use it where read and write needs genuinely diverge, not by default.

### 13.5 Hot, warm, and cold data

Not all data needs the same storage. Keep recent, frequently accessed data on fast storage; move older data to cheaper tiers or archives. Partition by time so retention is a cheap partition drop rather than a huge delete. This keeps working sets, indexes, and backups small.

## 14. Database Replication

Replication maintains multiple copies of data across database instances.

### 14.1 Primary-replica architecture

The primary handles writes; replicas receive changes and may serve reads.

```text
          Application
               |
        +------+------+
        |             |
      Writes         Reads
        |             |
        v             v
      Primary ----> Replica 1
        |
        +----------> Replica 2
```

The exact read and write routing depends on the database architecture.

### 14.2 Advantages

- Increases read capacity.
- Supports failover.
- Can distribute reads geographically.
- Can separate some read workloads (reports, analytics) from writes.

### 14.3 Limitations

- Replication lag can make reads stale.
- Replicas consume additional resources.
- **Writes are still limited by the single primary.** Replicas scale reads, not writes.
- Failover requires coordination and recovery procedures.
- Not every query can safely use a replica.

### 14.4 Read-after-write consistency

Suppose a customer updates their address and immediately reads their profile. If the read reaches a lagging replica, the old address is returned.

Solutions:

- Read from the primary for a defined interval after a write.
- Use session consistency mechanisms where supported.
- Wait for replication progress where appropriate.
- Use a database that provides the required consistency guarantees.

### 14.5 Multi-primary and leaderless replication

To scale writes across regions, some systems allow multiple nodes to accept writes. That introduces **write conflicts** that must be resolved (last-write-wins, version vectors, CRDTs, application-level merge). Treat it as a deliberate consistency trade-off, not a free capacity upgrade. See [Reliability, Section 11](reliability.md#11-consensus-quorums-and-leader-election).

Replication improves read capacity and availability; it is not the same as sharding.

## 15. Database Sharding

Sharding distributes records across multiple independent database partitions (shards). Each shard stores a subset of the data.

```text
                 Application
                      |
                 Shard Router
                      |
      +---------+-----+-----+---------+
      |         |           |         |
      v         v           v         v
   Shard 1   Shard 2     Shard 3   Shard 4
```

The diagram is conceptual: each shard owns a defined subset of records, and a routing layer determines the correct destination.

### 15.1 Hash-based sharding

`shard_id = hash(user_id) % number_of_shards`

Advantages:

- Distributes records relatively evenly.
- Supports direct routing when the shard key is known.

Disadvantages:

- **Changing the number of shards remaps most keys.** Going from 4 to 5 shards with simple modulo moves about 80% of keys (only keys where `h mod 4 = h mod 5`, roughly 20%, stay put).
- Range queries are difficult.
- Poor hash distribution can create hotspots.

### 15.2 Consistent hashing

Nodes and keys are placed on a hash ring; each key belongs to the next node clockwise. Adding a node moves only the keys between it and its predecessor, about `1 / (N + 1)` of all keys on average. Adding a fifth node to four moves roughly 20%, instead of about 80% with modulo.

Practical refinements:

- **Virtual nodes:** each physical node owns many points on the ring, which evens out load and lets stronger machines take more.
- **Replication:** store each key on the next `k` distinct nodes for durability.

### 15.3 Range-based sharding

Records are assigned by ranges of a shard key (for example, customer IDs 1 to 1,000,000 on shard A).

Advantages:

- Efficient range queries.
- Easy-to-understand mapping.

Disadvantages:

- Some ranges receive much more traffic than others.
- Sequential keys (timestamps, auto-increment IDs) create write hotspots on the newest range.

### 15.4 Directory-based sharding

A lookup service maps each key (or key range) to a shard. This allows flexible placement and rebalancing, at the cost of an extra lookup and a critical component that must itself be highly available and cached.

### 15.5 Geographic sharding

Data is assigned by geography (country or region). This helps locality and data-residency requirements, but cross-region operations, failover, and global queries become more complicated.

### 15.6 Planning for resharding

Resharding live data is expensive and risky. Reduce the pain up front:

- Create **many more logical shards than physical nodes** (for example, 1,024 logical shards mapped to 8 machines). Rebalancing then moves whole logical shards instead of rehashing keys.
- Keep the shard mapping in a versioned, replicated directory.
- Migrate with dual-writes or change-data-capture, verify, then cut over reads.

### 15.7 Sharding challenges

- Cross-shard joins and queries (scatter-gather, whose latency is the slowest shard).
- Distributed transactions.
- Rebalancing.
- Uneven data distribution and hot shards.
- Global uniqueness of IDs.
- Operational complexity.
- Backup and recovery coordination.

Introduce sharding when simpler approaches cannot meet the workload or when the growth trajectory clearly justifies the complexity.

## 16. Partitioning Strategies

Partitioning divides a dataset into smaller, manageable sections. A partition can live within a table, across storage files, or across a distributed architecture. Sharding is horizontal partitioning across database nodes.

### 16.1 Horizontal partitioning

Rows are divided into partitions. Example: an orders table partitioned by order date (one partition per month). This helps partition pruning, retention management, and query efficiency for suitable access patterns.

### 16.2 Vertical partitioning

Columns or responsibilities are separated. Example: profile and preference data stored separately from large binary assets. This reduces unnecessary data access, but does not automatically increase write capacity.

### 16.3 Choosing a partition key

Consider:

- **Cardinality:** enough distinct values to spread data.
- **Query patterns:** most queries should target one partition.
- **Write distribution:** avoid monotonically increasing keys as the sole key.
- Data volume and growth rate.
- Ordering requirements.
- Rebalancing needs.

A poor partition key creates hotspots even when many partitions exist. Examples of risky keys: country (a few huge values), a timestamp alone (all writes hit the newest partition), or a celebrity user ID.

Mitigation for a hot key: add a suffix or salt (for example, `key#0` to `key#9`) to spread writes, then merge on read.

## 17. Asynchronous Processing and Message Queues

Not every operation needs to complete before the user receives a response. Asynchronous processing moves non-immediate work into a queue so producers and consumers can operate at different rates.

### Example

When a customer places an order, the application may need to validate it, persist it, return a confirmation, send an email, generate analytics events, and update reporting. Sending emails and generating analytics events can usually happen asynchronously.

```text
Client
  |
  v
Order Service
  |
  +----> Order Database
  |
  +----> Durable Queue
               |
        +------+------+
        |             |
        v             v
    Email Worker  Analytics Worker
```

### Advantages

- Absorbs temporary traffic bursts.
- Decouples producers and consumers.
- Allows independent worker scaling.
- Keeps non-critical work off the synchronous request path.
- Buffers during downstream slowdowns.

### Design considerations

- Durable message storage.
- Delivery guarantees.
- Idempotent consumers.
- Retry policies and dead-letter queues.
- Ordering requirements.
- Retention.
- Consumer concurrency.
- Backpressure.
- Message age and processing deadlines.

### Queue stability

Let `λ` be the average arrival rate, `μ` the average processing rate per worker, and `c` the number of equivalent workers. Nominal capacity is `c × μ`. For a stable queue under sustained load:

`λ < c × μ`

Utilization is `ρ = λ / (c × μ)`. As `ρ` approaches 1, queueing delay grows steeply (see [Latency, Section 12](latency.md#12-queueing-theory-and-littles-law)).

If incoming work consistently exceeds capacity, the backlog grows until work is delayed, rejected, or storage limits are reached.

### Backlog drain time

After an incident or spike, how long does recovery take? With backlog `B` and spare capacity `(c × μ − λ)`:

`Drain time = B / (c × μ − λ)`

Example: 100,000 messages are queued, workers can process 1,200 per second, and 1,000 per second are still arriving:

`100,000 / (1,200 − 1,000) = 500 seconds`

The thinner the margin between capacity and arrivals, the longer recovery takes. At `c × μ = 1,050` the same backlog would take 2,000 seconds. Running workers near saturation turns a short outage into a long delay.

### Parallelism limits

- Partitioned logs (for example, Kafka-style topics) cap consumer parallelism at the **partition count**: extra consumers in a group beyond the number of partitions sit idle. Choose partition counts with future parallelism in mind, since changing them later can disturb key ordering.
- **Ordering vs parallelism:** strict global ordering forces serial processing. Preserve order only per key (for example, per order ID) and parallelize across keys.
- Adding workers helps only if downstream systems can handle the extra load.

### Scaling consumers

Scale on **backlog and message age**, not only CPU. A useful signal is backlog per worker (queue depth divided by worker count) compared with the number of messages one worker can drain within the acceptable delay.

## 18. Rate Limiting and Admission Control

Capacity is finite. Rate limiting and admission control protect the system (and fair access among clients) when demand exceeds it.

### 18.1 Why limit

- Prevent one client or tenant from exhausting shared resources.
- Protect downstream dependencies (databases, third-party APIs).
- Keep admitted requests fast instead of letting every request slow down.
- Blunt abuse and accidental retry storms.

### 18.2 Common algorithms

| Algorithm | How it works | Notes |
|---|---|---|
| **Token bucket** | A bucket holds up to `b` tokens, refilled at rate `r`; each request spends a token | Allows bursts up to `b`, sustained rate `r`. Very common. |
| **Leaky bucket** | Requests enter a queue drained at a fixed rate | Smooths output; excess is queued or dropped |
| **Fixed window counter** | Count requests per fixed time window | Simple, but allows up to ~2x the limit across a window boundary |
| **Sliding window (log or counter)** | Count requests in a rolling window | More accurate, more memory or computation |

Example: a token bucket with `r = 100` tokens per second and `b = 200` lets a client burst 200 requests instantly, then sustain 100 per second.

### 18.3 Distributed rate limiting

Counters must be shared across instances. Common approaches:

- A central fast store with atomic increments (for example, Redis with `INCR` and a TTL, or a Lua script for token-bucket logic).
- Local limiters per instance with a share of the global limit, accepting some inaccuracy.
- Limiting at the edge (API gateway, CDN) before traffic reaches your services.

Choose the limiting key deliberately: API key, user ID, tenant, IP address, or endpoint. IP-based limits misfire behind NAT and are easy to evade.

### 18.4 Responding to over-limit requests

Return `429 Too Many Requests` with a `Retry-After` header, and make clients back off with jitter. Prioritize critical traffic (for example, checkout over browsing) when shedding load.

For deeper treatment of overload behavior, see [Reliability, Section 18](reliability.md#18-overload-and-cascading-failures).

## 19. Auto Scaling

Auto scaling automatically adjusts infrastructure capacity based on demand or configured schedules, matching capacity to workload without manual intervention.

### 19.1 Reactive scaling

Uses observed metrics: CPU utilization, request count per target, queue depth, memory (where supported), and custom application metrics.

### 19.2 Scheduled scaling

Increases or decreases capacity at known times. If a business expects high traffic during a scheduled promotion, capacity can be added before it begins.

### 19.3 Predictive scaling

Uses historical patterns or forecasts to provision capacity ahead of anticipated demand. Effectiveness depends on workload predictability and what the scaling system supports.

### 19.4 Scaling policies

- **Target tracking:** adjust capacity to keep a metric near a target.
- **Step scaling:** add or remove capacity in steps tied to thresholds.
- **Scheduled scaling:** apply changes at known times.

Different cloud services expose different options.

### 19.5 The scaling calculation

Most target-tracking logic reduces to proportional control. Kubernetes' HPA, for instance, uses:

`desiredReplicas = ceil( currentReplicas × currentMetric / targetMetric )`

Example: 3 replicas at 90% average CPU with a 60% target:

`ceil(3 × 90 / 60) = ceil(4.5) = 5`

Choose targets with headroom. A 60% CPU target leaves capacity to absorb traffic growth during the minutes it takes new instances to arrive, whereas a 90% target leaves almost none.

### 19.6 Scale-out vs scale-in

- **Scale-out:** add capacity.
- **Scale-in:** remove excess capacity.

Scale-in requires care. Removing an instance can interrupt long-running requests, connections, or in-flight work. Use graceful shutdown, connection draining, and workload termination handling.

Scale in **more slowly than you scale out** (a stabilization or cooldown window) to avoid **flapping**, where capacity oscillates up and down as metrics fluctuate.

### 19.7 Autoscaling limitations

Autoscaling is not instantaneous. The system must detect demand, provision capacity, start applications, warm caches, pass readiness checks, and register instances.

**Worked example:** traffic doubles in 60 seconds. Metrics are evaluated every 60 seconds, new instances take 3 minutes to become ready, and the fleet was running at 60% utilization. Demand reaches 120% of existing capacity within a minute, and relief arrives roughly 4 or more minutes later. Without headroom or other protection, that is several minutes of overload.

For predictable spikes, pre-scale. For unpredictable ones, combine headroom, fast-starting images, warm pools, load shedding, and rate limits.

Also remember what autoscaling does **not** fix: a saturated database, an exhausted quota, or a dependency with its own limits. Set sensible maximums.

## 20. Microservices and Scalability

Microservices split an application into independently deployable services organized around specific responsibilities. They can enable independent scaling when workloads differ across functions (the Y-axis of the scale cube).

### Example

An e-commerce platform has catalog, search, cart, order, payment, and notification services. During a sale, search traffic rises dramatically while payments grow more slowly. Microservices can scale search separately from payments.

### Advantages

- Independent deployment and scaling.
- Different technology choices where justified.
- Better isolation of some workloads.
- Clearer ownership boundaries.

### Disadvantages

- More network calls.
- Distributed failure modes.
- More complex tracing and monitoring.
- Service discovery and configuration needs.
- More deployment and security boundaries.
- Data consistency challenges.
- Higher operational overhead.

### Important principle

Microservices do not automatically make a system scalable. A monolith can scale horizontally if designed appropriately, and a poorly designed microservices system can be **less** scalable than a well-designed monolith because every request may require many synchronous network calls.

Choose service boundaries by domain responsibility and independent scaling needs, not by splitting every function into a separate service.

### Multi-tenancy and noisy neighbors

When many tenants share infrastructure, one heavy tenant can degrade others. Use per-tenant quotas and rate limits, isolated resource pools, and, at larger scale, cell-based isolation. See [Reliability, Section 8](reliability.md#8-failure-domains-and-blast-radius).

## 21. Scalability in Distributed Systems

Distributed systems use multiple machines to store data or execute work. They allow scaling beyond one machine but introduce coordination overhead.

### 21.1 Network overhead

Communication between components consumes time and resources. A call that was an in-process function now requires a network round trip, serialization, and failure handling.

### 21.2 Coordination overhead

Distributed systems may need to coordinate ownership, leadership, transactions, replication, ordering, membership, and distributed locks. As components multiply, coordination can become the bottleneck. The best scalable designs **minimize coordination** by partitioning work so units rarely need to talk to each other.

### 21.3 Hotspots

A hotspot occurs when one component, partition, or key receives disproportionately high traffic. Example: a popular product has millions of views, but every request hits the same cache key or database partition.

Mitigations:

- Caching.
- Request coalescing.
- Partition-key redesign (salting).
- Replicated read paths.
- Workload isolation.

### 21.4 Fan-out

Fan-out occurs when one request triggers many downstream requests. If one request calls 20 services, traffic and failure exposure multiply (see [Latency, Section 9](latency.md#9-tail-latency-in-distributed-systems)).

Techniques:

- Parallelizing independent operations carefully.
- Caching.
- Reducing unnecessary dependencies.
- Setting deadlines.
- Limiting concurrency.
- Aggregating data asynchronously.

#### Fan-out on write vs fan-out on read

A classic design choice, for example for a social feed:

| Approach | How it works | Strength | Weakness |
|---|---|---|---|
| **Fan-out on read** | Compute the feed at read time by querying everyone you follow | Cheap writes, always fresh | Expensive, slow reads |
| **Fan-out on write** | Precompute each follower's feed when a post is created | Fast reads | Expensive writes; a user with millions of followers triggers millions of writes |

Large systems often combine them: fan out on write for typical users, but handle very popular accounts by merging their posts at read time. The lesson generalizes: **decide whether to pay at write time or read time based on which is more frequent and which has the tighter latency target.**

### 21.5 Backpressure

Backpressure prevents a faster producer from overwhelming a slower consumer, through bounded queues, admission control, rate limits, or producer throttling. It is a scalability mechanism because it keeps overload from becoming uncontrolled resource consumption.

## 22. Cloud Scalability with AWS

AWS provides services for scaling different parts of an application. The same concepts exist on other clouds.

### 22.1 Amazon EC2 Auto Scaling

Adjusts the number of EC2 instances based on policies and capacity requirements. Typical use: web fleets, application clusters, worker instances, variable compute workloads.

### 22.2 Elastic Load Balancing

Distributes traffic across eligible targets (EC2 instances, containers, IP addresses, and other supported endpoints), with Application (L7) and Network (L4) load balancer types serving different needs.

### 22.3 Amazon CloudFront

A content delivery network that caches and delivers supported content closer to users, reducing origin traffic and improving delivery performance for cacheable content.

### 22.4 Amazon ElastiCache

Managed caching that can reduce repeated database reads and improve response times. It does not replace durable storage for data that must survive eviction or loss.

### 22.5 Amazon RDS and Aurora

Managed relational databases with options for compute sizing, read scaling, storage growth, and replication. Read replicas increase read capacity for suitable workloads; write scaling and transaction contention need separate analysis.

### 22.6 Amazon DynamoDB

A managed NoSQL database designed to scale supported workloads with appropriate data modeling and capacity configuration. Partition-key design matters: a hot partition or poorly distributed access pattern limits throughput.

### 22.7 Amazon SQS

Managed message queues that decouple producers and consumers. Consumers can be scaled by backlog, processing time, downstream capacity, and service limits.

### 22.8 Amazon S3

Object storage for large datasets, media, backups, logs, and data-lake workloads. It removes the need to manage a storage-server fleet for many object use cases, though request patterns, object layout, and downstream processing still matter.

### 22.9 Serverless compute

Functions-as-a-service scale out automatically per request, with no instances to manage, but they have **concurrency quotas** (an account-level default per region, adjustable), cold starts, and per-invocation limits. The scaling boundary moves to the downstream systems: thousands of concurrent function invocations can overwhelm a database that was sized for a few hundred connections.

### 22.10 Quotas and limits

Every managed service has quotas (API rate limits, concurrency, throughput, IP addresses, connections). Track utilization against quotas, request increases ahead of need, and include quota headroom in capacity plans. Hitting a limit during a spike is a common cause of scaling failures.

### Example AWS architecture

```text
                 Users
                   |
                   v
             Amazon CloudFront
                   |
                   v
           Application Load Balancer
                   |
                   v
          EC2 Auto Scaling Group
                   |
           +-------+-------+
           |               |
           v               v
       Amazon RDS     Amazon ElastiCache
           |
           v
       Amazon S3

Background processing:

Application --> Amazon SQS --> Worker Fleet
```

This is one possible architecture. The right design depends on access patterns, consistency requirements, latency targets, and workload characteristics.

## 23. Scalability in Kubernetes

Kubernetes provides mechanisms for running and scaling containerized applications.

### 23.1 Pods and Deployments

A Deployment manages the desired number of replicas. Increasing the replica count allows more Pods to process requests, assuming the application and its dependencies can handle the load.

### 23.2 Horizontal Pod Autoscaler (HPA)

Adjusts Pod replica counts based on configured metrics: CPU utilization, memory utilization (where configured), custom application metrics, and external metrics.

Important details:

- **CPU utilization targets are measured relative to the container's CPU _request_**, not its limit or the node's capacity. If requests are set unrealistically low or high, HPA behavior is distorted.
- The HPA needs a metrics source (for example, the metrics server) and meaningful requests set on the Pods.
- Scale-down is deliberately conservative: by default, a stabilization window delays scale-in to avoid flapping.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 30            # respect database connection budgets
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

### 23.3 Event-driven autoscaling

CPU is a poor signal for queue consumers. Tools such as KEDA scale workloads on event sources (queue length, stream lag, custom metrics), including scaling to zero when idle. The HPA can also consume custom or external metrics directly.

### 23.4 Vertical Pod Autoscaler (VPA)

Recommends or adjusts CPU and memory requests according to its mode. Some update modes restart Pods to apply changes. Configure HPA and VPA carefully when both act on CPU or memory, to avoid conflicting behavior.

### 23.5 Cluster Autoscaler

Adds or removes cluster nodes when scheduling and capacity conditions require it. Adding Pods does not help if there is no node capacity to schedule them. Alternatives such as Karpenter provision nodes directly based on pending Pod requirements.

### 23.6 The three scaling layers

| Component | What it scales |
|---|---|
| HPA | Number of application Pod replicas |
| VPA | CPU and memory resource requests or recommendations |
| Cluster Autoscaler | Number of cluster nodes |

These address different capacity constraints.

### 23.7 Example

A service starts with two Pods.

1. Incoming traffic increases.
2. CPU utilization rises.
3. The HPA increases the desired replica count.
4. The scheduler attempts to place the new Pods.
5. If the cluster lacks capacity, the Pods stay `Pending`.
6. The Cluster Autoscaler may provision nodes.
7. New Pods start and pass readiness checks.
8. Traffic is routed to the new Pods.

Each step adds delay (metric collection, node provisioning, image pull, startup, readiness). The end-to-end reaction time can be several minutes, so headroom and pre-scaling still matter.

### 23.8 Kubernetes scalability checklist

- Multiple replicas for horizontally scalable workloads.
- CPU and memory requests that reflect real usage.
- Application-level saturation metrics.
- HPA where appropriate, with a sensible `maxReplicas`.
- Sufficient cluster capacity and node autoscaling.
- Readiness probes.
- Replicas spread across failure domains.
- Monitoring of pending Pods and scheduling failures.
- Protection for databases and external services from excessive concurrency.
- Graceful shutdown and connection draining.

For Pod-level reliability configuration (PDBs, probes, topology spread), see [Reliability, Section 23](reliability.md#23-reliability-in-kubernetes).

## 24. Capacity Planning and Performance Metrics

Capacity planning estimates the resources needed to support expected workloads.

### 24.1 Requests per second

RPS measures the rate at which requests arrive or are processed. An API receiving 60,000 requests over 60 seconds averages:

`60,000 / 60 = 1,000 RPS`

This hides burstiness: traffic may arrive evenly or in sudden spikes. Plan for **peak**, not average. A common starting assumption is that peak is a small multiple of average (for example, 2x to 5x), but measure your own traffic.

### 24.2 Concurrency and Little's Law

`L = λ × W`

- `L` = average number of requests in the system.
- `λ` = average throughput.
- `W` = average time in the system.

An API processing 1,000 requests per second with an average time in the system of 0.2 seconds has about `1,000 × 0.2 = 200` requests in flight on average.

That is an average, not the concurrency needed at peak, and it is the same relationship used to size thread and connection pools (see [Latency, Section 12.3](latency.md#123-sizing-pools-with-littles-law)).

### 24.3 Throughput

The amount of work completed per unit of time: requests per second, transactions per second, messages per second, gigabytes per hour.

### 24.4 Latency percentiles

Average latency hides slow requests. Track P50, P95, and P99. If P99 latency is 2 seconds, about 1% of measured requests take at least around that long, subject to measurement details. See [Latency, Sections 6 to 8](latency.md#6-latency-percentiles-p50-p90-p95-p99-and-p999).

### 24.5 Resource utilization

Monitor CPU utilization and throttling, memory, disk throughput and latency, network throughput, database connections, queue depth, and worker saturation. High utilization is not automatically bad, but sustained saturation leads to queueing, elevated latency, and errors.

### 24.6 Capacity headroom

Headroom is the difference between available capacity and current demand. A system near its maximum sustainable throughput has no room for spikes or component failures. Plan for peak demand, scaling delay, dependency limits, and the capacity required during failures.

For failure scenarios, compute N+1 or zone-loss capacity explicitly (for example, surviving the loss of one of three zones requires about 150% of peak capacity in total). See [Reliability, Section 9.1](reliability.md#91-capacity-for-failure-n1-and-beyond).

### 24.7 Scaling efficiency

Measure how well added resources translate into throughput:

`Scaling efficiency = Throughput(N) / (N × Throughput(1))`

A perfectly linear system has efficiency 100% at every `N`. If 8 instances deliver 6x the throughput of one, efficiency is `6 / 8 = 75%`. Falling efficiency reveals shared bottlenecks, coordination costs, or contention ([Section 26](#26-bottlenecks-and-the-laws-of-scalability)).

### 24.8 Define maximum sustainable throughput

Rather than testing for "peak RPS," find the **highest load at which the system still meets its latency and error SLOs**. That value, per instance, is the unit you use for capacity planning. See [Latency, Section 24](latency.md#24-load-testing-and-capacity-validation).

## 25. Back-of-the-Envelope Estimation

Rough calculations at the start of a design reveal whether you are building for 100 RPS or 100,000 RPS, and which resource will break first. Aim for the right order of magnitude, not precision.

### 25.1 Useful numbers

| Quantity | Approximate value |
|---|---|
| Seconds in a day | 86,400 (about 10⁵) |
| Seconds in a month | about 2.6 million |
| Seconds in a year | about 31.5 million |
| 1 KB / 1 MB / 1 GB | 10³ / 10⁶ / 10⁹ bytes (approximately) |
| 1 Gbps | about 125 MB/s |

### 25.2 Method

1. State assumptions explicitly (users, actions per user, object sizes, read/write ratio, retention).
2. Convert to **average** RPS, then apply a **peak factor**.
3. Estimate **storage** (records × size × retention, plus replication and index overhead).
4. Estimate **bandwidth** (RPS × payload size).
5. Estimate **instance count** from a measured or assumed per-instance capacity, then add headroom.
6. Identify the component that hits its limit first.

### 25.3 Worked example: a read-heavy API

Assumptions: 10 million daily active users, 20 requests per user per day, average response 50 KB, peak is 3x average.

- Daily requests: `10,000,000 × 20 = 200,000,000`
- Average RPS: `200,000,000 / 86,400 ≈ 2,315`
- Peak RPS: `2,315 × 3 ≈ 6,900` (call it about 7,000)
- Average egress: `2,315 × 50 KB ≈ 116 MB/s ≈ 926 Mbps`
- Peak egress: about `2.8 Gbps`
- Instances: if one instance sustains 500 RPS within the latency SLO (an assumption you must measure), peak needs `7,000 / 500 = 14` instances. Allowing for the loss of one of three zones (about 150% of peak), provision roughly `14 × 1.5 = 21`.

The numbers now suggest what to ask next: can a CDN absorb a large share of that egress? Is the database ready for the read volume behind 7,000 RPS if the cache hit rate is 90%? (That leaves about 700 reads per second reaching the database.)

The value of the exercise is not the exact figures. It is discovering that, for example, bandwidth or database connections will break before CPU does.

## 26. Bottlenecks and the Laws of Scalability

A bottleneck is a component that limits the performance or throughput of the whole system.

### 26.1 Example

An application has eight application servers, one database, and a shared connection pool. Adding more application servers may not increase throughput if the database is already saturated. The database is the limiting component.

### 26.2 Amdahl's Law

Amdahl's Law gives the theoretical speedup when only part of a workload can be parallelized:

`S(N) = 1 / ((1 − P) + P / N)`

- `S(N)` = speedup with `N` processors.
- `P` = fraction of work that can be parallelized.
- `N` = number of processors or equivalent parallel resources.

If 80% of a workload is parallelizable (`P = 0.8`) and `N = 8`:

`S(8) = 1 / (0.2 + 0.8 / 8) = 1 / 0.3 ≈ 3.33`

An eightfold increase in resources yields only about 3.33x speedup (efficiency about 42%). As `N` grows without bound, speedup approaches `1 / (1 − P) = 5`. **The serial fraction sets the ceiling.**

### 26.3 Gustafson's Law

Amdahl's Law assumes a fixed problem size. In practice, bigger systems often handle **bigger problems** (more users, more data). Gustafson's Law models this scaled speedup:

`S(N) = N − (1 − P) × (N − 1)`

With `P = 0.8` and `N = 8`: `8 − 0.2 × 7 = 6.6`.

The two laws are complementary: Amdahl warns that a fixed job cannot be sped up indefinitely; Gustafson explains why large systems remain useful when the workload grows with the resources.

### 26.4 The Universal Scalability Law

Real systems often do worse than Amdahl predicts because of **coordination costs** (locks, cache coherence, distributed consensus, cross-node chatter). Neil Gunther's Universal Scalability Law models relative capacity `C(N)` as:

`C(N) = N / (1 + α(N − 1) + β N (N − 1))`

- `α` = contention (serialized portion, like Amdahl's serial fraction).
- `β` = coherency (pairwise coordination cost, which grows roughly with `N²`).

The `β` term makes throughput **peak and then decline** (retrograde scaling). With `α = 0.05` and `β = 0.001`:

| N | Relative capacity C(N) |
|---:|---:|
| 1 | 1.00 |
| 2 | 1.90 |
| 4 | 3.44 |
| 8 | 5.69 |
| 16 | 8.04 |
| 32 | 9.03 |
| 64 | 7.82 |

Capacity peaks near `N* = √((1 − α) / β) ≈ 31`, and adding nodes beyond that **reduces** throughput. The practical lessons:

- Contention and coordination, not hardware, set the ceiling.
- Past some point, more nodes make things worse.
- Reduce `α` and `β` by partitioning work, avoiding shared locks and global coordination, and keeping nodes independent.

You can fit `α` and `β` to measured throughput at several load levels to predict where scaling will stall.

### 26.5 Diminishing returns

Additional servers provide progressively smaller improvements when database writes remain centralized, network communication increases, lock contention rises, coordination overhead grows, or the workload cannot be distributed evenly.

### 26.6 The bottleneck-first approach

When a system struggles under load:

1. Measure the workload.
2. Identify the saturated resource.
3. Find the operation consuming it.
4. Optimize that operation.
5. Increase capacity if necessary.
6. Re-measure after the change.

Do not assume CPU is the bottleneck just because the application is slow. Every fix moves the bottleneck somewhere else; scalability work is the repeated cycle of finding the next one.

## 27. The CAP Theorem and Scalability

The CAP theorem describes a trade-off in distributed data systems during a network partition.

- **Consistency:** operations behave according to a specified single-copy (linearizable) consistency model.
- **Availability:** every request to a non-failing node receives a response, though not necessarily with the latest data.
- **Partition tolerance:** the system continues operating despite communication failures between nodes.

When a partition occurs, a distributed system cannot guarantee both strong consistency and availability for every request.

### CP-oriented behavior

The system may reject or delay some operations during a partition to preserve consistency.

### AP-oriented behavior

The system may continue responding during a partition, potentially returning stale or conflicting data that must be reconciled.

### Beyond CAP: latency in normal operation

CAP is only about behavior **during partitions**. In normal operation there is still a trade-off between **latency and consistency**, because stronger consistency requires more coordination, such as waiting for replicas. The PACELC formulation captures this: if there is a Partition, choose Availability or Consistency; Else, choose Latency or Consistency.

### Why this matters for scalability

Coordination limits scale (Section 26). Systems that relax consistency (eventual consistency, per-key ordering) can partition and scale with less coordination, and systems that need strict global consistency pay in latency, throughput, or availability.

Decide per operation:

- A product catalog may tolerate temporary stale reads.
- A financial ledger may need stronger consistency for critical operations.
- A shopping cart may use a conflict-resolution strategy that favors availability.

CAP does not mean every system permanently chooses two of three properties in all circumstances. See also [Reliability, Section 20.4](reliability.md#204-consistency-trade-offs).

## 28. Cost and Scalability Trade-Offs

A scalable architecture must balance performance, reliability, and cost.

### 28.1 Overprovisioning

Excess capacity protects against spikes but increases ongoing cost.

### 28.2 Underprovisioning

Insufficient capacity causes latency increases, failed requests, and queue buildup.

### 28.3 Autoscaling costs

Autoscaling reduces idle capacity, but frequent scaling adds provisioning overhead, cold starts, and fluctuating costs. A mixed approach is common: reserved or committed capacity for the predictable baseline, on-demand or autoscaled capacity for variable load, and interruptible (spot) capacity for fault-tolerant batch work.

### 28.4 Distributed architecture costs

More instances and services increase compute charges, network traffic (especially cross-zone and cross-region transfer), storage replication, logging and monitoring costs, operational overhead, and engineering effort.

### 28.5 Optimize cost per unit of work

Useful metrics:

- Cost per 1,000 requests.
- Cost per processed transaction.
- Cost per gigabyte stored.
- Cost per million messages processed.
- Cost per successful order.

These show whether the architecture becomes more or less efficient as traffic grows. Healthy scaling usually shows flat or **decreasing** unit cost; rising unit cost signals diminishing returns.

The cheapest infrastructure is not necessarily the cheapest system to operate if it causes frequent incidents or requires heavy manual work.

## 29. Designing a Scalable E-Commerce System

Consider an e-commerce platform that must grow from 10,000 daily users to millions.

### 29.1 Requirements

Functional:

- Browse products.
- Search the catalog.
- Add products to a cart.
- Place orders.
- Process payments.
- Track shipments.

Non-functional:

- Support growing traffic.
- Maintain acceptable latency.
- Protect transaction correctness.
- Handle traffic spikes.
- Scale background processing.
- Meet availability and recovery objectives.

### 29.2 High-level architecture

```text
                   Users
                     |
                     v
               CDN / DNS
                     |
                     v
               Load Balancer
                     |
                     v
               API Services
                     |
        +------------+------------+
        |            |            |
        v            v            v
     Catalog       Search       Orders
        |            |            |
        v            v            v
      Cache        Search DB    Order DB
                                  |
                                  v
                            Durable Queue
                                  |
                       +----------+----------+
                       |                     |
                       v                     v
                  Payment Worker       Notification Worker
```

This is a conceptual architecture. Real payment workflows require carefully defined transaction states and safe handling of uncertain outcomes (see [Reliability, Section 30](reliability.md#30-designing-a-reliable-e-commerce-system)).

### 29.3 Step 1: Scale the application tier

Run multiple stateless instances behind a load balancer. Scale out when traffic exceeds the current fleet's capacity.

### 29.4 Step 2: Cache read-heavy data

Cache product details, categories, and other frequently read data, and serve static assets from a CDN. Choose TTLs and invalidation policies by how quickly changes must become visible.

### 29.5 Step 3: Scale the database

First optimize queries and indexes. Then evaluate read replicas, connection pooling, partitioning, sharding, and separating analytical workloads. Choose based on actual database pressure and access patterns.

### 29.6 Step 4: Decouple background processing

Use durable messaging for work that does not need to finish synchronously. Scale consumers by backlog and processing latency, and ensure downstream dependencies can support the increased rate.

### 29.7 Step 5: Scale search independently

Search workloads may need a separate engine or indexing service, scaled by indexing rate, query rate, and data volume. This is Y-axis scaling in the scale cube.

### 29.8 Step 6: Protect the system during spikes

- Apply rate limits and admission control.
- Configure autoscaling and pre-scale for predictable events.
- Bound queue sizes or apply controlled overload behavior.
- Protect critical database and payment operations.
- Gracefully degrade non-essential features (recommendations, reviews).

Flash-sale inventory is a classic hot-key problem: many buyers contend for one row. Consider reserving stock in a fast store with atomic decrements, queueing purchase intents, and reconciling to the system of record, rather than letting every request lock the same database row.

### 29.9 Step 7: Measure success

Track:

- Request throughput.
- P95 and P99 latency.
- Successful checkout ratio.
- Database CPU and connections.
- Cache hit ratio.
- Queue depth and message age.
- Cost per successful order.
- Error-budget consumption.

Scalability is demonstrated when the system handles greater demand while continuing to meet its defined service objectives.

## 30. Designing a Scalable URL Shortener

A URL shortener converts a long URL into a short identifier that redirects users to the original destination.

```text
Long URL:
https://example.com/products/category/item/12345

Short URL:
https://short.example/abc123
```

### 30.1 Requirements

- Create short URLs.
- Redirect users to the original URLs.
- Support high read traffic.
- Store mappings durably.
- Prevent identifier collisions.
- Handle popular links receiving enormous traffic.

### 30.2 Estimation

Assumptions: 100 million new URLs per month, a 100:1 read-to-write ratio, 500 bytes per record, 5 years of retention, peak equal to 3x average.

- Write rate: `100,000,000 / 2,592,000 s ≈ 39` writes per second.
- Read rate: `39 × 100 ≈ 3,900` redirects per second on average, about `11,700` at peak.
- Records in 5 years: `100M × 12 × 5 = 6 billion`.
- Storage: `6 billion × 500 bytes = 3 TB` (before replication and indexes).
- Identifier space: Base62 with 6 characters gives `62⁶ ≈ 56.8 billion` combinations, which comfortably covers 6 billion URLs. Seven characters give about 3.5 trillion.

Conclusions: this is **read-dominated** (cache aggressively), the write rate is modest, and 3 TB fits a single large node or a few shards. The challenge is read latency and hot links, not raw write volume.

### 30.3 Architecture

```text
            Users
              |
              v
          Load Balancer
              |
              v
        Redirect Service
              |
              v
            Cache
              |
         Cache miss
              |
              v
          URL Database
```

A separate write path generates and stores short identifiers.

### 30.4 Read-heavy workload

Redirect requests vastly outnumber creations. Caching popular mappings reduces database reads, and because the mapping rarely changes, a long TTL is usually safe.

### 30.5 Identifier generation

| Strategy | Pros | Cons |
|---|---|---|
| Database sequence encoded in Base62 | Unique, compact | Central coordination; predictable IDs; sequence can become a bottleneck |
| Random ID with collision check | No coordination, unpredictable | Collision handling; extra read or conditional write |
| Pre-allocated ranges per node (a range allocator) | Low coordination, fast | Gaps if a node dies; needs an allocator service |
| Hash of the long URL (truncated) | Same URL gives same ID | Collisions need handling; one URL, one ID may be unwanted |
| Distributed ID generators (time-ordered IDs) | No central point | Longer IDs |

A range-allocation approach (each application node leases a block of IDs and hands them out locally) is a common way to avoid a coordination bottleneck on every write.

### 30.6 Redirect semantics

- `301` (permanent) redirects can be cached by browsers and intermediaries, which cuts load but means you lose visibility into repeat clicks and cannot easily change the destination.
- `302` or `307` (temporary) redirects are not cached the same way, so every click reaches you, which supports analytics and editable destinations at higher cost.

### 30.7 Hot keys

A link referenced by a major event may receive millions of requests. Mitigations:

- CDN caching when redirect and cache rules permit.
- Distributed cache with local in-process caching for hot entries.
- Request coalescing.
- Replicated read paths.
- Traffic monitoring and rate limiting.

### 30.8 Scaling the mapping store

If volume outgrows one node, shard by short ID (a hash of the key, using consistent hashing), so the redirect service can compute the owning shard directly from the identifier, with no scatter-gather. Since the access pattern is key-value, a key-value store fits well.

### 30.9 Reliability considerations

- Durable URL mappings and backups.
- Cache invalidation (for deleted or disabled links).
- Safe redirect validation (block malicious destinations).
- Abuse prevention and rate limits on creation.
- Monitoring of popular links.
- Recovery procedures and capacity testing.

This example shows how estimation, caching, horizontal scaling, and appropriate data modeling work together for a read-heavy service.

## 31. Common Scalability Mistakes

1. **Adding servers without identifying the bottleneck.** The real limit may be the database, network, or an external dependency.
2. **Scaling the application but not the database.** More application instances can increase database contention.
3. **Ignoring connection limits.** A larger application fleet can exhaust downstream connection capacity.
4. **Using a poor shard or partition key.** Uneven traffic creates hotspots.
5. **Assuming caching is always safe.** Stale data can violate business requirements, and a cold cache can overload the database.
6. **Scaling every microservice independently without coordination.** Downstream systems can be overwhelmed.
7. **Ignoring queue growth.** A queue can accumulate work faster than consumers process it, and drain time grows steeply as spare capacity shrinks.
8. **Relying only on average latency.** Tail latency may become unacceptable under load.
9. **Ignoring cold starts and scaling delay.** Reactive autoscaling may respond too late to sudden spikes.
10. **Using unlimited retries.** Retries can amplify overload.
11. **Ignoring state management.** Local session state prevents safe horizontal scaling.
12. **Overengineering too early.** Premature sharding and microservices add complexity before it is needed.
13. **Ignoring cost efficiency.** More infrastructure does not automatically mean a better design.
14. **Failing to load test.** The real capacity limit may be far lower than expected.
15. **Ignoring failure scenarios during capacity planning.** The system may lack capacity when a zone or instance fails.
16. **Assuming linear scaling.** Contention and coordination flatten, then reverse, gains (USL).
17. **Autoscaling on the wrong signal.** CPU is a poor trigger for queue consumers or I/O-bound services.
18. **Setting no autoscaling maximum.** A runaway scale-out can exhaust quotas, budgets, or downstream capacity.
19. **Sticky sessions as a substitute for stateless design.** They create imbalance and lose state on failure.
20. **Using modulo hashing for a changing node set.** Adding a node reshuffles most keys.
21. **Unbounded fan-out.** One request multiplying into many downstream calls multiplies load and tail latency.
22. **Skipping back-of-the-envelope estimates.** Designing without numbers hides which resource breaks first.

## 32. Practical SRE Scalability Workflow

Use this workflow when investigating capacity or performance problems in production.

### Step 1: Understand the workload

Collect average and peak RPS, request distribution, data volume, read/write ratio, latency, traffic seasonality, and background processing rate.

### Step 2: Establish service objectives

Define measurable targets for request success, P95 and P99 latency, queue processing delay, data freshness, and availability.

### Step 3: Identify bottlenecks

Use metrics, logs, traces, and profiling to examine CPU and memory saturation, database query performance, connection pools, network latency, cache performance, queue depth, and downstream dependencies. The USE method (Utilization, Saturation, Errors) applied to each resource finds the saturated one quickly.

### Step 4: Optimize before adding capacity

Consider query optimization, indexes, caching, fewer network calls, batch processing, better algorithms, efficient serialization, and connection reuse.

### Step 5: Choose a scaling strategy

Select the **least complex** approach that meets requirements:

- Vertical scaling.
- Horizontal scaling.
- Replication.
- Partitioning or sharding.
- Asynchronous processing.
- Autoscaling.
- Workload isolation.

### Step 6: Validate capacity

Perform load, stress, and soak testing. Test expected traffic and realistic spikes, including failed components and retries. Find the maximum sustainable throughput at the SLO. Use open-model load generation ([Latency, Section 8.4](latency.md#84-coordinated-omission)).

### Step 7: Automate scaling and recovery

Configure scaling policies with sensible minimums and maximums, health checks, graceful shutdown, and overload protection (rate limits, load shedding).

### Step 8: Monitor after deployment

Compare behavior before and after the change. Verify that the bottleneck moved or disappeared and that the system still meets its SLOs.

### Step 9: Review cost

Measure additional capacity and operating cost relative to the workload handled, using cost per unit of work.

Scalability work succeeds when it improves the system's ability to serve growth **efficiently**, not merely when it increases the number of servers.

## 33. System Design Interview Questions

### Beginner

**Q1. What is scalability?**

The ability of a system to handle increasing workload by adding resources while continuing to meet its requirements.

**Q2. What is the difference between vertical and horizontal scaling?**

Vertical scaling increases the resources of an existing machine. Horizontal scaling adds more machines or instances.

**Q3. What is a load balancer?**

A component that distributes incoming requests across eligible backend targets.

**Q4. Why is stateless architecture useful for scaling?**

Any healthy instance can process a request without depending on locally retained session state, making it easier to add and remove instances.

**Q5. What is a bottleneck?**

A resource or operation that limits overall throughput or performance.

**Q6. What is the difference between scalability and elasticity?**

Scalability is the ability to handle growth by adding resources. Elasticity is automatically adding and removing capacity to track demand.

### Intermediate

**Q7. How does caching improve scalability?**

It serves repeated requests without accessing the original data source, reducing backend load and often latency. Raising the hit ratio from 90% to 99% cuts backend load tenfold.

**Q8. What is the difference between replication and sharding?**

Replication maintains multiple copies of data (scaling reads and availability). Sharding distributes different subsets of data across nodes (scaling data volume and writes).

**Q9. Why can adding application servers make performance worse?**

The extra servers may increase pressure on a shared database, cache, or external dependency, for example by exhausting connection limits.

**Q10. How does a message queue improve scalability?**

It decouples producers from consumers, absorbs bursts, and lets background processing scale independently.

**Q11. What is a hot partition?**

A partition that receives disproportionate traffic, limiting throughput even when other partitions have spare capacity.

**Q12. What is the difference between an L4 and an L7 load balancer?**

L4 balances TCP/UDP connections without inspecting content; L7 understands HTTP and can route per request by path, host, or header. L7 is better for long-lived multiplexed connections, which L4 can distribute unevenly.

**Q13. Compare token bucket and fixed-window rate limiting.**

A token bucket allows controlled bursts up to its capacity while enforcing a sustained rate. A fixed window is simpler but permits up to roughly double the limit around a window boundary.

### Advanced

**Q14. How would you scale a system from 1,000 to 100,000 RPS?**

Measure the workload and bottlenecks, optimize the request path, scale stateless compute, add caching and appropriate database strategies, decouple asynchronous work, validate downstream capacity, add rate limiting and headroom for failures, and load test against latency and reliability targets.

**Q15. When would you introduce database sharding?**

When query optimization, vertical scaling, caching, and replicas cannot meet capacity or data-size requirements, or when growth clearly justifies distribution. Plan the shard key and resharding approach up front.

**Q16. Why is consistent hashing preferred over `hash % N` for caches?**

With modulo, changing `N` remaps most keys (about 80% going from 4 to 5 nodes), causing a flood of cache misses. Consistent hashing moves only about `1 / (N + 1)` of keys.

**Q17. How do you scale a read-heavy system?**

Use indexes, caching at multiple layers, read replicas, CDN delivery, denormalized read models (CQRS) where justified, and data partitioning if necessary.

**Q18. How do you scale a write-heavy system?**

Analyze write contention and transaction requirements first. Consider batching, partitioning with a well-distributed key, asynchronous processing, append-oriented storage, workload isolation, and distributed data stores. Replicas do not scale writes.

**Q19. What happens if a queue's arrival rate exceeds its processing rate?**

The backlog grows, delay increases, and the system may hit retention, storage, or deadline limits. Increase processing capacity, reduce incoming work, or apply backpressure. Recovery time is `backlog / (capacity − arrival rate)`, so it rises sharply as the margin shrinks.

**Q20. How does Kubernetes scale an application?**

The HPA changes Pod replicas (CPU targets are relative to requests), KEDA or custom metrics can scale on queue depth, and the Cluster Autoscaler adjusts node capacity. Resource requests, readiness, limits, and downstream capacity still matter.

**Q21. What are Amdahl's Law and the Universal Scalability Law?**

Amdahl's Law shows the serial fraction caps speedup (`1 / (1 − P)`). The USL adds a coordination term so throughput can peak and then decline as nodes are added, explaining why unbounded scale-out eventually backfires.

**Q22. Fan-out on write or on read for a social feed?**

Fan-out on write gives fast reads but expensive writes (and a problem for accounts with millions of followers); fan-out on read is the opposite. Large systems usually mix them: precompute for typical users and merge popular accounts at read time.

### Scenario-based questions

**Q23. Your CPU utilization is only 30%, but the application is slow. What do you investigate?**

Database latency, network calls, connection pools, locks, queueing, external dependencies, memory pressure, thread contention, and traces. CPU alone cannot identify every bottleneck.

**Q24. Your application handles twice as much traffic after scaling, but the database is failing. What do you do?**

Protect the database with admission control and concurrency limits. Check connection counts, slow queries, and lock contention; optimize data access; then evaluate caching, replicas, partitioning, or sharding based on the workload. Cap autoscaling at what the database can handle.

**Q25. How would you handle a sudden tenfold traffic spike?**

Pre-scale if predictable; otherwise rely on autoscaling with headroom, caching, CDN, bounded queues, rate limits, and graceful degradation. Verify dependencies and quotas can absorb the load, because autoscaling takes minutes to take effect.

**Q26. How would you scale a global application?**

Use regional compute, CDN caching, data locality, geographic routing, and replication, with tested regional recovery. Data consistency and residency requirements determine what is feasible.

**Q27. How do you prove a system is scalable?**

Show through representative load tests and production measurements that increased workload is handled while meeting latency, error-rate, correctness, and cost objectives, and that scaling efficiency stays acceptable as resources are added.

**Q28. Estimate the capacity for a service with 10 million daily users making 20 requests each.**

`200M requests per day / 86,400 s ≈ 2,300 RPS average`, roughly 7,000 RPS at a 3x peak. Divide by measured per-instance capacity, add headroom for failure and scaling delay, then check bandwidth, database load behind the cache hit rate, and connection limits.

## 34. Scalability Cheat Sheet

| Concept | Meaning |
|---|---|
| Scalability | Ability to accommodate increasing workload |
| Elasticity | Capacity follows demand up and down automatically |
| Vertical scaling | Increase resources on one machine |
| Horizontal scaling | Add machines or instances |
| Scale cube | X (clone), Y (split by function), Z (partition data) |
| Load balancing | Distribute requests across targets (L4 by connection, L7 by request) |
| Stateless architecture | Instances do not depend on local retained session state |
| Caching | Reuse data to reduce repeated work |
| Hit ratio | 90% to 99% cuts backend load 10x |
| Replication | Multiple copies of data (scales reads, not writes) |
| Sharding | Subsets of data across nodes (scales data and writes) |
| Consistent hashing | Adding a node moves about `1/(N+1)` of keys |
| Partitioning | Divide a dataset into smaller sections |
| Message queue | Buffer and distribute asynchronous work |
| Backlog drain time | `backlog / (capacity − arrival rate)` |
| Backpressure | Control incoming work when capacity is constrained |
| Rate limiting | Token bucket: burst `b`, sustained rate `r` |
| Autoscaling | Adjust resources based on demand or policy |
| HPA formula | `ceil(replicas × current / target)` |
| HPA / VPA / CA | Pod replicas / Pod resource requests / cluster nodes |
| RPS | Requests per second |
| Throughput | Work completed per unit of time |
| Little's Law | `L = λ × W` |
| P95 / P99 | Tail-latency percentiles |
| Bottleneck | Component limiting overall system performance |
| Amdahl's Law | Speedup capped at `1 / (1 − P)` |
| Gustafson's Law | Scaled speedup when problem size grows with resources |
| USL | Contention + coherency; throughput can decline past a peak |
| Scaling efficiency | `Throughput(N) / (N × Throughput(1))` |
| Hot partition | Partition receiving disproportionate traffic |
| Capacity headroom | Available capacity beyond current demand |
| CAP / PACELC | Partition: C vs A; otherwise: latency vs consistency |

## 35. Key Takeaways

1. Scalability is the ability to accommodate growth while meeting service requirements, and elasticity is making that growth (and shrinkage) automatic.
2. Vertical scaling increases one machine's capacity; horizontal scaling adds machines; the scale cube adds functional and data axes.
3. Stateless application tiers are generally easier to scale horizontally.
4. Load balancing distributes traffic but cannot fix an overloaded shared dependency.
5. Caching can reduce backend demand dramatically, but the cache tier needs its own capacity, partitioning, and failure planning.
6. Replication scales reads and availability; sharding scales data and writes. Choose partition keys and hashing schemes that avoid hotspots and mass remapping.
7. Message queues decouple processing rates but require backpressure and backlog monitoring, and recovery time depends on spare capacity.
8. Rate limiting and admission control keep a saturated system fast for admitted work.
9. Autoscaling must account for provisioning delay, headroom, quotas, and downstream capacity; set sensible maximums.
10. Kubernetes HPA, VPA, and Cluster Autoscaler address different scaling layers, and CPU-based scaling is often the wrong signal for queue consumers.
11. Amdahl's Law, Gustafson's Law, and the Universal Scalability Law explain why serial work and coordination cap or even reverse the gains from more resources.
12. Back-of-the-envelope estimation tells you which resource breaks first, before you build.
13. Scalability must be validated through realistic load tests and production measurements against SLOs.
14. The best architecture balances capacity, reliability, latency, cost, and operational complexity.

**The central principle:** A scalable system does not simply use more resources. It distributes work efficiently, removes bottlenecks, minimizes coordination, protects shared dependencies, and grows capacity in line with actual demand.

## 36. References

1. AWS Well-Architected Framework: [Performance Efficiency Pillar](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/welcome.html)
2. AWS Well-Architected Framework: [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
3. Amazon EC2 Auto Scaling: [What Is Amazon EC2 Auto Scaling?](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
4. Amazon CloudFront: [What Is Amazon CloudFront?](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
5. Amazon DynamoDB: [Partition Key Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)
6. Kubernetes Documentation: [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
7. Kubernetes Documentation: [Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
8. Kubernetes Documentation: [Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
9. KEDA: [Kubernetes Event-driven Autoscaling](https://keda.sh/)
10. Google SRE Book: [Handling Overload](https://sre.google/sre-book/handling-overload/)
11. Martin Kleppmann: *Designing Data-Intensive Applications*, O'Reilly Media.
12. Martin Abbott and Michael Fisher: *The Art of Scalability* (the scale cube).
13. Neil Gunther: *Guerrilla Capacity Planning* (the Universal Scalability Law).
14. David Karger et al.: *Consistent Hashing and Random Trees* (STOC 1997).

---

*Last updated: October 2026*

*Suggested repository path: `01-fundamentals/scalability.md`*
