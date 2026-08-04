# Chapter 8 – High Availability, Disaster Recovery, and Replication

## 1. Concept Overview

Hardware fails. Disks crash, network switches die, and data centers lose power. A Database Architect must design a system that survives these catastrophic events. This discipline is divided into two distinct concepts:

*   **High Availability (HA):** The ability of the database to survive a localized failure (like a motherboard burning out) *automatically* and instantly, without user intervention. The goal is to keep the application running.
*   **Disaster Recovery (DR):** The ability to recover the database after a massive, site-wide failure (like a hurricane destroying the data center) or a logical failure (a DBA accidentally dropping a table). This usually requires manual intervention.

To measure success, architects use two metrics:
*   **RPO (Recovery Point Objective):** How much data can the business afford to lose? (e.g., "We can lose the last 5 minutes of data" vs. "Zero Data Loss").
*   **RTO (Recovery Time Objective):** How long can the system be offline before the business goes bankrupt? (e.g., "4 hours" vs. "99.999% uptime, maximum 5 minutes downtime per year").

To achieve HA and DR, databases use **Replication**—constantly copying data from a Primary node to one or more Secondary (Replica) nodes.

## 2. History

In the 1980s, DR meant stopping the database on Friday night, copying the files to a magnetic tape, and driving the tape to a salt mine. If the server crashed on Thursday, you lost 6 days of data (RPO = 6 days).
In the 1990s, **Log Shipping** was invented. The primary database would take a backup of its transaction log every 15 minutes and send it over the network to a "Warm Standby" server.
In the 2000s, **Synchronous Replication** emerged. The Primary database would not commit a transaction to the user until a "Hot Standby" server confirmed it also saved the transaction, guaranteeing Zero Data Loss (RPO = 0), at the cost of network latency.

## 3. Real-world analogy

*   **Standalone Database (No HA/DR):** Flying a single-engine plane with no parachute. If the engine dies, you crash.
*   **Backups (DR only):** Flying a plane with a parachute. If the engine dies, the plane crashes, but you survive. However, you have to hike back to civilization (High RTO).
*   **Log Shipping (Warm Standby):** Flying a twin-engine plane, but the second engine is turned off. If the first dies, you have to manually climb out on the wing and pull-start the second engine.
*   **High Availability (Hot Standby):** Flying a commercial jet with a Co-Pilot. If the Captain passes out, the Co-Pilot instantly grabs the yoke. The passengers barely notice a bump.

## 4. Business problem solved

*   **Hardware Failure:** When the motherboard on the primary database fries, the application automatically redirects to the secondary server in 10 seconds.
*   **Read Scalability:** Millions of users are generating massive read traffic. The architect routes all `SELECT` queries to the Secondary Read Replicas, leaving the Primary server 100% dedicated to `INSERT/UPDATE/DELETE` traffic.
*   **Ransomware/Oops:** A developer executes `UPDATE Users SET Password = 'pwd';` without a `WHERE` clause. HA replicates this mistake instantly. DR (Point-in-Time Recovery backups) allows the DBA to rewind the database to exactly 1 second before the developer hit enter.

---

## 5. Microsoft SQL Server explanation

SQL Server provides the absolute gold standard for enterprise HA/DR: **Always On Availability Groups (AGs)**. 

An AG relies on the underlying OS infrastructure: **Windows Server Failover Cluster (WSFC)**. The WSFC monitors the heartbeat of the servers. If the primary SQL Server stops responding, the WSFC orchestrates an automatic failover. 
Unlike older technologies (like SQL Server Failover Cluster Instances) which shared a single SAN storage array (a single point of failure), Always On AGs are "Shared-Nothing." Every server has its own independent CPU, RAM, and internal Disks.

## 6. SQL Server syntax

Setting up an AG requires extensive PowerShell and Windows GUI configuration. However, the database backup strategy is pure T-SQL:

