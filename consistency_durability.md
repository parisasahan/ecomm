# Data Consistency and Durability for E-commerce Azure Solution

This document details the data consistency and durability aspects for key Azure services utilized in the e-commerce platform.

## 1. Data Consistency

Data consistency ensures that data is valid, accurate, and reliable according to defined rules and expectations, especially in concurrent operations or distributed systems.

### a. Azure SQL Database

*   **Strong Consistency (ACID Properties):** Azure SQL Database, as a relational database management system (RDBMS), provides strong consistency guarantees for all transactions through ACID properties:
    *   **Atomicity:** Ensures that all operations within a transaction are treated as a single, indivisible unit. Either all operations succeed, or none are applied (the transaction is rolled back).
    *   **Consistency:** Guarantees that a transaction transitions the database from one valid state to another. All defined rules, such as constraints, triggers, and cascades, must be satisfied.
    *   **Isolation:** Ensures that concurrently executing transactions produce the same outcome as if they were executed sequentially. Azure SQL Database offers various isolation levels (e.g., Read Committed, Repeatable Read, Serializable) to manage the trade-off between concurrency and data phenomena like dirty reads or phantom reads.
    *   **Durability:** Ensures that once a transaction is committed, its changes are permanent and survive system failures (e.g., power outages, crashes). (Detailed further in the Durability section).

*   **Usage in E-commerce for Critical Operations:**
    *   **Order Placement:** Operations like creating an order, adding items, updating inventory, and recording payment status are wrapped in a database transaction. This ensures that an order is fully processed and all related data is consistent, or the entire operation fails, preventing partial order states.
    *   **Inventory Updates:** Decrementing stock upon sale or incrementing upon restock are performed within transactions. This ensures atomic updates to stock levels, preventing overselling or data discrepancies.

### b. Azure Cosmos DB

Azure Cosmos DB offers five distinct consistency levels, allowing developers to choose the optimal balance between consistency, availability, latency, and throughput for their specific needs.

*   **Five Consistency Levels:**
    1.  **Strong:** Provides linearizability. Reads are guaranteed to return the most recent committed version of an item. Offers the highest data consistency but can introduce higher latency, especially in globally distributed scenarios, as reads must be confirmed by a quorum of replicas.
    2.  **Bounded Staleness:** Reads might lag behind writes by a configurable time window (K) or a number of versions (T). Guarantees that reads are not stale beyond K prefixes or T versions. Offers a good balance between consistency and performance.
    3.  **Session:** Guarantees monotonic reads, monotonic writes, read-your-own-writes, and consistent-prefix for a single client session. This is the most widely used level, providing intuitive consistency for user-centric applications.
    4.  **Consistent Prefix:** Reads are guaranteed to return a prefix of all writes, with no gaps. Updates are seen in order, but not necessarily the most recent version immediately.
    5.  **Eventual:** No ordering guarantee for reads. With no further writes, replicas eventually converge, and reads will return all committed values. Offers the lowest latency and highest throughput but the weakest consistency.

*   **Recommended Default Consistency Levels for E-commerce Data:**
    *   **Product Catalog:**
        *   **Default Level:** `Session` or `Eventual`.
        *   **Justification:** Product data is primarily read-heavy. While data freshness is important, minor delays (seconds to a few minutes) in global propagation of updates are often acceptable. `Session` ensures an administrator updating product details sees their changes. `Eventual` provides the best read performance for geographically distributed users. Critical updates (like price changes) can temporarily request stronger consistency for specific operations if needed.
    *   **Customer Profiles (Extended - preferences, non-critical data):**
        *   **Default Level:** `Session`.
        *   **Justification:** Users expect to see their own profile changes immediately (read-your-own-writes), which `Session` consistency provides. For other users or services accessing this data, slight staleness is generally acceptable.
    *   **Product Reviews & Ratings:**
        *   **Default Level:** `Session`.
        *   **Justification:** When a user submits a review, `Session` consistency ensures they see their submission. For others viewing reviews, eventual consistency is typically acceptable. `Consistent Prefix` can also be an option if the order of reviews is important and some staleness is okay.
    *   **User Behavior Data (for personalization):**
        *   **Default Level:** `Eventual` or `Session`.
        *   **Justification:** This data (clicks, views) is often write-intensive. `Eventual` consistency offers the highest write throughput and lowest latency for telemetry ingestion. If this data is used for immediate in-session personalization, `Session` consistency ensures the current session's actions can influence recommendations within that same session.

### c. Strategies for Multi-Service Consistency

Maintaining consistency when a single business operation spans multiple distinct data stores (e.g., Azure SQL DB for orders, Azure Cosmos DB for user profiles) is a significant challenge in distributed systems.

