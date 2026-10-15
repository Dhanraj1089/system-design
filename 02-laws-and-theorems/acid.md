# ACID Properties in System Design

## 1. Introduction

ACID is a set of four properties that ensure reliable database transactions:

- **A — Atomicity**
- **C — Consistency**
- **I — Isolation**
- **D — Durability**

These properties help databases maintain correctness even when transactions fail, multiple users access data concurrently, or systems experience crashes.

ACID is particularly important in systems that handle financial transactions, inventory, payments, banking, order processing, and other operations where incorrect data can have serious consequences.

### Why ACID matters

Consider an online banking system where a user transfers ₹1,000 from Account A to Account B.

The system must ensure that:

1. ₹1,000 is deducted from Account A.
2. ₹1,000 is credited to Account B.
3. Both operations succeed together, or neither operation takes effect.
4. Concurrent transactions do not corrupt account balances.
5. Once the transaction is committed, the transfer survives a database crash.

ACID provides the foundation for achieving these guarantees.

---

## 2. Atomicity — All or Nothing

### Definition

Atomicity guarantees that a transaction is treated as one indivisible unit of work. Either all its operations succeed, or none of them take effect.

### Example: Bank transfer

Initial balances:

- Account A: ₹5,000
- Account B: ₹2,000

Transfer amount: ₹1,000.

The transaction performs two operations:

```sql
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 'A';

UPDATE accounts
SET balance = balance + 1000
WHERE account_id = 'B';

COMMIT;
```

If both updates succeed, the transaction commits.

Final balances:

- Account A: ₹4,000
- Account B: ₹3,000

If the database fails after deducting ₹1,000 but before crediting Account B, the transaction must be rolled back so that the deduction does not remain as a partial transfer.

### How atomicity works

Databases commonly use transaction logs, undo information, and recovery mechanisms to reverse incomplete transactions.

The precise implementation depends on the database engine.

### What happens without atomicity?

Account A might lose ₹1,000 while Account B never receives it.

This produces an inconsistent financial outcome.

### Real-world applications

- Bank transfers
- Order creation and payment recording
- Inventory updates
- Multi-row database updates
- Updating related records together

### System design takeaway

Atomicity protects a transaction from partial completion within its transactional boundary.

However, a transaction spanning independent services usually cannot rely on one ordinary database transaction. Distributed workflows may require a Saga pattern, compensating actions, or another distributed transaction mechanism.

---

## 3. Consistency — Preserve Valid Data

### Definition

Consistency guarantees that a successfully committed transaction takes the database from one valid state to another, preserving applicable integrity constraints and business rules.

Consistency here means **data correctness according to defined rules**, not necessarily that every replica immediately contains the latest data.

### Example: Account balance

Suppose a banking system requires that an account balance never be negative.

A transaction attempts to withdraw ₹6,000 from an account containing ₹5,000.

The database or application should reject the withdrawal.

A database constraint could enforce the rule:

```sql
ALTER TABLE accounts
ADD CONSTRAINT non_negative_balance
CHECK (balance >= 0);
```

The exact syntax and enforcement behavior can vary by database.

A safer withdrawal operation should also account for concurrent requests:

```sql
UPDATE accounts
SET balance = balance - 6000
WHERE account_id = 'A'
  AND balance >= 6000;
```

The application must verify that the expected row was updated before reporting success.

### Other consistency rules

- An order must reference a valid customer.
- A unique email address must not be duplicated.
- A payment amount must be greater than zero.
- Inventory must not fall below the permitted limit.
- A transaction must preserve accounting rules.

### How consistency is maintained

Common mechanisms include:

- Primary keys and foreign keys
- Unique constraints
- Check constraints
- Transactions
- Validation logic
- Appropriate concurrency control

### What happens without consistency?

The database may contain invalid relationships, duplicate records, negative balances, or values that violate business rules.

### System design takeaway

Consistency depends on correctly defined invariants and the mechanisms enforcing them. ACID does not automatically guarantee that every business rule is implemented correctly.

---

## 4. Isolation — Control Concurrent Transactions

### Definition

Isolation determines how concurrently executing transactions interact with each other.

The goal is to prevent concurrent operations from producing incorrect results while balancing correctness, throughput, and latency.

### Example: Two users buy the last product

An online store has one unit remaining in inventory.

Two users attempt to purchase it simultaneously.

Without appropriate concurrency control:

1. Transaction A reads stock = 1.
2. Transaction B reads stock = 1.
3. Transaction A confirms a purchase.
4. Transaction B confirms a purchase.
5. Two orders are accepted for one available unit.

This is an overselling problem.

### One solution: Conditional update

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 'P100'
  AND quantity > 0;