```sql
-- SQL SERVER SYNTAX
-- 1. Full Backup (The Foundation)
BACKUP DATABASE NextEventDB 
TO DISK = 'Z:\Backups\NextEventDB_FULL.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;
GO

-- 2. Differential Backup (Everything that changed since the last Full)
BACKUP DATABASE NextEventDB 
TO DISK = 'Z:\Backups\NextEventDB_DIFF.bak'
WITH DIFFERENTIAL, COMPRESSION;
GO

-- 3. Transaction Log Backup (Must be run every 5-15 minutes for Point-in-Time Recovery)
BACKUP LOG NextEventDB 
TO DISK = 'Z:\Backups\NextEventDB_LOG.trn'
WITH COMPRESSION;
GO
```

## 7. SQL Server internals

Always On AGs work by intercepting the **Write-Ahead Log (WAL / Transaction Log)**.
When a transaction commits on the Primary:
1.  **Asynchronous Commit (DR / Read Replicas):** The Primary writes the transaction to its local Log file and returns "Success" to the user immediately. In the background, it sends the log block over the network to the Secondary. (Very fast, but risk of data loss if Primary dies before network transmission).
2.  **Synchronous Commit (HA):** The Primary writes to its local log, sends the block to the Secondary, and **waits**. The Secondary writes the block to its local log and sends a "Received" acknowledgement back. Only then does the Primary tell the user "Success." (Guarantees zero data loss, but adds round-trip network latency to every transaction).

## 8. SQL Server execution

**The Split-Brain Problem and Quorum**
Imagine Server A (Primary) and Server B (Secondary) lose network connectivity to each other, but both can still talk to the application. If both servers decide they are the Primary and start accepting writes, the databases will diverge, destroying data integrity. This is **Split-Brain**.

SQL Server prevents this using **Quorum**. To own the database, a server must have a majority of "votes" in the cluster. 
If you have 2 nodes, you add a 3rd microscopic server (or an Azure Cloud Witness) as a tie-breaker. 
If the network splits, Server A can talk to the Witness (2 votes). Server B is isolated (1 vote). Server A stays Primary. Server B goes offline to protect the data.

## 9. SQL Server enterprise examples

*   **Global Distribution:** An enterprise has a Primary SQL Server in New York and a Synchronous Secondary in New York for instant HA. They also have an Asynchronous Secondary in London for DR. If the entire Eastern Seaboard goes dark, they can manually failover to London (accepting a few seconds of data loss to save the company).
*   **Read-Only Routing:** Applications use an "ApplicationIntent=ReadOnly" tag in their connection strings. SQL Server's AG Listener automatically routes these connections to the secondary replicas, vastly increasing capacity.

## 10. SQL Server performance considerations

*   **Synchronous Latency:** If you place a Synchronous Replica in a data center 300 miles away, the speed of light in fiber optics dictates a ~10ms round trip. Every single `COMMIT` in your application will now take 10ms longer. 
*   **Log Send Queue:** If the Secondary server has slow disks, it won't be able to write the incoming transaction logs fast enough. In an Asynchronous setup, the "Log Send Queue" grows. If the Primary dies, any data sitting in that queue is lost forever.

## 11. SQL Server security considerations

*   **Database Mirroring Endpoints:** The network traffic between AG replicas must be secured. SQL Server uses explicit Endpoints (port 5022) which must be secured using Windows Active Directory authentication or TLS certificates.

## 12. SQL Server common mistakes

*   **Forgetting to backup the Transaction Log:** In the `FULL` recovery model, SQL Server keeps transactions in the log file until a Log Backup is taken. If a junior DBA only takes Full Backups every night, the Log File will grow infinitely until it fills the hard drive and crashes the server.
*   **Not checking replica status:** Setting up HA and ignoring it. Six months later, the network drops briefly, replication breaks, and no one notices. When the primary dies, they discover the secondary is 6 months out of date.

## 13. SQL Server best practices

*   Always use a File Share Witness or Cloud Witness for a 2-node AG to maintain Quorum.
*   Perform regular "Fire Drills." Manually failover the AG to the Secondary node during a low-traffic window to prove the application connection strings and routing actually work.