*   **Challenges:** Traditional distributed transaction coordinators (like those using Two-Phase Commit) are often complex to implement, have performance implications, and may not be supported across heterogeneous database technologies.
*   **Common Patterns:**
    *   **Two-Phase Commit (2PC):** A protocol ensuring atomicity across distributed participants. While providing strong consistency, it's often avoided in microservices due to its blocking nature and impact on availability.
    *   **Saga Pattern (Eventual Consistency):** A sequence of local transactions. Each service performs its own transaction and then publishes an event or message that triggers the next local transaction in the chain. If any step fails, compensating transactions are executed to undo preceding work, effectively rolling back the business operation. This pattern leads to eventual consistency and is generally preferred for its resilience and scalability in distributed environments.
    *   **Domain-Event-Driven Design:** Services react to domain events published by other services. For example, an "OrderPlaced" event from the Order service (SQL DB) can be consumed by a Loyalty service (Cosmos DB) to update points.
*   **Recommendation:** Prioritize asynchronous, event-driven approaches (like Saga or domain events) that achieve eventual consistency for operations spanning multiple services. Carefully design service boundaries and domain models to minimize the need for distributed transactions.

### d. Azure Cache for Redis

*   **Role in Performance:** Azure Cache for Redis is an in-memory data store used to cache frequently accessed data, significantly reducing read latency and the load on backend persistent databases.
*   **Consistency Considerations (Cache Invalidation):** Since data in Redis is typically a replica of data from a source of truth (e.g., SQL Database, Cosmos DB), consistency revolves around ensuring the cached data is not overly stale.
    *   **Strategies:**
        *   **Time-To-Live (TTL):** Automatically expire cache entries after a defined period.
        *   **Write-Through Caching:** Application writes to both cache and persistent store.
        *   **Cache-Aside Pattern:** Application attempts to read from cache; on a miss, reads from persistent store and populates the cache.
        *   **Explicit Invalidation/Update:** When the source data changes in the persistent store, the application logic (often triggered by events) explicitly invalidates or updates the corresponding entry in Redis. This is crucial for maintaining reasonable data freshness.

## 2. Data Durability

Data durability ensures that once data is committed, it is permanently stored and protected against loss from hardware failures, system crashes, or other physical disruptions.

### a. Azure SQL Database

*   **Ensuring Durability:**
    *   **Transaction Logging:** All database changes are first recorded in a transaction log before being written to data files. This log is essential for recovery.
    *   **Data Replication:**
        *   **Locally-Redundant Storage (LRS):** Synchronously replicates data to three copies within a single datacenter in the primary region.
        *   **Zone-Redundant Storage (ZRS) (for General Purpose, Business Critical, Hyperscale tiers):** Synchronously replicates data to three copies across different Azure Availability Zones within the primary region.
    *   **Commit Acknowledgment:** A transaction is considered committed (and thus durable) only after its log records are secured on durable storage.
*   **HA/DR Reference:** The replication mechanisms (LRS, ZRS) and geo-replication features (for failover groups) detailed in the `ha_dr_strategy.md` are fundamental to Azure SQL Database's ability to ensure data durability against various failure scenarios, from disk failures to zonal or even regional outages.

### b. Azure Cosmos DB

*   **Ensuring Durability:**
    *   **Local Replication (within a region):** Data is synchronously replicated across multiple fault domains and upgrade domains (a minimum of 3 replicas, 4 for accounts with multiple writable regions) within a region. A write operation is acknowledged only after it has been durably committed by a quorum of replicas.
    *   **Zone Redundancy (if enabled):** Replicas are distributed across different Availability Zones, providing durability against entire datacenter/zonal failures within a region.
    *   **Multi-Region Replication (if configured):** Data is asynchronously replicated to other configured Azure regions, providing an additional layer of durability against large-scale regional disasters.
*   **HA/DR Reference:** The local (intra-region) and global (cross-region) replication strategies outlined in `ha_dr_strategy.md` are key to Cosmos DB's strong durability guarantees. Data is persisted across multiple physical replicas before a write is fully acknowledged to the client, according to the configured consistency level and replication settings.

### c. Azure Blob Storage

*   **Ensuring Durability:**
    *   **LRS (Locally-Redundant Storage):** Maintains at least three synchronous copies of data within a single physical location (datacenter) in the primary region. This protects against hardware failures like bad disk drives or node failures.
    *   **ZRS (Zone-Redundant Storage):** Maintains at least three synchronous copies of data spread across different Availability Zones within the primary region. This protects against the failure of an entire datacenter or Availability Zone.
    *   **GRS (Geo-Redundant Storage) / RA-GRS (Read-Access Geo-Redundant Storage):** Data is first replicated using LRS in the primary region (3 copies), and then asynchronously replicated to a secondary region where it is again stored using LRS (another 3 copies). This provides durability against major regional disasters.
    *   **Data Integrity Checks:** Azure Storage uses CRC (Cyclic Redundancy Check) mechanisms to verify data integrity for data at rest and in transit.
*   **HA/DR Reference:** The various storage replication options (LRS, ZRS, GRS, RA-GRS) discussed in `ha_dr_strategy.md` are the core mechanisms providing different levels of data durability for Azure Blob Storage, safeguarding data against a wide range of failure scenarios.

This document provides a foundational understanding of the consistency and durability models for the selected Azure services, crucial for building a reliable e-commerce platform.
