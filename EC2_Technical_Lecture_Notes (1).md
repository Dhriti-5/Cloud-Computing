# Amazon Elastic Compute Cloud (EC2) — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Prerequisite recap assumed: Cloud Computing fundamentals, AWS basics, Hypervisor fundamentals (Type-1/Type-2, Xen, KVM)

---

## 1. Positioning EC2 in the AWS Stack

Amazon EC2 (Elastic Compute Cloud) is AWS's core **Infrastructure-as-a-Service (IaaS)** compute offering. Where a hypervisor (which you have already studied — Xen, KVM, VMware ESXi) provides the mechanism to partition a single physical machine into multiple isolated virtual machines, EC2 is the **service layer** that:

1. Abstracts the hypervisor and physical host away from the customer.
2. Exposes virtual machines ("instances") as an on-demand, API-driven, billable resource.
3. Adds elasticity — the ability to programmatically create, resize, and destroy compute capacity in seconds rather than the weeks/months required to provision physical servers.

Formally, EC2 sits at this layer of the stack:

```
Physical Data Center (AWS-owned hardware, Regions/Availability Zones)
        |
Nitro System (hardware offload + lightweight hypervisor)   <-- hypervisor layer
        |
EC2 (virtual machine provisioning, lifecycle, API, billing) <-- IaaS layer
        |
AMI / Guest OS + your application stack
```

EC2 is not "a server." EC2 is a **control plane + hypervisor fabric** that produces virtual machines called **instances** as its unit of output.

---

## 2. The AWS Nitro System — EC2's Modern Hypervisor Architecture

This is the direct continuation of your hypervisor module, so we go deep here.

### 2.1 Why Nitro exists

