# Cloud Storage & Database Services — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Companion document to: "Amazon EC2 — Technical Lecture Notes", "Hypervisors — Technical Lecture Notes", and "Cloud Networking (VPC Deep Dive) — Technical Lecture Notes"
### Prerequisite recap assumed: EBS volume types (EC2 notes §7), Nitro System (EC2 notes §2), VPC Endpoints (VPC notes §8.4), memory overcommit/replication concepts (Hypervisor notes §4.3)

---

## 1. Positioning Storage & Database Services in the AWS Stack

Compute (EC2) is stateless-by-default and ephemeral-by-design — an instance can be terminated and replaced at any moment (EC2 notes §3.1). **Every durable byte of application state has to live somewhere else.** This document covers that "somewhere else": the block, file, and object storage layer, and the managed database layer built on top of it.

Formally, extending the layering diagrams from the prior three documents:

```
Physical Data Center (AWS-owned hardware, Regions/Availability Zones)
        |
Nitro System (hardware offload)                        <-- hypervisor layer (compute)
        |
Nitro Card for EBS / Nitro Card for Instance Storage    <-- block storage data-plane (EC2 notes §2.2)
        |
EBS (network-attached block) | Instance Store (local)   <-- block storage layer (EC2 notes §7)
        |
S3 (object) | EFS/FSx (file) | RDS/Aurora/DynamoDB (managed database)   <-- THIS DOCUMENT
```

The key structural distinction that organizes this entire document is the **data model**, not just "where the bytes live":

| Data model | Access granularity | AWS primitives | Conceptual analogue |
|---|---|---|---|
| **Block** | Fixed-size blocks, addressed by LBA (logical block address), mounted as a raw device | EBS, Instance Store | Already covered — EC2 notes §7 |
| **File** | Hierarchical namespace, POSIX file semantics, shared concurrent access | EFS, FSx (for Windows/Lustre/NetApp ONTAP) | New in this document, §3 |
| **Object** | Flat namespace, whole-object PUT/GET, rich metadata, no partial in-place mutation | S3, S3 Glacier | New in this document, §2 |
| **Managed relational database** | SQL, ACID transactions, fixed schema | RDS, Aurora | New in this document, §5–6 |
| **Managed NoSQL database** | Schema-flexible, horizontally partitioned, tunable consistency | DynamoDB | New in this document, §7–8 |

---

## 2. Object Storage — Amazon S3

### 2.1 Formal definition and architectural reasoning