---

## 14. PostgreSQL explanation

PostgreSQL achieves High Availability via **Streaming Replication**. It is incredibly robust, but fundamentally different from SQL Server. 
Postgres replication is strictly at the **Instance (Cluster) level**. You cannot replicate a single database; you replicate the entire server.

Furthermore, native Postgres **does not have built-in automatic failover or cluster management.** Postgres only knows how to stream logs. To achieve true HA, enterprise architects must pair Postgres with a 3rd party tool like **Patroni** (developed by Zalando) or **Repmgr**, combined with a distributed consensus store like `etcd` or `ZooKeeper` to manage Quorum and automate failover.

## 15. PostgreSQL syntax

To configure a Postgres replica, you don't write SQL. You configure files (`postgresql.conf` and `pg_hba.conf`) and use OS tools.

```bash
# POSTGRESQL SYNTAX (OS Level)
# 1. On the Primary: Create a replication user
psql -c "CREATE ROLE replicator WITH REPLICATION PASSWORD 'super_secret' LOGIN;"

# 2. On the Secondary: Take a base backup (copy the physical files from the primary)
pg_basebackup -h primary_ip -D /var/lib/postgresql/data -U replicator -P -v -R -X stream -C -S replica_slot1

# 3. Start the Secondary. The '-R' flag above automatically created standby.signal, 
# telling Postgres to start in Read-Only Replica mode and stream the WAL.
```

## 16. PostgreSQL internals

Postgres replication is based on the **WAL (Write-Ahead Log)**. 
1. The Primary writes changes to the WAL.
2. A `walsender` process on the Primary streams these 16MB WAL segments over the network to the Secondary.
3. A `walreceiver` process on the Secondary writes the stream to its local disk.
4. A `startup` process on the Secondary constantly replays the WAL against its local data pages (Continuous Recovery).

## 17. PostgreSQL execution

**Synchronous Replication in Postgres:**
By default, Postgres replication is asynchronous. To enforce zero data loss:
1. Set `synchronous_standby_names = 'replica1'` in `postgresql.conf` on the Primary.
2. When a transaction commits, the Primary will hang, waiting for `replica1` to acknowledge receipt of the WAL.
*Danger:* If `replica1` goes offline, the Primary will completely freeze and stop accepting all writes, prioritizing data integrity over availability.

## 18. PostgreSQL enterprise examples

*   **Point-In-Time Recovery (PITR) with pgBackRest:** Enterprise DBAs use `pgBackRest` to push WAL files directly to Amazon S3. If someone accidentally drops the `Users` table at 2:05 PM, the DBA restores the nightly base backup, and tells Postgres to replay the WAL files from S3 exactly up until 2:04:59 PM.
*   **Patroni HA Cluster:** The industry standard. 3 Postgres nodes running Patroni. They use an `etcd` cluster for Quorum. Patroni routes all application traffic through HAProxy. If the primary Postgres node crashes, Patroni detects it, promotes a secondary, and reconfigures HAProxy to point to the new primary within 10 seconds.

## 19. PostgreSQL performance considerations

*   **Replication Conflicts:** If a user is running a 5-minute analytical `SELECT` query on the Replica, and the Primary sends over a WAL record that deletes the exact rows the user is currently reading... what happens? 
    *   This is a Replication Conflict. Postgres will cancel the user's `SELECT` query to allow the WAL to replay. To mitigate this, DBAs set `max_standby_streaming_delay` to allow the query to finish, but this causes the Replica to fall behind the Primary.