Traditional hypervisors (Xen in EC2's first ~10 years, 2006–2017) run **all** virtualization functions — CPU scheduling, memory management, storage I/O emulation, network I/O emulation, and the management/control-plane agent — inside software on the host. This software layer consumes host CPU cycles ("hypervisor tax"), typically 10–30% of a host's total capacity, and increases the attack surface available to a guest attempting to escape its VM.

AWS re-architected this starting around 2013 (public from 2017) into the **Nitro System**, which decomposes the hypervisor into specialized hardware and a minimal software core.

### 2.2 Components of the Nitro System

| Component | Function |
|---|---|
| **Nitro Cards** (Nitro Card for VPC, Nitro Card for EBS, Nitro Card for Instance Storage) | Dedicated hardware (custom AWS-designed offload cards, based on ASICs/FPGAs) that handle networking (VPC packet processing, encapsulation, security group enforcement) and storage I/O (EBS block storage protocol, NVMe translation) **entirely outside the host CPU**. |
| **Nitro Security Chip** | A hardware root of trust embedded on the motherboard. Continuously monitors and validates firmware, and prevents unauthorized access to hardware — including protecting against tampering by AWS operators themselves. Enforces the guarantee that AWS personnel cannot access customer instance memory or data. |
| **Nitro Hypervisor** | A stripped-down, KVM-derived hypervisor with a minimal software footprint. Its *only* remaining jobs are CPU and memory virtualization (VM entry/exit via VT-x/AMD-V extended page tables). All I/O virtualization has been moved to the Nitro Cards, so the hypervisor itself has near-zero involvement in the data path. |

### 2.3 Consequence: Nitro instances vs Xen instances

| Property | Xen-based instances (legacy) | Nitro-based instances (current generation) |
|---|---|---|
| I/O path | Emulated in software (Xen dom0), consumes host CPU | Offloaded to dedicated Nitro Card hardware, bypasses host CPU almost entirely |
| Hypervisor overhead | Measurable (~10–30% depending on workload) | Near bare-metal (<1%) — AWS advertises that nearly 100% of server resources can be sold as customer-usable |
| Bare metal support | Not possible (hypervisor is software-resident on the CPU) | Possible — `.metal` instance types give customers direct hardware access with **no hypervisor at all** on the CPU, since I/O still flows through the Nitro Cards |
| Security boundary | Enforced by software hypervisor | Enforced partly in hardware (Nitro Security Chip), reducing attack surface |

**Exam-relevant point:** Nitro is a concrete, production example of hardware/software co-designed virtualization — it directly answers "how do you reduce hypervisor overhead" from your hypervisor unit. It is functionally similar in spirit to SR-IOV (Single Root I/O Virtualization) taken to its logical extreme: I/O device virtualization moved fully into hardware.

---

## 3. The EC2 Instance Lifecycle

### 3.1 Instance states (finite state machine)

An EC2 instance is formally a finite-state object. The valid states and legal transitions:

```
pending → running → stopping → stopped → (pending on restart)
                  ↘ shutting-down → terminated
running → rebooting → running   (no state change in billing; instance keeps its instance ID, private IP)
stopped → shutting-down → terminated
```

| State | Meaning | Billed for compute? | Billed for EBS storage? |
|---|---|---|---|
| `pending` | Instance is being launched (AMI copy, network attach) | No | Yes (from volume creation) |
| `running` | Instance is executing | Yes | Yes |
| `stopping` | Graceful shutdown in progress (EBS-backed only) | No (partial second, rounds down) | Yes |
| `stopped` | Instance halted, EBS root volume persisted, instance retains instance ID, private IP (may lose public IP unless Elastic IP) | No | Yes |
| `shutting-down` | Termination in progress | No | Depends on `DeleteOnTermination` flag |
| `terminated` | Instance permanently destroyed, instance ID retained in metadata for ~1 hour then purged | No | Only if `DeleteOnTermination=false` volumes remain |

**Critical distinction — Stop vs Terminate:**
- **Stop** is only possible for **EBS-backed** instances (root volume on EBS, not instance store). CPU/memory state is lost, but disk state on the root EBS volume persists. You are not billed for compute-hours while stopped.
- **Terminate** destroys the instance permanently. By default the root EBS volume is deleted with it (`DeleteOnTermination=true` by default for the root volume; false by default for additional attached volumes).
- Instance-store-backed instances **cannot be stopped** — only terminated — because their data lives on physical disk directly attached to the host, which is wiped and reassigned to another customer the moment the instance is not running there.

### 3.2 What happens under the hood on `RunInstances`

1. API call `RunInstances` hits the EC2 control plane.
2. Control plane selects a physical host in the target Availability Zone with capacity matching the requested instance type (this is the internal AWS placement/scheduling algorithm — not customer-visible).
3. The chosen AMI (Amazon Machine Image) is used to materialize the root volume:
   - If **EBS-backed**: a new EBS volume is created from the AMI's backing EBS snapshot (copy-on-write — the block device is lazily hydrated from S3-backed snapshot storage on first read).
   - If **instance-store-backed**: the AMI is streamed from S3 directly onto the physical host's local disk.
4. Nitro Card for VPC allocates an Elastic Network Interface (ENI) with a private IP from the subnet's CIDR block, and applies the attached Security Group rules at the hardware level.
5. Nitro hypervisor performs a VM entry, guest OS boots, user-data script (if provided via `--user-data`) executes at first boot via cloud-init.
6. Instance transitions `pending → running`.

---

## 4. AMIs (Amazon Machine Images) — Technical Detail

An AMI is a template consisting of:

| Component | Detail |
|---|---|
| Root volume template | A snapshot (EBS) or a template image (instance store) containing OS + preinstalled software |
| Launch permissions | Controls which AWS accounts may launch instances from this AMI (private / public / shared with specific account IDs) |
| Block device mapping | Maps volumes (root + additional) to device names (`/dev/xvda`, `/dev/sdf`, etc.) and specifies volume type, size, `DeleteOnTermination` per volume |
| Virtualization type | `hvm` (Hardware Virtual Machine — the only type supported on current-generation instances; uses full hardware virtualization via VT-x/AMD-V) — the older `paravirtual (PV)` type is legacy and unsupported on Nitro |
| Architecture | x86_64, i386 (legacy), or arm64 (for AWS Graviton processor instances) |

AMI sources:
- **AWS-provided** (Amazon Linux 2/2023, Windows Server, etc.)
- **AWS Marketplace** (vendor-published, often with additional per-hour software licensing cost layered on top of instance cost)
- **Community AMIs**
- **Custom AMIs**: created by the customer via `CreateImage` from a running/stopped instance — this snapshots the current EBS root volume state, allowing "golden image" workflows (bake once, launch many identical instances — critical for Auto Scaling, covered in Section 9).

---

## 5. Instance Type Nomenclature and Families

Instance type strings follow a strict grammar:

```
[family][generation][additional capability flags].[size]

Example: m6i.2xlarge → family=m (general purpose), generation=6, capability flag=i (Intel), size=2xlarge
Example: c6gn.4xlarge → family=c (compute optimized), generation=6, flags=g(Graviton ARM)+n(network optimized), size=4xlarge
```

| Flag | Meaning |
|---|---|
| `a` | AMD processor |
| `g` | AWS Graviton (ARM64) processor |
| `i` | Intel processor |
| `n` | Network-optimized (enhanced networking bandwidth) |
| `d` | NVMe instance store (local SSD) included |
| `e` | Extra storage or extra memory variant |
| `z` | High frequency variant (e.g. z1d) |
| `metal` | Bare metal — no hypervisor layer for CPU/memory virtualization |

### 5.1 Instance Family Taxonomy

| Family | Optimized for | Representative types | vCPU:Memory ratio (approx) | Typical workload |
|---|---|---|---|---|
| **General Purpose** | Balanced compute/memory/network | T3, T4g, M5, M6i, M6g, M7g | 1:4 (GiB per vCPU) | Web servers, small-medium DBs, dev/test |
| **Burstable General Purpose** | Baseline CPU + burst credits | T2, T3, T3a, T4g | 1:4 | Variable-load workloads, microservices, low-traffic web apps |
| **Compute Optimized** | High vCPU:memory ratio, high clock | C5, C6i, C6g, C7g | 1:2 | Batch processing, scientific modeling, ad serving, gaming servers, HPC front-ends |
| **Memory Optimized** | Large RAM per vCPU | R5, R6i, R6g, X2gd, High Memory (u-*) | 1:8 (R), up to 1:16+ (X) | In-memory DBs (Redis, SAP HANA), large-scale caching |
| **Storage Optimized** | High sequential IOPS / local NVMe throughput | I3, I4i, D3, D3en, H1 | Variable | NoSQL DBs (Cassandra, MongoDB), data warehousing, distributed file systems |
| **Accelerated Computing** | GPU / custom silicon | P4, P5, G5, Trn1, Inf2, F1 | Variable | Deep learning training (P/Trn), inference (Inf), graphics/ML (G), FPGA workloads (F1) |
| **HPC Optimized** | Tightly-coupled cluster networking (EFA) | Hpc6a, Hpc7g | High vCPU count | MPI-based simulations, CFD, weather modeling |

### 5.2 Size scaling within a family

Sizes scale in a roughly geometric progression: `nano, micro, small, medium, large, xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge, 24xlarge, metal`. Each step (from `large` upward) approximately **doubles** vCPU count and memory, while the vCPU:memory *ratio* for a given family stays constant. This is a useful exam relationship:

$$\text{vCPU}(size_{n+1}) \approx 2 \times \text{vCPU}(size_n), \quad \text{Memory}(size_{n+1}) \approx 2 \times \text{Memory}(size_n)$$

---

## 6. Burstable Performance Instances (T-family) — CPU Credit Model

This is the most *quantitatively rich* topic in EC2 and is well suited to numerical problems.

### 6.1 The model

T-family instances (T2/T3/T3a/T4g) are priced for a **baseline** CPU performance level (a fraction of a full vCPU core, e.g., 20% for `t3.medium`), and accumulate **CPU credits** during idle/low-usage periods that can be spent to **burst** above the baseline when demand spikes.

**1 CPU credit = 1 vCPU running at 100% utilization for 1 minute** (equivalently, 1 full vCPU-minute).

### 6.2 Governing formulas

Let:
- $B$ = baseline utilization (as a fraction, e.g., 0.20 for 20%)
- $n$ = number of vCPUs
- $t$ = elapsed time in minutes
- $C_{max}$ = maximum credit balance (cap, varies by instance size)
- $U$ = actual CPU utilization used (as a fraction)

**Credit earn rate (per minute):**
$$\text{Credits earned per minute} = B \times n$$

**Credit balance evolution (discrete, per minute):**
$$C_{t+1} = \min\big(C_{max},\ C_t + (B \times n) - (U_t \times n)\big)$$

- If $U_t < B$: instance is earning net credits (surplus banked, up to $C_{max}$).
- If $U_t > B$: instance is spending banked credits to sustain the burst.
- If $U_t = B$: credit balance is unchanged (steady state).

**Time to exhaustion under sustained burst** (once $U > B$ and $C_t$ known):
$$t_{exhaust} = \frac{C_t}{(U - B)\times n}$$

Once credits are exhausted:
- **Standard mode**: CPU is throttled back down to the baseline $B$ — hard ceiling, no further burst.
- **Unlimited mode** (T3/T3a/T4g support this; T2 does not): instance may continue bursting above baseline, but AWS charges an additional per-vCPU-hour **surplus credit charge** for credits spent beyond what was earned in a rolling 24-hour window.

### 6.3 Worked example (representative of what to set as a tutorial problem)

`t3.medium`: 2 vCPUs, baseline 20% per vCPU → $B \times n = 0.20 \times 2 = 0.40$ credits/minute earned at idle, i.e., 24 credits/hour at full idle.

If a `t3.medium` starts with $C_0 = 144$ credits (its baseline credit cap) and is suddenly driven to 100% CPU utilization ($U=1.0$) on both vCPUs:

$$\text{Net burn rate} = (U - B)\times n = (1.0 - 0.20)\times 2 = 1.6 \text{ credits/minute}$$

$$t_{exhaust} = \frac{144}{1.6} = 90 \text{ minutes}$$

After 90 minutes of sustained 100% load, the instance (in Standard mode) is throttled back to 40% aggregate (20% per vCPU).

This type of problem — given baseline %, vCPU count, starting credit balance, and target utilization, compute time-to-throttle, or given a usage pattern over a day compute end-of-day credit balance — is directly reusable as a numerical assignment question.

---

## 7. EC2 Storage Architecture

### 7.1 Instance Store vs EBS — fundamental distinction

| Property | Instance Store | Elastic Block Store (EBS) |
|---|---|---|
| Physical location | Local NVMe SSD physically attached to the host server | Network-attached block storage, physically separate from the compute host, replicated within an AZ |
| Persistence | **Ephemeral** — data lost on stop, terminate, or underlying hardware failure | **Persistent** — survives stop/start; deleted only on explicit deletion or `DeleteOnTermination=true` at terminate |
| Performance | Very high IOPS/throughput, extremely low latency (no network hop) | High but network-bound; performance is a provisioned, billed parameter |
| Snapshot capability | No native snapshot | Yes — point-in-time snapshots to S3 (incremental) |
| Resizability | Fixed at launch (instance type determines instance store size) | Elastically resizable (`ModifyVolume`) without downtime |
| Use case | Cache, buffer, scratch space, replicated/shardable data (e.g., one node in a Cassandra ring) | Boot volumes, databases, any data requiring durability |

### 7.2 EBS Volume Types — the quantitative core

| Volume type | Media | Use case | Max IOPS | Max Throughput | IOPS:GiB ratio | Notes |
|---|---|---|---|---|---|---|
| **gp3** (General Purpose SSD, current gen) | SSD | Default boot volumes, general workloads | 16,000 (baseline 3,000 included free) | 1,000 MiB/s (baseline 125 MiB/s included free) | Decoupled from size — IOPS and throughput provisioned independently | Recommended default; cheaper than gp2 for equivalent performance |
| **gp2** (General Purpose SSD, legacy) | SSD | Legacy boot volumes | 16,000 | 250 MiB/s | 3 IOPS per GiB baseline, burst to 3,000 | Burst-credit bucket model (analogous to T-family CPU credits!) |
| **io2 Block Express** | SSD | Mission-critical DBs (Oracle, SAP HANA, high-IOPS OLTP) | 256,000 | 4,000 MiB/s | Up to 500 IOPS/GiB | 99.999% durability (highest of any EBS type), sub-millisecond latency |
| **io1/io2** | SSD | High-IOPS DBs | 64,000 | 1,000 MiB/s | Up to 50 IOPS/GiB (io1), 500 (io2) | Provisioned IOPS billed separately from capacity |
| **st1** (Throughput Optimized HDD) | HDD | Big data, log processing, data warehouses | N/A (throughput-based) | 500 MiB/s | — | Cannot be a boot volume |
| **sc1** (Cold HDD) | HDD | Infrequently accessed data, lowest cost | N/A | 250 MiB/s | — | Cannot be a boot volume |

**gp2 burst-credit formula** (structurally identical to the T-family CPU credit model above, note the pedagogical parallel worth drawing for students):

$$\text{Baseline IOPS} = 3 \times \text{Volume Size (GiB)}, \quad \text{minimum } 100, \quad \text{maximum } 16{,}000$$

A 100 GiB gp2 volume: baseline IOPS $= 3 \times 100 = 300$ IOPS, with the ability to burst to 3,000 IOPS by drawing down an I/O credit bucket (max bucket size 5.4 million credits), replenished at the baseline rate when usage is below baseline — exactly analogous to CPU credits, making this an easy "compare and contrast the two credit systems" exam question.

### 7.3 EBS Multi-Attach and Snapshots

- **Multi-Attach** (io1/io2 only): a single EBS volume can be attached to up to 16 Nitro instances simultaneously within the same AZ — requires cluster-aware filesystem (e.g., not ext4/NTFS without a coordination layer) since EBS does not itself provide write locking.
- **Snapshots**: incremental, block-level, stored in S3 (not directly visible/billed as S3 though). First snapshot = full copy; subsequent snapshots store only changed blocks since the last snapshot, but are logically "full" restore points (AWS manages the block delta chain internally, not exposed to the user).

---

## 8. Networking: ENI, Security Groups, and the Nitro Data Path

### 8.1 Elastic Network Interface (ENI)

An ENI is a virtual network card, the fundamental network attachment unit in EC2. Each ENI has:
- One primary private IPv4 address (+ optionally secondary private IPs)
- One MAC address
- Association with exactly one subnet, hence one Availability Zone
- One or more Security Groups attached
- Optionally, one Elastic IP or public IPv4 address

Instances can have multiple ENIs attached (count varies by instance type — larger instances support more ENIs, enabling multi-homed network configurations, e.g., separating management traffic from data traffic).

**Enhanced Networking**: Nitro-based instances use the **ENA (Elastic Network Adapter)** driver, exposing the network device directly to the guest OS via SR-IOV-like hardware passthrough (physically implemented by the Nitro Card for VPC), achieving up to 100 Gbps (up to 200 Gbps on select instances) with minimal CPU overhead — again a direct continuation of your hardware-assisted virtualization discussion.

### 8.2 Security Groups vs Network ACLs (NACLs) — comparison table

| Property | Security Group | Network ACL |
|---|---|---|
| Operates at | Instance/ENI level | Subnet level |
| State | **Stateful** (return traffic automatically allowed regardless of outbound rules) | **Stateless** (return traffic must be explicitly allowed by a matching rule) |
| Rule types | Allow rules only | Allow AND explicit Deny rules |
| Rule evaluation | All rules evaluated; if any rule matches, traffic allowed | Rules evaluated **in numeric order**, first match wins (lower rule number = higher priority) |
| Default behavior | Deny all inbound, allow all outbound (default SG) | Allow all traffic (default NACL) |
| Enforcement point | At the Nitro Card (hardware), effectively zero-overhead firewall | At the subnet boundary (VPC router level) |

Security Group enforcement being pushed into Nitro Card hardware means packet filtering does not consume guest OS CPU cycles — another direct Nitro System benefit.

---

## 9. Placement Groups

Placement groups control the physical placement strategy of instances relative to underlying hardware, directly affecting latency and fault-isolation characteristics.

| Placement Group Strategy | Physical behavior | Benefit | Risk / Constraint |
|---|---|---|---|
| **Cluster** | All instances packed onto hardware within a single AZ, on the same low-latency, high-bandwidth network spine | Lowest possible inter-instance latency, highest throughput (useful for tightly-coupled HPC/MPI workloads) | Single point of failure — a single hardware/power/network fault can take down the whole group; recommended only within one AZ |
| **Spread** | Each instance placed on **distinct** underlying hardware (distinct racks, distinct power sources, distinct network) | Maximum fault isolation — failure of one rack does not affect others | Hard limit of 7 running instances per AZ per spread group |
| **Partition** | Instances divided into logical partitions (up to 7 per AZ), each partition on separate hardware racks; instances **within** a partition may share hardware | Balances fault isolation with scale — used by distributed systems like Hadoop/Cassandra/Kafka that have rack-awareness built into their own replication logic | Coarser isolation than "Spread" |

---

## 10. Pricing Models — Quantitative Comparison

| Model | Commitment | Discount vs On-Demand | Best for |
|---|---|---|---|
| **On-Demand** | None | Baseline (0%) | Unpredictable, short-term, spiky workloads |
| **Reserved Instances (Standard)** | 1 or 3 years | Up to ~72% | Steady-state, predictable workloads (fixed instance family/size/AZ or region) |
| **Reserved Instances (Convertible)** | 1 or 3 years | Up to ~54% | Predictable workloads where instance family may need to change over the term |
| **Savings Plans (Compute)** | 1 or 3 years, $/hour spend commitment | Up to ~66% | Flexible — applies automatically across instance family, size, OS, region, and even to Fargate/Lambda |
| **Spot Instances** | None (interruptible) | Up to ~90% | Fault-tolerant, stateless, or checkpointable batch workloads |

### 10.1 Spot Instance mechanics

Spot instances draw from AWS's **unused capacity pool**. Pricing is set by AWS based on long-term supply/demand trends for that instance type in that AZ (no longer a live bidding auction as it was pre-2017 — this is a common outdated misconception worth explicitly correcting for students).

An instance is reclaimed (a **Spot interruption**) when AWS needs the capacity back. Sequence:
1. AWS sends a **Spot interruption notice** via the instance metadata service (`http://169.254.169.254/latest/meta-data/spot/instance-action`) and an EventBridge event.
2. The customer's application has a **2-minute warning window** to checkpoint state, drain connections, or migrate work before reclamation.
3. Instance is terminated (or stopped/hibernated, depending on the configured interruption behavior).

**Effective cost saving formula:**
$$\text{Savings \%} = \left(1 - \frac{\text{Spot Price}}{\text{On-Demand Price}}\right) \times 100$$

This is a natural numerical exercise: given On-Demand price and observed Spot price, compute savings %; given a workload's total compute-hour requirement and expected interruption rate, compute expected effective cost including redo work from interruptions.

---

## 11. Auto Scaling Groups (ASG)

An ASG maintains a **desired capacity** of instances within a defined `[MinSize, MaxSize]` range, automatically launching (from a Launch Template, which references an AMI + instance type + security groups) or terminating instances to match load.

### 11.1 Scaling policy types

| Policy type | Mechanism | Formula/logic |
|---|---|---|
| **Target Tracking** | Maintains a metric (e.g., average CPU utilization) at a target value, analogous to a thermostat / PID-style feedback controller | $\text{Desired capacity} \propto \frac{\text{Current metric value}}{\text{Target value}} \times \text{Current capacity}$ (AWS computes this internally and adjusts) |
| **Step Scaling** | Defines multiple step adjustments based on how far the metric breaches the alarm threshold | e.g., CPU 70–80% → +1 instance; CPU 80–90% → +2 instances; CPU >90% → +4 instances |
| **Simple Scaling** | Single scaling adjustment per alarm breach, followed by a cooldown period before any further scaling action | Legacy; largely superseded by Step and Target Tracking |
| **Scheduled Scaling** | Time-based, sets min/max/desired at specific calendar times | Useful for known load cycles (e.g., business-hours traffic patterns) |
| **Predictive Scaling** | ML-based (uses historical CloudWatch data) forecasting to pre-provision capacity ahead of anticipated load | Combines forecasted + dynamic scaling |

### 11.2 Health checks and replacement

ASG continuously performs health checks (EC2 status checks, and optionally ELB health checks). An unhealthy instance is automatically terminated and replaced — this is the mechanism that gives Auto Scaling Groups their **self-healing** property, independent of load-based scaling.

---

## 12. Elastic Load Balancing (ELB) Integration

| Load Balancer type | OSI Layer | Use case |
|---|---|---|
| **Application Load Balancer (ALB)** | Layer 7 (HTTP/HTTPS) | Content-based routing (path/host-based), microservices, container workloads |
| **Network Load Balancer (NLB)** | Layer 4 (TCP/UDP/TLS) | Ultra-high throughput, low latency, static IP requirement, millions of requests/sec |
| **Gateway Load Balancer (GWLB)** | Layer 3 (network layer, GENEVE encapsulation) | Transparent insertion of third-party virtual appliances (firewalls, IDS/IPS) into the traffic path |
| **Classic Load Balancer (CLB)** | Layer 4/7 (legacy) | Deprecated for new designs; retained for legacy EC2-Classic-era workloads |

---

## 13. Identity and Access — Instance Profiles

EC2 instances should never embed long-term IAM access keys inside the OS or application code. Instead:

1. An **IAM Role** is created with a policy defining permitted API actions (e.g., read access to a specific S3 bucket).
2. An **Instance Profile** (a thin wrapper container for the role) is attached to the EC2 instance at launch.
3. The Nitro-exposed **Instance Metadata Service (IMDS)** at `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>` serves **temporary, automatically-rotated credentials** to any process running on the instance.

**IMDSv2** (token-based, session-oriented, mandatory on new accounts by default since 2024) closes a known SSRF (Server-Side Request Forgery) attack class present in IMDSv1, where an attacker exploiting an application-layer vulnerability could directly GET the metadata endpoint and exfiltrate credentials without any additional authentication. IMDSv2 requires a PUT request first to obtain a session token (with a TTL), and every subsequent metadata GET must carry that token in a header — this defeats simple SSRF because most SSRF-vulnerable applications can only be tricked into issuing GET requests, not the required PUT.

---

## 14. Monitoring — CloudWatch Integration

| Monitoring tier | Metric granularity | Cost |
|---|---|---|
| **Basic Monitoring** | 5-minute intervals | Free, enabled by default |
| **Detailed Monitoring** | 1-minute intervals | Paid, must be explicitly enabled |

Default EC2 metrics (host-level, collected by the hypervisor/Nitro layer without needing an agent): `CPUUtilization`, `NetworkIn/Out`, `DiskReadOps/WriteOps` (instance store only), `StatusCheckFailed` (composite of System Status Check — underlying hardware/network — and Instance Status Check — guest OS-level reachability).

**Note**: Memory utilization and disk space utilization inside the guest OS are **not** available by default (the hypervisor cannot see inside guest OS memory without cooperation) — these require the **CloudWatch Agent** installed inside the guest, publishing custom metrics.

---

## 15. Suggested Numerical / Formula-Based Problem Bank

For assignment or exam use, structured the way you've preferred for algorithmic problems (formula application over step tracing):

1. **CPU Credits**: Given instance type baseline %, vCPU count, and a piecewise utilization schedule over 3 hours, compute the ending credit balance and identify if/when throttling occurs.
2. **gp2 IOPS**: Given a required sustained IOPS target, compute the minimum gp2 volume size needed (inverse of the $3\times\text{GiB}$ formula), and compare cost/performance against provisioning the same IOPS on gp3.
3. **Spot savings**: Given On-Demand price, Spot price, and an expected interruption rate (interruptions/hour) with a known checkpoint overhead (minutes lost per interruption), compute expected effective hourly cost of a Spot-based batch job versus On-Demand.
4. **Auto Scaling target tracking**: Given current instance count, current average CPU%, and target CPU%, compute the ASG's next desired capacity (rounding rules: AWS rounds up).
5. **Reserved Instance breakeven**: Given On-Demand hourly rate, Reserved upfront + hourly rate, and 1-year term, compute the breakeven utilization percentage / breakeven number of hours at which Reserved becomes cheaper than On-Demand.

---

## 16. Summary Table — EC2 Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Hardware/hypervisor | Nitro System | Hypervisor tax minimized by offloading I/O to dedicated hardware cards; security enforced by a hardware root of trust |
| Compute | Instance types/families | Naming grammar encodes family, generation, capability, size; families trade off vCPU:memory:storage:network ratios |
| Compute economics | T-family credits | Baseline + burst, governed by a credit accumulation/depletion linear model — directly analogous to gp2 IOPS credits |
| Storage | EBS vs Instance Store | Persistent+network-attached vs ephemeral+local; EBS type selection is an IOPS/throughput/cost optimization problem |
| Network | ENI + Security Groups | Stateful, hardware-enforced, instance-level firewalling via Nitro Card |
| Placement | Placement Groups | Explicit control over the latency vs fault-isolation tradeoff |
| Economics | Pricing models | On-Demand (flexibility) → Reserved/Savings Plans (commitment discount) → Spot (steep discount, interruption risk) |
| Elasticity | Auto Scaling Groups | Policy-driven desired-capacity control loop + self-healing via health checks |
| Security | IAM Instance Profiles + IMDSv2 | Temporary, auto-rotated credentials; IMDSv2 token requirement closes an SSRF credential-theft class |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 15 reserved as a follow-up tutorial/assignment.*