S3 (Simple Storage Service, launched 2006 — AWS's *first* public service, predating EC2) is **not a filesystem**. It is a **flat key-value namespace** per bucket: every object is addressed by a unique key (an opaque string, not a real directory path — the "folder" structure seen in the S3 console is a UI convenience built by splitting keys on `/`, not a real hierarchical filesystem). This is a deliberate architectural simplification relative to a POSIX filesystem, and it is the reason S3 can offer **virtually unlimited scale with flat, predictable performance**: a hierarchical filesystem requires directory-level locking/metadata operations that become a bottleneck at scale; a flat key space with no directory inode to contend over does not.

**Formal object model**: an S3 object = `(key, value/bytes, version ID, metadata, ACL)`, stored in a **bucket** (a Regional, globally-uniquely-named container). Objects are **immutable** — there is no in-place partial byte-range write; any "update" is really a full object PUT that creates a new version (if versioning is enabled) or overwrites the key.

### 2.2 Consistency model — a direct extension of the Hypervisor notes' consistency reasoning

S3 originally offered only **eventual consistency** for overwrite PUTs and DELETEs (a reader could briefly see stale data after a write, while always offering read-after-write consistency for new-object PUTs). As of **December 2020**, S3 provides **strong read-after-write consistency for all operations** — GET immediately after a PUT (new object, overwrite, or DELETE) is guaranteed to return the latest version, with no eventual-consistency window at all. This is a useful, exam-relevant historical inflection point: it is the same class of engineering problem (and the same underlying tradeoff — the CAP-theorem-style tension between availability/partition-tolerance and consistency) that governs the DynamoDB quorum model in §8, and is structurally the same tradeoff space as the Hypervisor notes' memory-coherence problem across NUMA nodes (Hypervisor notes §6) — multiple physical copies of the same logical data, and a protocol deciding when all readers are guaranteed to observe the latest write.

### 2.3 Storage Classes — the quantitative core of S3

S3 storage classes trade **retrieval latency and per-request cost** against **per-GB storage cost**, governed by how "hot" (frequently accessed) the data is expected to be — directly analogous in *spirit* to the EBS volume type tradeoff table in the EC2 notes (§7.2), where SSD vs HDD-backed volume types trade IOPS/throughput against $/GB.

| Storage Class | Availability SLA | Min storage duration | Retrieval time | Typical use case |
|---|---|---|---|---|
| **S3 Standard** | 99.99% | None | Milliseconds | Frequently accessed, general-purpose data |
| **S3 Intelligent-Tiering** | 99.9% | None | Milliseconds (auto-tiers between access-pattern-based tiers) | Unknown/changing access patterns — AWS monitors access and moves objects automatically, no retrieval fee for tier changes |
| **S3 Standard-IA** (Infrequent Access) | 99.9% | 30 days | Milliseconds | Backups, DR data accessed occasionally but needing rapid access |
| **S3 One Zone-IA** | 99.5% (single-AZ, no cross-AZ redundancy) | 30 days | Milliseconds | Re-creatable data (e.g., secondary backup copies), where losing an AZ is acceptable |
| **S3 Glacier Instant Retrieval** | 99.9% | 90 days | Milliseconds | Archive data still needing millisecond access (e.g., medical images, news media archives) |
| **S3 Glacier Flexible Retrieval** | 99.99% | 90 days | Minutes to hours (Expedited: 1–5 min; Standard: 3–5 hr; Bulk: 5–12 hr) | Archive data, infrequent retrieval acceptable |
| **S3 Glacier Deep Archive** | 99.99% | 180 days | 12–48 hours | Long-term compliance archives (7–10 year regulatory retention), lowest cost per GB of any AWS storage |

**Lifecycle policies** automate the movement of objects across these classes by object age (e.g., Standard → Standard-IA at 30 days → Glacier at 90 days → Deep Archive at 365 days → expiration at 7 years) — this is the storage-layer equivalent of the EC2 notes' Auto Scaling policies (§11): a **declarative, policy-driven control loop** that continuously reconciles actual state (object age/access pattern) against a target (desired storage class), removing the need for manual per-object intervention.

### 2.4 Durability — the "11 nines" formalized

S3 Standard, Standard-IA, and Intelligent-Tiering advertise **99.999999999% (11 nines) annual durability**, achieved by synchronously replicating each object across a **minimum of 3 Availability Zones** within the Region at the time of the PUT.

**Formal interpretation**: if $D = 0.99999999999$ is the annual durability probability for a single object, and you store $n$ independent objects, the **expected number of objects lost per year**:

$$E[\text{objects lost/year}] = n \times (1 - D)$$

**Worked example**: for $n = 10{,}000{,}000$ objects stored: $E[\text{loss}] = 10^7 \times (1 - 0.99999999999) = 10^7 \times 10^{-11} = 10^{-4}$ objects/year — i.e., you would statistically expect to lose one object roughly every **10,000 years** at this scale, versus the single-digit-nines durability of a typical unreplicated single-disk deployment.

**Durability vs Availability — a distinction students frequently conflate:**

| Property | What it measures | S3 Standard figure |
|---|---|---|
| **Durability** | Probability data is **not permanently lost** over a year (a storage-layer/replication guarantee) | 99.999999999% (11 nines) |
| **Availability** | Probability the service **responds successfully to a request** at any given moment (an uptime/operational guarantee) | 99.99% |

An object can be 100% durable (never lost) while the *service* is briefly unavailable (a transient 503) — these are independent axes, exactly as the EC2 notes distinguish "instance state" (billing/lifecycle) from "status check" (operational health, EC2 notes §14).

### 2.5 S3 request performance and partitioning

Modern S3 (post-2018 architectural change) achieves **at least 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second, per prefix**, with no practical limit on the number of prefixes a bucket can have — meaning aggregate bucket throughput scales roughly linearly with the number of distinct key prefixes used, since S3 automatically partitions the keyspace across its internal index infrastructure. (Historically, pre-2018, S3 performance depended heavily on **key-name randomization** to avoid hot-partitioning on sequential key prefixes, such as timestamp-prefixed keys — a legacy design constraint still worth knowing since a meaningful fraction of production S3 usage patterns and interview questions still reference it.)

**Aggregate throughput formula:**

$$\text{Max aggregate request rate} = k \times R_{prefix}$$

where $k$ = number of distinct prefixes in use and $R_{prefix}$ = the per-prefix request-rate ceiling (3,500 or 5,500 depending on operation type).

---

## 3. File Storage — EFS and FSx

### 3.1 Why file storage is a distinct category from block and object

A shared, POSIX-compliant filesystem that multiple EC2 instances can **mount concurrently** cannot be built directly on EBS (an EBS volume, outside of Multi-Attach io1/io2 — EC2 notes §7.3 — is fundamentally single-writer, and even Multi-Attach provides no filesystem-level coordination, only block-level concurrent attachment) and cannot be built on S3 (no POSIX semantics, no partial in-place writes, no file locking). **Amazon EFS (Elastic File System)** fills this gap: a fully managed, elastic NFSv4-compliant filesystem, mountable concurrently by thousands of EC2 instances (and Lambda functions) across multiple AZs simultaneously.

### 3.2 EBS vs EFS vs S3 — comparison table

| Property | EBS | EFS | S3 |
|---|---|---|---|
| Data model | Block | File (POSIX, hierarchical) | Object (flat key-value) |
| Attachment scope | Single instance at a time (Multi-Attach: up to 16, same AZ, io1/io2 only) | Thousands of instances/AZs concurrently | Not "attached" — accessed via API/HTTP from anywhere |
| AZ scope | Single AZ (must match the instance's AZ) | Regional — spans all AZs automatically | Regional (with optional Cross-Region Replication) |
| Elasticity | Manually resized (`ModifyVolume`) | Automatically grows/shrinks with usage, no pre-provisioning | Effectively infinite, no provisioning at all |
| Performance model | Provisioned (IOPS/throughput as a billed parameter, EC2 notes §7.2) | Two throughput modes: Bursting (scales with filesystem size) or Provisioned (fixed, billed independently) | Request-rate based (§2.5), not IOPS-based |
| Typical use case | Boot volumes, databases (single-writer) | Shared web content, CMS uploads, big data/analytics working directories, container persistent volumes shared across tasks | Static assets, data lakes, backups, archives |

### 3.3 FSx family (brief, for completeness)

| FSx variant | Underlying filesystem | Typical use case |
|---|---|---|
| **FSx for Windows File Server** | SMB, native Windows ACLs/AD integration | Windows-based enterprise applications requiring native SMB |
| **FSx for Lustre** | Lustre (HPC-oriented parallel filesystem) | Machine learning training, HPC, genomics — can be linked directly to an S3 bucket as its backing data repository |
| **FSx for NetApp ONTAP** | NetApp ONTAP | Lift-and-shift of on-premises NetApp-based workloads, needing ONTAP-specific features (snapshots, dedup, multi-protocol) |
| **FSx for OpenZFS** | OpenZFS | POSIX workloads needing ZFS-specific snapshot/cloning performance |

---

## 4. Storage Consistency and Replication — Unifying Theory

### 4.1 The general replication problem (echoing the Hypervisor notes)

Every AWS storage/database service that offers durability guarantees is solving the same underlying problem the Hypervisor notes' live migration section addresses (Hypervisor notes §7): **keeping multiple physical copies of the same logical data synchronized while continuing to serve reads/writes.** The specific mechanism differs by service, but the taxonomy is shared:

| Replication strategy | Mechanism | AWS example |
|---|---|---|
| **Synchronous replication** | Write is not acknowledged to the client until **all** (or a quorum of) replicas confirm | RDS Multi-AZ (standby), Aurora storage layer (6-way, quorum-based, §6.2) |
| **Asynchronous replication** | Write is acknowledged after the **primary** persists it; replicas catch up afterward, with a nonzero replication lag | RDS Read Replicas (cross-Region), S3 Cross-Region Replication |
| **Quorum-based replication** | A configurable subset ($W$ of $N$) of replicas must acknowledge a write, and a configurable subset ($R$ of $N$) must be read, with the guarantee $R + W > N$ giving strong consistency | DynamoDB internal replication (§8) |

### 4.2 RPO and RTO — the formal metrics for "how much data/downtime can you tolerate"

| Metric | Definition | Formula/relationship |
|---|---|---|
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss, measured as time | Bounded below by replication lag: $\text{RPO} \geq \text{Replication Lag}$ for asynchronous replication; $\text{RPO} \approx 0$ for synchronous replication |
| **RTO (Recovery Time Objective)** | Maximum acceptable time to restore service after a failure | Determined by failover mechanism: automated Multi-AZ failover (RDS: typically 60–120 seconds) vs manual restore-from-snapshot (potentially hours, proportional to data volume) |

This RPO/RTO framing is the database-layer analogue of the Hypervisor notes' live migration downtime discussion (Hypervisor notes §7.1–7.2): pre-copy migration minimizes downtime (low RTO-equivalent) at the cost of migration complexity, exactly as synchronous Multi-AZ replication minimizes RPO at the cost of write latency (§6.1).

---

## 5. Managed Relational Databases — Amazon RDS

### 5.1 What "managed" removes from the customer's operational burden

RDS provisions a database engine (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, or Aurora — §6) **on top of EC2-class compute and EBS-class storage that the customer never directly manages**: AWS handles OS patching, engine version patching (customer-controlled maintenance window), automated backups, and Multi-AZ failover orchestration. This is the database-layer instance of the same "managed convenience vs customer control" tradeoff seen twice already: NAT Gateway vs NAT Instance (VPC notes §5.2) and EBS vs Instance Store (EC2 notes §7.1) — RDS trades the ability to SSH into the underlying host (and hence run arbitrary OS-level tuning or third-party agents) for zero patching/backup/failover-orchestration burden.

### 5.2 RDS Multi-AZ — mechanism

An RDS Multi-AZ deployment provisions a **synchronous standby replica in a second AZ**. Every write is applied to the primary and **synchronously replicated to the standby before being acknowledged to the client** — this is why Multi-AZ is a *high-availability* feature, not a *read-scaling* feature (the standby is not directly queryable in the classic Multi-AZ model; RDS Multi-AZ with **two readable standbys**, a newer deployment option, does allow read traffic against replicas, structurally converging toward Aurora's model in §6).

On primary failure, RDS automatically **promotes the standby**, and updates the DNS CNAME endpoint to point at the new primary — client applications reconnect using the same endpoint hostname, with typical failover completing in 60–120 seconds (this interval is the RDS Multi-AZ RTO figure referenced in §4.2).

### 5.3 RDS Read Replicas — mechanism and consistency

Read Replicas use **asynchronous, engine-native replication** (e.g., MySQL binlog replication) to maintain up to 5 (engine-dependent) read-only copies, which **can** be cross-Region. Because replication is asynchronous, Read Replicas exhibit **eventual consistency** — a read against a replica may return data slightly older than the primary, with **replication lag** as the operative metric (visible via the `ReplicaLag` CloudWatch metric, in seconds).

### 5.4 RDS Multi-AZ vs Read Replicas — comparison table

| Property | Multi-AZ (standby) | Read Replica |
|---|---|---|
| Purpose | High availability / failover | Read scalability, and can itself be promoted to a standalone primary (e.g., for offloading reporting or as a DR base) |
| Replication mode | Synchronous | Asynchronous |
| Consistency for reads | N/A in classic mode (standby not readable) | Eventual consistency, bounded by replication lag |
| Automatic failover | Yes (automatic, DNS-based) | No (manual promotion required) |
| Cross-Region support | Only via the newer Multi-AZ DB Cluster deployment option | Yes, natively |
| Number of replicas | Exactly 1 standby (classic) | Up to 5 (engine-dependent) |
| Billing | Full second instance, billed continuously | Full additional instance, billed continuously |

### 5.5 RDS storage — the EBS connection

RDS storage volumes are, under the hood, **EBS volumes** (gp3, io1/io2, or magnetic — the same volume type taxonomy from EC2 notes §7.2), meaning RDS IOPS/throughput provisioning and cost math follows the **exact same formulas** already covered there — this is a direct, literal reuse of the EC2 notes' quantitative content rather than merely an analogy.

---

## 6. Amazon Aurora — Re-architected Relational Storage

### 6.1 Why Aurora exists — architectural reasoning

Standard RDS (§5) treats the database engine and its underlying storage as tightly coupled: the engine issues writes to a local (EBS-backed) volume, and Multi-AZ HA is achieved by shipping the **entire write stream (or full page images)** to a synchronous standby — which means HA cost scales with total data volume replicated wholesale. Aurora **decouples the database engine (compute) from a purpose-built, distributed, log-structured storage layer**, replicated 6 ways across 3 AZs **at the storage layer itself**, rather than at the engine layer. This is architecturally the same "separate the thing that needs elasticity/redundancy from the thing that needs raw compute speed" reasoning behind the Nitro System's separation of I/O (offloaded to hardware) from CPU/memory virtualization (Hypervisor notes §3.4, EC2 notes §2) — Aurora offloads storage durability/replication out of the database engine process entirely, onto a dedicated distributed storage service, the same way Nitro offloads I/O out of the hypervisor process onto dedicated cards.

### 6.2 Aurora's 6-way quorum storage model

Aurora writes **only the redo log stream** (not full data pages) to its storage layer — a substantially smaller amount of data per transaction than traditional page-based replication. This log stream is replicated **6 ways across 3 AZs** (2 copies per AZ), using a **quorum write/read protocol**:

$$\text{Write quorum} = 4 \text{ of } 6, \quad \text{Read quorum} = 3 \text{ of } 6$$

satisfying the general quorum-consistency condition $R + W > N$ (here $3 + 4 = 7 > 6$), which guarantees any read quorum overlaps with any prior write quorum by at least one copy — this is the **same quorum mathematics formalized in full generality in §8.2** for DynamoDB; Aurora is simply a fixed-parameter instance of it ($N{=}6, W{=}4, R{=}3$).

**Practical consequence**: Aurora tolerates losing an entire AZ **without losing write availability** (4-of-6 is still achievable with one AZ, i.e., 2 copies, down), and tolerates losing an AZ **plus one additional copy** without losing read availability.

### 6.3 RDS (standard) vs Aurora — comparison table

| Property | RDS (standard engines) | Aurora |
|---|---|---|
| Storage architecture | Single EBS volume per instance, replicated wholesale for HA | Distributed, log-structured storage service, 6-way quorum replicated across 3 AZs natively |
| Storage scaling | Manually provisioned, up to 64 TiB (engine-dependent) | Auto-scales storage in 10 GB increments, up to 128 TiB, with no pre-provisioning |
| Read replica count | Up to 5 | Up to 15 Aurora Replicas, sharing the **same underlying storage volume** as the primary (near-zero replication lag since replicas read the same log-structured storage, not a separately materialized copy) |
| Failover time | 60–120 seconds (DNS-based) | Typically under 30 seconds (Aurora Replicas can be promoted almost immediately since they already share the storage layer) |
| Engine compatibility | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL-compatible and PostgreSQL-compatible only |
| Cost | Generally lower baseline cost | ~20% higher compute cost than equivalent RDS, offset by storage efficiency and reduced replica lag/failover benefits |
| Serverless option | Limited | **Aurora Serverless v2** — scales compute capacity (in fine-grained Aurora Capacity Units, ACUs) up/down automatically based on load, without connection-dropping failover |

---

## 7. Amazon DynamoDB — Managed NoSQL

### 7.1 Formal data model and architectural motivation

DynamoDB is a **managed, horizontally partitioned key-value/wide-column NoSQL database**, built on the internal lineage of Amazon's 2007 **Dynamo** paper — the same paper that popularized the quorum-consistency and consistent-hashing techniques now common across the distributed-systems field. DynamoDB exists because relational databases (§5–6), even when horizontally read-scaled via replicas, remain **fundamentally single-writer per shard/partition** at the write path and require a fixed schema — properties that do not suit workloads needing **massive, horizontally-scalable write throughput with flexible, per-item schema** (e.g., a shopping cart, a session store, an IoT telemetry sink).

**Core data model**: a DynamoDB table is a collection of **items** (analogous to rows), each uniquely identified by a **primary key** — either a simple **Partition Key (PK)** alone, or a **composite key** (Partition Key + Sort Key). Items within a table need not share the same set of attributes (schema-flexible), unlike a relational table's fixed column set.

### 7.2 Partitioning mechanism — the direct link to consistent hashing

DynamoDB automatically partitions a table's data across multiple physical storage partitions, using the **hash of the Partition Key** to determine which partition an item lives on. This is the same fundamental mechanism as the original Dynamo paper's **consistent hashing** ring, and it is why **partition key selection quality directly determines achievable throughput**: a poorly chosen partition key (e.g., a key with very few distinct values, or a "hot" value receiving disproportionate traffic) causes a **hot partition**, since DynamoDB's total table throughput is the **sum of each partition's individual throughput ceiling**, not a single shared pool.

$$\text{Table throughput ceiling} = \sum_{i=1}^{n_{partitions}} \text{Partition}_i \text{ throughput ceiling}$$

A hot partition means one $\text{Partition}_i$ is saturated while others sit idle — the table-level `ProvisionedThroughputExceededException`/throttling occurs **even if aggregate table capacity is nowhere near fully utilized**, because DynamoDB cannot "borrow" unused capacity from a cold partition for a hot one.

### 7.3 Capacity modes

| Capacity mode | Billing unit | Mechanism | Best for |
|---|---|---|---|
| **Provisioned** | RCU (Read Capacity Unit) / WCU (Write Capacity Unit), pre-declared per table (or per Auto Scaling policy) | Customer declares a target throughput; DynamoDB provisions partitions to support it | Predictable, steady-state workloads; cheaper at sustained high utilization |
| **On-Demand** | Per-request, pay-per-use | DynamoDB auto-scales partitions transparently based on observed traffic, no pre-declaration | Unpredictable/spiky workloads, new applications with unknown traffic patterns |

### 7.4 RCU/WCU formulas — the quantitative core of DynamoDB

**1 RCU** = one **strongly consistent** read of up to **4 KB** per second (equivalently, **two eventually consistent reads** of up to 4 KB per second, since eventually consistent reads cost half as much — a direct, quantified expression of the consistency-vs-throughput tradeoff, exactly the kind of formula-rich tradeoff this document series favors).

**1 WCU** = one write of up to **1 KB** per second.

**Required RCU for a given read workload:**

$$\text{RCUs required} = \left\lceil \frac{\text{Item size (KB)}}{4} \right\rceil \times \text{Reads/sec} \times \begin{cases} 1 & \text{strongly consistent} \\ 0.5 & \text{eventually consistent} \end{cases}$$

**Required WCU for a given write workload:**

$$\text{WCUs required} = \left\lceil \frac{\text{Item size (KB)}}{1} \right\rceil \times \text{Writes/sec}$$

**Worked example**: an application performs 100 strongly consistent reads/sec of 6 KB items, and 50 writes/sec of 2 KB items.

$$\text{RCUs} = \lceil 6/4 \rceil \times 100 \times 1 = 2 \times 100 = 200 \text{ RCU}$$
$$\text{WCUs} = \lceil 2/1 \rceil \times 50 = 2 \times 50 = 100 \text{ WCU}$$

This RCU/WCU quantization (round item size **up** to the next 4 KB or 1 KB boundary) is structurally identical in spirit to the EC2 notes' CPU credit model (EC2 notes §6) and the gp2 IOPS credit model (EC2 notes §7.2) — a discretized unit of a continuous resource, deliberately designed to be a clean, formula-friendly billing/provisioning primitive.

### 7.5 DynamoDB consistency model — read consistency choice per request

Unlike S3 (where consistency is now uniformly strong, §2.2) or RDS Read Replicas (where consistency is uniformly eventual, §5.3), DynamoDB is unusual in offering **per-request consistency choice**:

| Read type | Guarantee | Cost (RCU) | Latency |
|---|---|---|---|
| **Eventually consistent read** (default) | May return stale data if a very recent write has not yet propagated to all replicas (typically sub-second propagation) | 0.5 RCU per 4 KB | Lower |
| **Strongly consistent read** | Always returns the most recent successful write | 1 RCU per 4 KB | Slightly higher, and unavailable during certain network partition scenarios (unlike eventually consistent reads, which remain available) |

This is a direct, table-level realization of the general **CAP theorem tradeoff** (and the same tradeoff structurally underlying Aurora's read quorum in §6.2 and S3's 2020-era consistency upgrade in §2.2): DynamoDB explicitly exposes the availability/consistency choice to the *application developer*, on a per-request basis, rather than baking in one fixed answer.

### 7.6 Global Tables — multi-Region active-active replication

**DynamoDB Global Tables** replicate a table across multiple Regions with **multi-active** (multi-master) writes accepted in any participating Region, reconciled via **last-writer-wins** conflict resolution (based on internal timestamps). This is the database-layer analogue of the VPC notes' multi-Region connectivity discussion (VPC notes §13.3) — Global Tables solve "how do independently-writable copies in different Regions converge to the same state," exactly as Transit Gateway inter-Region peering solves "how do independently-routable VPCs in different Regions communicate."

---

## 8. Quorum Consistency — Formalized

This section generalizes the quorum reasoning introduced for Aurora (§6.2) into the full $N/R/W$ model, since it is the single most numerically productive concept in this document and directly reusable across S3, Aurora, and DynamoDB reasoning.

### 8.1 The general model

Let $N$ = total number of replicas of a piece of data, $W$ = number of replicas that must acknowledge a write before it is considered successful, $R$ = number of replicas that must respond to a read for it to be considered successful.

**Strong consistency condition:**

$$R + W > N$$

This guarantees that any read set and any write set **must overlap in at least one replica** — pigeonhole principle — so a read is guaranteed to see at least one copy reflecting the most recent acknowledged write.

### 8.2 What different $(N, R, W)$ choices mean in practice

| Configuration | Property | Real-world analogue |
|---|---|---|
| $W = N$, $R = 1$ | Slow, highly durable writes; fast reads (any single replica is guaranteed current since **all** replicas were required to ack) | RDS Multi-AZ (conceptually — synchronous write to all copies before ack) |
| $W = 1$, $R = N$ | Fast writes; slow, expensive reads (must check every replica to be sure of freshness) | Rarely used in practice — read cost usually dominates |
| $R + W \leq N$ | **Eventual consistency** — no overlap guarantee, but higher availability (fewer replicas need to be reachable for either operation) | DynamoDB eventually consistent reads; async Read Replicas |
| $R + W > N$, balanced | **Strong consistency** with a tunable availability/latency balance | Aurora ($N{=}6, W{=}4, R{=}3$); Cassandra-style tunable quorums (`QUORUM` reads/writes) |

### 8.3 Availability implication of quorum size

A write (or read) can only succeed if **at least $W$ (or $R$) replicas are reachable**. Given each individual replica has independent availability probability $p$, the probability that **at least $k$ of $N$ replicas are simultaneously reachable** follows the binomial tail:

$$P(\text{at least } k \text{ of } N \text{ available}) = \sum_{i=k}^{N} \binom{N}{i} p^{i} (1-p)^{N-i}$$

This is the formal justification for why quorum systems tolerate partial failures gracefully: as long as **fewer than $N - W + 1$** replicas are simultaneously down, writes still succeed (for Aurora's $N{=}6, W{=}4$: up to 3 replicas — i.e., a full AZ's worth of 2 copies plus one more — can be down and writes still succeed).

---

## 9. Backup, Snapshot, and DR Mechanics

### 9.1 EBS/RDS snapshots — the shared incremental mechanism

RDS automated backups and EBS snapshots (already introduced in EC2 notes §7.3) share the same underlying mechanism: an **incremental, block-level snapshot chain stored in S3**, where the first snapshot is a full copy and every subsequent snapshot stores only changed blocks since the previous snapshot, while each individual snapshot remains independently restorable to a complete point-in-time image (AWS manages the delta-chain bookkeeping internally). RDS additionally supports **point-in-time restore (PITR)**, reconstructing database state at any second within the backup retention window (1–35 days) by replaying the transaction log forward from the nearest snapshot — directly analogous to a database write-ahead log (WAL) replay.

### 9.2 Cross-service backup comparison table

| Service | Backup mechanism | Granularity of recovery | Cross-Region support |
|---|---|---|---|
| EBS | Incremental snapshot to S3 | Point-in-time (at snapshot time only, unless combined with a WAL-based service on top) | Yes (manual/automated copy) |
| RDS | Automated backups + transaction log shipping | Any second within retention window (PITR) | Yes (automated backups can be replicated cross-Region) |
| Aurora | Continuous backup to S3 (no performance impact on primary, since backup reads from the distributed storage layer, not the engine) | Any second within retention window (up to 35 days) | Yes |
| S3 | Versioning (per-object) + Cross-Region Replication | Per-object-version restore | Yes (CRR, asynchronous) |
| DynamoDB | On-demand backups (full, no performance impact) + Point-in-Time Recovery (continuous, 35-day window) | Any second within PITR window | Global Tables provide the multi-Region equivalent (§7.6), rather than a "backup" per se |

---

## 10. Cross-Reference Summary — How This Document Connects to EC2, Hypervisor, and VPC Notes

| Concept in this document | Connects to |
|---|---|
| RDS/Aurora storage volumes are EBS-backed | EC2 notes §7.2 — IOPS/throughput formulas apply directly, not by analogy |
| Incremental snapshot chain (EBS/RDS) | EC2 notes §7.3 — identical underlying block-delta mechanism |
| Aurora's decoupling of compute from a distributed storage layer | Hypervisor/EC2 notes' Nitro System reasoning — same "move durability/redundancy off the primary compute path onto dedicated infrastructure" pattern |
| S3/DynamoDB/Aurora consistency models | Hypervisor notes §6 (NUMA memory coherence) and Hypervisor notes §4.3 (overcommitment) — all are instances of "multiple physical copies, one logical value" |
| RCU/WCU discretized billing unit | EC2 notes §6 (CPU credits) and §7.2 (gp2 IOPS credits) — same discretized-resource-unit design pattern |
| Lifecycle policies (S3) | EC2 notes §11 (Auto Scaling policies) — same declarative, policy-driven reconciliation-loop pattern |
| DynamoDB partitioning / hot partitions | VPC notes' CIDR/subnet partitioning discussion (§14.1) — both are cases where a resource's *effective* capacity depends on how evenly a key/address space is divided, not just total nominal capacity |
| Global Tables multi-Region active-active | VPC notes §13.3 (cross-Region connectivity via TGW peering) — both solve multi-Region consistency/reachability, at the database layer vs the network layer respectively |
| RDS Multi-AZ failover via DNS CNAME | VPC notes §10 (Route 53 Resolver) — same "indirection via DNS to survive endpoint changes" pattern used for PrivateLink private DNS names |
| VPC Endpoints for S3/DynamoDB (Gateway Endpoint) | VPC notes §8.4 — direct reuse: S3 and DynamoDB are literally the two services Gateway Endpoints support |

---

## 11. Suggested Numerical / Formula-Based Problem Bank

1. **S3 durability**: Given a target annual durability of 99.999999999% and a stated total object count stored across a fleet of buckets, compute the expected number of objects lost per year, and the expected number of years between individual object-loss events.
2. **DynamoDB RCU/WCU sizing**: Given item size, target reads/sec (specify strongly or eventually consistent), and target writes/sec, compute the required provisioned RCU and WCU; then, given a per-RCU and per-WCU hourly rate, compute the monthly provisioned-capacity cost, and compare it against the equivalent On-Demand cost at the same request volume.
3. **DynamoDB hot partition detection**: Given a table's total provisioned throughput and the number of partitions DynamoDB has created (derivable from total provisioned throughput and per-partition limits), and an observed traffic distribution heavily skewed toward one partition key value, compute the effective achievable throughput before throttling occurs, and contrast it with the table's nominal provisioned throughput.
4. **Quorum availability**: Given $N$ replicas each with independent availability probability $p$, and a required write quorum $W$, compute the probability that a write succeeds (at least $W$ of $N$ replicas reachable) using the binomial tail formula, for Aurora's default $(N{=}6, W{=}4)$ configuration at a given $p$.
5. **RPO/backup cost tradeoff**: Given a database's data change rate (GB/hour), a chosen backup/replication strategy (synchronous Multi-AZ vs asynchronous cross-Region replica with a stated lag), and a disaster scenario, compute the expected data loss (RPO) under each strategy, and the added replication data-transfer cost for the lower-RPO option.

---

## 12. Summary Table — Storage & Database Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Object storage | S3 flat key-value model | Immutable objects, flat namespace enables near-infinite flat-performance scale, unlike a hierarchical filesystem |
| Object storage | Storage Classes | Retrieval latency/cost vs $/GB storage cost tradeoff, automatable via Lifecycle policies |
| Object storage | Durability vs Availability | Independent axes — 11-nines durability (data survival) is not the same guarantee as 99.99% availability (request success) |
| File storage | EFS/FSx | Fills the "shared, concurrent, POSIX" gap that neither block (single-writer) nor object (no partial writes) storage can serve |
| Replication theory | Sync vs Async vs Quorum | RPO is bounded below by replication lag; synchronous replication ≈ zero RPO at a write-latency cost |
| Relational DB | RDS Multi-AZ vs Read Replicas | Synchronous HA-only standby vs asynchronous, eventually consistent, horizontally read-scaling replicas |
| Relational DB | Aurora | Decouples compute from a 6-way quorum-replicated distributed log-structured storage layer; replicas share storage, not just data |
| Quorum math | $R + W > N$ | The general strong-consistency condition; Aurora ($4/3$ of 6) and DynamoDB per-request consistency choice are both concrete instances |
| NoSQL | DynamoDB partitioning | Table throughput = sum of per-partition ceilings; partition key choice, not aggregate capacity, is the real throughput constraint |
| NoSQL | RCU/WCU | Discretized billing units (4 KB read, 1 KB write), same design pattern as EC2/EBS credit systems |
| Multi-Region | Global Tables | Multi-active writes, last-writer-wins conflict resolution — database-layer analogue of VPC's inter-Region TGW peering |
| DR | RPO / RTO | RPO governed by replication mode and lag; RTO governed by failover automation — Multi-AZ automates both, cross-Region async does not |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 11 reserved as a follow-up tutorial/assignment. Designed to be read as the fourth document in this set — Sections 2.4, 4, 6.2, 7.4, and 8 map directly onto the credit-model and consistency reasoning in the EC2 and Hypervisor notes, and Section 10 indexes every cross-document connection explicitly.*