```

The application should confirm that exactly one row was updated before accepting the purchase.

If another transaction has already consumed the final unit, the second update affects zero rows and the purchase should not be accepted as available.

Other approaches include row-level locking, optimistic concurrency control, and serializable transactions.

### Isolation levels

| Isolation level | General behavior | Potential issues |
|---|---|---|
| Read Uncommitted | May allow reading uncommitted changes | Dirty reads and other anomalies |
| Read Committed | Reads only committed data | Non-repeatable reads may occur |
| Repeatable Read | Provides stronger repeatability for reads | Some anomalies may still occur, depending on the database |
| Serializable | Provides behavior equivalent to some serial execution of transactions | May reduce throughput or cause retries |

Exact behavior differs across database engines. For example, PostgreSQL and MySQL/InnoDB do not implement every isolation level identically.

### Common concurrency anomalies

**Dirty read:** A transaction reads changes that another transaction has not committed.

**Non-repeatable read:** A transaction reads a row twice and gets different committed values.

**Phantom read:** Repeating a query returns a different set of rows because another transaction changed matching records.

**Lost update:** One transaction overwrites another transaction's update without properly accounting for it.

**Write skew:** Concurrent transactions independently satisfy a condition based on the same initial state, but their combined changes violate a business invariant.

### Isolation versus performance

Stronger isolation can improve correctness but may increase locking, contention, aborts, retries, and latency.

System designers choose isolation based on business requirements rather than automatically selecting the strongest level for every workload.

### Real-world applications

- Inventory reservation
- Bank balance updates
- Ticket booking
- Seat reservation
- Concurrent order processing
- Account limit enforcement

### System design takeaway

Isolation is a major factor in preventing race conditions and protecting invariants under concurrent access.

---

## 5. Durability — Committed Data Survives Failures

### Definition

Durability means that once a transaction is successfully committed, its changes survive subsequent failures covered by the database's durability guarantees.

### Example

A customer completes a ₹2,000 payment.

The database confirms that the payment transaction has committed.

Immediately afterward, the database server loses power.

When the database recovers, the committed payment record should still exist.

### How durability works

Databases commonly use mechanisms such as:

- Write-ahead logging (WAL)
- Transaction logs
- Persistent storage
- Crash recovery
- Replication, depending on the required failure guarantees

With write-ahead logging, the database records the necessary log information before acknowledging a commit under the configured durability policy.

The database can then recover committed changes after a crash.

### Durability is not the same as backup

| Mechanism | Primary purpose |
|---|---|
| Transaction log | Supports recovery and transaction guarantees |
| Persistent storage | Retains data across supported failures |
| Replication | Maintains copies across nodes or locations |
| Backup | Provides recovery from deletion, corruption, or other historical failures |
| Point-in-time recovery | Restores a database to a supported earlier point |

Replication alone is not a substitute for backups. Accidental deletion or logical corruption can propagate to replicas.

### Synchronous versus asynchronous replication

**Synchronous replication** waits for acknowledgments from specified replicas before confirming an operation, depending on the database configuration.

**Asynchronous replication** can acknowledge a write before replicas have received it, leaving a possible replication lag or data-loss window during certain failures.

A database can provide local durability without synchronous replication to another Region. Therefore, understand the precise failure scope of the durability guarantee.

### System design takeaway

Durability protects committed transactions, but recovery objectives, replication configuration, storage guarantees, and backups determine how well a system recovers from broader failures.

---

## 6. ACID Summary

| Property | Main question | Example failure it prevents |
|---|---|---|
| Atomicity | Did all operations succeed together? | Partial bank transfer |
| Consistency | Are database rules preserved? | Negative balance |
| Isolation | Are concurrent transactions handled safely? | Overselling inventory |
| Durability | Does committed data survive a supported failure? | Lost committed payment after a crash |

A simple way to remember ACID:

- **Atomicity:** All or nothing.
- **Consistency:** Valid state before and after.
- **Isolation:** Safe concurrent execution.
- **Durability:** Committed means persistent under the configured guarantees.

---

## 7. ACID in System Architecture

ACID is a transaction-level concept, but its implications extend to the overall system architecture.

### Monolithic application

A monolithic application using one relational database can often perform related changes in a single database transaction.

Example: creating an order and reserving inventory in the same database.

### Microservices architecture

In a microservices architecture, each service may own its own database.

For example:

- Order Service
- Payment Service
- Inventory Service
- Shipping Service

An order might require changes in several independently managed databases.

A normal SQL transaction cannot automatically provide atomicity across all those databases.

Possible approaches include:

- Saga pattern
- Transactional outbox
- Idempotent event processing
- Distributed transactions where supported and appropriate
- Compensating actions and reconciliation

A Saga breaks a distributed business process into local transactions. If a later step fails, the workflow may execute compensating actions to address earlier successful steps.

A compensation is not necessarily a true rollback. For example, a refund is a new business transaction, not erasure of the original payment.

---

## 8. ACID vs. BASE

BASE is commonly used to describe availability-oriented distributed data systems, particularly those that allow eventual consistency.

| Dimension | ACID | BASE |
|---|---|---|
| Main focus | Transaction correctness and integrity | Availability and flexible distributed operation |
| Consistency | Database constraints and transactional guarantees | Often eventual convergence |
| Transactions | Strong transactional guarantees within the supported boundary | May rely on local transactions and asynchronous workflows |
| Failure handling | Rollback, recovery, isolation controls | Retries, reconciliation, replication and convergence |
| Typical use cases | Financial ledgers, relational transactions | Distributed catalogs, feeds and some high-scale data platforms |

BASE is a conceptual model, not a universal database specification. Likewise, ACID and BASE are not absolute opposites: distributed systems can provide ACID transactions while also using replication and availability techniques.

### Example

A bank ledger may require strict transactional correctness for a transfer.

A product catalog may tolerate a short delay before an updated product description appears on every replica.

The right choice depends on the data and business invariants.

---

## 9. ACID and CAP Theorem

The CAP theorem concerns consistency, availability, and partition tolerance in a distributed system during a network partition.

ACID concerns transaction properties.

They describe different aspects of system behavior and should not be treated as interchangeable concepts.

For example, a distributed database may provide ACID transactions while offering different consistency and availability behavior across Regions, depending on its configuration and architecture.

During a network partition, a distributed design that prioritizes linearizable consistency may need to reject or delay some operations rather than serve potentially conflicting writes.

### Exam and interview distinction

- ACID: How reliable database transactions behave.
- CAP: Trade-offs involving consistency and availability during a network partition.
- BASE: A broad approach often associated with highly available, eventually consistent systems.
- Eventual consistency: Replicas converge if updates stop and the system continues making progress under the model's assumptions.

---

## 10. ACID in Real-World Systems

| System | Why ACID matters | Important consideration |
|---|---|---|
| Banking | Protects balance and ledger invariants | Concurrent transfers and isolation |
| E-commerce | Protects order and inventory updates | Distributed order workflows |
| Payment processing | Protects payment records | Idempotency and external payment-provider outcomes |
| Ticket booking | Prevents double booking | Atomic reservation or conditional writes |
| Healthcare records | Protects data integrity | Authorization, auditing and recovery |
| Stock trading | Protects order and ledger correctness | Concurrency, ordering and business-specific guarantees |

ACID is not a substitute for idempotency. A payment API may receive the same request multiple times because a client retries after a timeout. An idempotency key helps ensure that repeated requests do not create duplicate business operations.

Similarly, a transaction can commit successfully even if the client never receives the response. The client must not automatically assume that a timeout means the transaction failed.

---

## 11. System Design Interview Questions

### Q1. Why is atomicity important in a bank transfer?

It prevents a transfer from leaving only one side of the balance update applied within the transaction.

### Q2. Does ACID guarantee that all replicas have the latest data immediately?

No. ACID transaction guarantees and cross-replica consistency are related but distinct. Replication topology and consistency settings matter.

### Q3. Is stronger isolation always better?

No. Stronger isolation may increase contention, latency, and retries. Choose the weakest level that correctly preserves the required invariants, subject to the database's semantics.

### Q4. Does a successful database commit mean data can never be lost?

No system is immune to every possible failure. Durability depends on the database's documented guarantees, configuration, storage, replication and failure scope.

### Q5. How can two users avoid buying the last item simultaneously?

Use a suitable concurrency-control mechanism, such as an atomic conditional update, appropriate locking, or serializable transaction handling.

### Q6. Can ACID span multiple microservices automatically?

Not through a normal local database transaction. Distributed transactions or workflow patterns such as Sagas may be needed.

### Q7. How is ACID different from idempotency?

ACID governs transaction behavior. Idempotency makes repeated execution of the same logical request safe against unintended duplicate effects.

### Q8. How do ACID and eventual consistency differ?

ACID defines transactional guarantees. Eventual consistency describes how replicas can converge over time. A system can use ACID locally and eventual consistency for replication or asynchronous workflows.

---

## 12. Final Takeaways

When designing a reliable system, ask four questions:

1. **Atomicity:** Can a partial operation leave the data in an invalid state?
2. **Consistency:** Which business rules must always hold?
3. **Isolation:** What happens when multiple requests update the same data concurrently?
4. **Durability:** What failures must committed data survive?

Use these questions to guide database selection, transaction boundaries, concurrency control, replication, recovery planning, and distributed workflow design.

ACID is a foundation for reliable transactional systems, but end-to-end reliability also requires idempotency, observability, fault handling, backups, security, and appropriate distributed-system patterns.