*   **Replication Slots:** Always use Physical Replication Slots. They force the Primary to retain WAL files on its disk until the Secondary confirms it has received them. (If you don't use slots, and the Secondary disconnects for 2 hours, the Primary might delete the old WAL files. When the Secondary reconnects, it can never catch up and must be rebuilt from scratch).

## 20. PostgreSQL security considerations

*   The `pg_hba.conf` file on the Primary must explicitly grant the `replicator` user access *only* from the IP address of the Secondary node using strong TLS encryption.

## 21. PostgreSQL common mistakes

*   **Assuming Postgres handles failover natively:** Installing Postgres, setting up streaming replication, and assuming you have HA. You only have DR. When the primary dies, your application will just throw connection errors until a human logs in and types `pg_ctl promote` on the secondary.
*   **Orphaned Replication Slots:** You spin up a replica using a slot, then permanently delete the replica VM but forget to drop the slot on the primary. The primary will keep every WAL file forever waiting for the dead replica, eventually filling the entire hard drive and crashing the primary.

## 22. PostgreSQL best practices

*   Always use a 3rd party tool (Patroni) for automated HA.
*   Always use Replication Slots, but monitor `pg_replication_slots` closely.
*   Never run long-reporting queries on a hot replica if the primary is highly transactional (to avoid query cancellation conflicts).

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server (AGs) | PostgreSQL (Patroni) | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Granularity** | Database Level | Instance (Cluster) Level | SQL Server allows failing over just one DB while leaving others on the primary. Postgres fails over the entire server. |
| **Automated Failover** | Native (via WSFC) | Requires 3rd party (Patroni + etcd) | SQL Server is a tightly integrated ecosystem. Postgres requires assembling Lego pieces (but Patroni is incredibly reliable). |
| **Connection Routing** | Native AG Listener | Requires HAProxy / PgBouncer | SQL Server handles Read-Only routing natively via connection strings. |
| **Replication Mechanism** | Log Block Shipping | WAL Streaming / Logical Decoding | Both operate on the Write-Ahead Log. Both support Sync and Async modes. |

## 24. Architect recommendations

**The Quorum/Split-Brain Defense Strategy**
Never deploy a 2-node database cluster across two data centers and expect automatic failover. 
If DC1 and DC2 lose network connection to each other, neither can establish a majority (1 vote out of 2). The cluster will shut down to prevent Split-Brain.
**Architectural Standard:** Always deploy 3 nodes across 3 Availability Zones (AZs). Node A (Primary in AZ1), Node B (Secondary in AZ2), and Node C (Witness/Tie-Breaker in AZ3). If AZ1 completely burns down, AZ2 and AZ3 still form a majority (2 out of 3 votes) and can safely promote AZ2 to Primary.

## 25. DBA recommendations

**STONITH (Shoot The Other Node In The Head)**
In high-end Linux HA clusters, if the primary server stops responding, the cluster orchestration software will physically cut the power to the primary server via an API call to the PDU (Power Distribution Unit) before promoting the secondary. This absolutely guarantees the primary is dead and cannot cause a Split-Brain scenario if its network card suddenly wakes up.

## 26. Developer recommendations

*   **Read-Replica Lag:** When you route a user's `SELECT` query to a Read Replica to save load on the primary, you must handle replication lag. If a user updates their profile picture (hits Primary) and instantly refreshes the page (hits Secondary), they might see their old picture because the WAL hasn't replayed yet (Eventual Consistency). 
*   *Fix:* Implement "Read-Your-Own-Writes." If a user modifies data, write a cookie/token, and force all their read queries to route to the Primary for the next 5 seconds.

## 27. Production case study

**The NextEvent AWS us-east-1 Outage**

*Scenario:* NextEvent was hosting a massive ticket sale on Postgres. We deployed a Patroni HA cluster spanning AWS `us-east-1a` (Primary), `us-east-1b` (Secondary), and `us-east-1c` (etcd Witness). During the peak of the sale, an Amazon backhoe severed the fiber line to data center `1a`. The primary database vanished instantly.

*Resolution:* 
1. Patroni on `1b` and `1c` detected the loss of heartbeat from `1a`.
2. They held an election via `etcd`. `1b` won Quorum.
3. Patroni promoted `1b` to Primary.
4. Patroni reconfigured the local HAProxy nodes to point port 5432 to `1b`.
*Result:* The entire failover took 12 seconds. Application users experienced a brief 12-second loading spinner, followed by a successful ticket purchase. Zero data was lost. The Architecture saved the business.

## 28. ASCII diagrams wherever helpful

**High Availability Quorum Architecture (3-Node)**

```text
       [ APPLICATION (Connection String points to VIP/Listener) ]
                            |
           +---------------------------------+
           |                                 |
 [ NODE A: PRIMARY ]   <-- Sync WAL -->  [ NODE B: SECONDARY ]
 (Location: East US)                     (Location: West US)
           \                                 /
            \                               /
             \-- Heartbeat --+-- Heartbeat -/
                             |
                   [ NODE C: WITNESS ]
                   (Location: Central US)

* If Node A dies, Node B talks to Node C. B + C = Majority (Quorum). 
* Node B is promoted to Primary. VIP moves to Node B.
```

## 29. Enterprise design discussion

**Active-Active vs. Active-Passive Replication**
Executives frequently ask, "Why do we have a $50,000 server sitting there as a replica doing nothing? Let's make it Active-Active so we can write to both at the same time!"

*Architectural Truth:* **Active-Active relational databases are a myth** (or at best, a nightmare). 
If User 1 buys the last ticket on Server A, and User 2 buys the last ticket on Server B at the exact same millisecond, both servers accept the write. When they attempt to synchronize across the network, they encounter a hard conflict. Resolving distributed update conflicts is mathematically impossible without severe application-level logic (e.g., CRDTs). 
Stick to **Active-Passive (Primary-Replica)** for relational databases. If you need multi-master writes across the globe, you must use a NoSQL database (Cassandra) and abandon strict ACID consistency.

## 30. Hands-on exercises

1. Configure a basic Primary/Replica setup using Postgres Streaming Replication (use Docker to spin up two containers).
2. Write a script to insert 1,000 rows into the Primary. Query the Secondary to verify they arrived.
3. `docker stop` the Primary container. Attempt to query the Secondary. Attempt to `INSERT` into the Secondary (it will fail because it is read-only).
4. Run `pg_promote()` on the Secondary to make it the new Primary. Attempt the `INSERT` again.

## 31. Coding exercises

1. In SQL Server, write the exact T-SQL syntax to perform a Log Backup, and then write the syntax to Restore the database `WITH NORECOVERY` (the precursor to log shipping).
2. Write a Postgres query against `pg_stat_replication` to view the current replication lag (in bytes) between the Primary and the Secondary.

## 32. Mini project

**Objective:** Design the DR strategy for the NextEvent Platform.
1. Document the RPO and RTO for the Tickets database (e.g., RPO=0, RTO=1 minute).
2. Design a backup schedule that supports Point-in-Time Recovery (Full backup weekly, Diff nightly, Log every 5 mins).
3. Draw an architecture diagram showing the Primary Server, the HA Replica (Sync), the DR Replica (Async), and the Quorum Witness, placing them in distinct geographic locations.

## 33. Quiz

1. What is the difference between RPO and RTO?
2. Why is Synchronous Replication dangerous if the Secondary server goes completely offline?
3. What is Split-Brain, and what mechanism do clusters use to prevent it?

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is the difference between High Availability (HA) and Disaster Recovery (DR)?
    *   **A:** HA is about keeping the system online through localized failures automatically (e.g., failing over to a hot standby in 10 seconds). DR is about recovering from catastrophic data loss or data center destruction, usually requiring manual backups and taking significantly longer.
*   **Q:** What is a Read Replica?
    *   **A:** A secondary database server that maintains a copy of the primary's data. It is restricted to Read-Only queries, allowing the application to offload heavy `SELECT` traffic and reduce CPU load on the Primary server.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** How does a database achieve Point-in-Time Recovery (PITR)?
    *   **A:** By relying on the Transaction Log (WAL). You restore the last Full Backup, the last Differential Backup, and then sequentially replay all the Transaction Log backups up to the exact millisecond before the disaster occurred.
*   **Q:** In PostgreSQL, you notice the primary server is completely out of disk space, and the `pg_wal` directory is 500GB. What happened?
    *   **A:** A replication slot was created for a secondary replica, but that replica has gone offline permanently. The primary is hoarding all the WAL files, waiting for the dead replica to reconnect. The DBA must drop the orphaned replication slot immediately to allow Postgres to delete the old WAL files.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** You have configured a 2-node SQL Server Always On Availability Group with Synchronous Commit. During peak hours, the application team complains of massive latency spikes on `INSERT` statements. CPU and Disk on the Primary are perfectly fine. What is the bottleneck?
    *   **A:** The bottleneck is the network or the disk on the *Secondary* server. In Synchronous Commit, the Primary must wait for the Secondary to harden the log block to disk and acknowledge it over the network before the Primary can commit. If the network is saturated, or the Secondary's log disk is slow, it throttles the Primary's performance directly.
*   **Q:** Why does Postgres replication (Hot Standby) sometimes cancel long-running `SELECT` queries on the replica? How do you architect around this?
    *   **A:** Postgres uses MVCC. If the Primary executes an `UPDATE` and a subsequent Autovacuum cleans up the old tuple, that WAL record is sent to the Replica. If a user on the Replica is currently reading that old tuple in a long-running query, a Replication Conflict occurs. To maintain data consistency with the primary, Postgres cancels the user's query. To architect around this, you set `hot_standby_feedback = on` (which tells the primary not to vacuum rows the replica is still looking at, though this causes bloat on the primary) or configure `max_standby_streaming_delay` to pause WAL replay temporarily.

## 35. Chapter summary

### Learning Summary
We explored the critical architectural disciplines of High Availability and Disaster Recovery. We learned that RPO and RTO dictate the entire physical design of a system. We configured SQL Server Always On Availability Groups and PostgreSQL Streaming Replication, understanding how both rely on shipping the Transaction Log (WAL) to secondary nodes. We tackled complex distributed systems theory, mastering Quorum, Split-Brain prevention, and the severe latency implications of Synchronous Replication.

### Key Takeaways
*   HA prevents downtime (Auto-Failover); DR prevents data loss (Backups).
*   Synchronous replication guarantees zero data loss at the cost of write-latency.
*   Asynchronous replication is fast but risks data loss during an abrupt failover.
*   Never run a 2-node cluster without a 3rd Witness node to maintain Quorum.
*   PostgreSQL requires 3rd party tools (like Patroni) to achieve true automated High Availability.

### Glossary
*   **RPO:** Recovery Point Objective (Acceptable data loss).
*   **RTO:** Recovery Time Objective (Acceptable downtime).
*   **Quorum:** The majority of votes required for a cluster to function and prevent split-brain.
*   **Log Shipping / WAL Streaming:** The physical process of moving transaction logs to a replica.
*   **PITR:** Point-in-Time Recovery.

### Common Mistakes
*   Running a 2-node cluster without a tie-breaker.
*   Neglecting to test restoring from backups (A backup is worthless until it has been successfully restored).
*   Routing all application read traffic to a replica without accounting for replication lag in the UI.

### Best Practices
*   Use physical replication slots in Postgres to prevent replicas from falling out of sync.
*   Deploy HA nodes across different Availability Zones (AZs).
*   Implement "Shoot The Other Node In The Head" (STONITH) to absolutely guarantee a failed primary node is dead before promoting a secondary.

### Further Reading
*   SQL Server Always On Architecture Guide.
*   PostgreSQL High Availability, Load Balancing, and Replication documentation.
*   Patroni Documentation (Zalando).

### Preparation for Next Chapter
In Chapter 9, we will pivot to **Security, Encryption, and Auditing**. We will learn how to defend the database against SQL Injection, configure Row-Level Security (RLS) to build multi-tenant applications safely, encrypt data at rest (TDE) and in transit (TLS), and mask sensitive PII data from junior developers without breaking the application.
