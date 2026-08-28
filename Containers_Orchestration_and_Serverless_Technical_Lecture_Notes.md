# Containers, Orchestration & Serverless — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Companion document to: "Amazon EC2", "Hypervisors", "Cloud Networking (VPC Deep Dive)", and "Cloud Storage & Database Services" — Technical Lecture Notes
### Prerequisite recap assumed: Hypervisor CPU/memory/I/O virtualization (Hypervisor notes §3–5), Nitro System (EC2 notes §2), Auto Scaling Groups (EC2 notes §11), ENI (EC2/VPC notes), EFS (Storage notes §3)

---

## 1. Positioning Containers in the Virtualization Stack

Every document so far in this set has covered a form of **resource isolation**: the Hypervisor notes covered isolating one physical machine into multiple **virtual machines**; the EC2 notes covered turning that mechanism into an elastic, billable **service**. This document covers a **second, fundamentally different isolation mechanism** — **OS-level virtualization (containers)** — and the orchestration/serverless layers AWS built on top of it.

Formally, extending the layering diagram from the Hypervisor and EC2 notes:

```
Physical Data Center (AWS-owned hardware, Regions/AZs)
        |
Nitro System (hardware offload + lightweight hypervisor)   <-- hypervisor layer
        |
EC2 instance (a full guest OS kernel, isolated via VT-x/AMD-V + EPT)   <-- VM-level isolation (Hypervisor notes §3-4)
        |
Container runtime (containerd/Docker) running INSIDE the guest OS      <-- OS-level isolation (THIS DOCUMENT, §2)
        |
Orchestrator (ECS / Kubernetes via EKS) — schedules containers across many hosts   <-- control plane (THIS DOCUMENT, §5-8)
        |
Fargate / Lambda — the orchestrator's host layer itself becomes a managed abstraction  <-- serverless compute (THIS DOCUMENT, §9-12)
```

**The central conceptual thread of this document**: each layer above hides the layer below it, exactly as EC2 hides the hypervisor and the hypervisor hides physical hardware. Containers hide the kernel; Fargate hides the EC2 instance a container runs on; Lambda hides the container/runtime altogether. This is the same "abstraction stacked on abstraction, each backed by a narrower and more automated control plane" pattern that has organized every document in this series.

---

## 2. Containers — Formal Definition and OS-Level Virtualization Mechanism

### 2.1 Why containers exist — the architectural gap they fill

A VM (Hypervisor notes §1–4) provides **strong isolation** (a full guest kernel, its own memory space via EPT/NPT, its own virtualized devices) but pays for that isolation with **overhead**: each VM needs its own full OS kernel, its own boot process (tens of seconds), and a memory/disk footprint measured in hundreds of MB to GBs even before an application starts. For workloads that need **many, short-lived, densely-packed, mutually-isolated processes** — the microservices pattern — VM-per-service is often too heavyweight: booting a new VM to scale out a stateless web service takes tens of seconds and consumes a fixed OS-kernel memory tax per instance.

**Containers solve this by moving the isolation boundary from the hardware/CPU layer (hypervisor) to the OS kernel layer.** Instead of virtualizing hardware to run multiple kernels, a container runtime uses **a single shared host kernel** and isolates groups of processes from each other using kernel-native mechanisms — no second kernel, no boot process, no hardware virtualization involved at all.

### 2.2 The two Linux kernel primitives that make containers possible

| Primitive | What it isolates | Mechanism |
|---|---|---|
| **Namespaces** | **Visibility** — what a process can *see* | Each namespace type (PID, NET, MNT, UTS, IPC, USER) gives a process group its own view of a global kernel resource: a container's PID namespace makes its own init process appear as PID 1, while the host sees it as an ordinary high-numbered PID; a NET namespace gives a container its own virtual network stack (interfaces, routing table, iptables rules) |
| **cgroups (control groups)** | **Consumption** — how much of a shared resource a process may *use* | Enforces hard/soft limits on CPU shares, memory, block I/O, and network bandwidth per process group, and provides per-group usage accounting — this is the direct Linux-kernel analogue of the Hypervisor notes' vCPU scheduling and memory overcommitment discussion (Hypervisor notes §4.3, §6), except enforced by the **host OS kernel's own scheduler**, not a hypervisor's vCPU scheduler |

**Formal statement**: a container is **not** a lightweight VM. It is an **ordinary host process (or process group)**, made to *believe* it has its own filesystem, network stack, and process tree via namespaces, and *restricted* in its resource consumption via cgroups. There is exactly **one kernel** shared by the host and every container on it — this is the single most important distinction from a VM, and the reason containers boot in milliseconds rather than tens of seconds: there is no second kernel to boot at all.

### 2.3 VM vs Container — the formal comparison, extending Hypervisor notes §1

| Property | Virtual Machine | Container |
|---|---|---|
| Isolation boundary | Hardware (via hypervisor + VT-x/AMD-V + EPT, Hypervisor notes §3–4) | OS kernel (via namespaces + cgroups) |
| Kernel | Each VM runs its **own** full kernel | **Shared** host kernel across all containers |
| Popek-Goldberg applicability | Directly applicable — a hypervisor is a VMM in the formal sense (Hypervisor notes §1) | Not applicable in the same sense — a container runtime is not virtualizing an instruction set, only namespacing kernel resources |
| Startup time | Seconds to tens of seconds (full kernel boot) | Milliseconds (only the application process starts; no kernel boot) |
| Base image size | Hundreds of MB – several GB (full guest OS) | MBs – tens of MB (application + minimal userland; no kernel included) |
| Isolation strength | Strong — a VM escape is a serious, rare vulnerability class (Hypervisor notes §8.1) | Weaker by default — a kernel vulnerability or namespace escape can compromise the whole host, since **all containers share the exact same kernel attack surface** |
| Cross-platform kernel | Guest can run a **different** OS/kernel than the host (e.g., Windows guest on a Linux-hosted hypervisor) | Container **must** share the host's kernel — a Linux container cannot run natively on a Windows kernel or vice versa without a compatibility layer |
| Resource overhead per instance | Fixed kernel memory/CPU tax per VM | Near-zero fixed tax; overhead scales with the application itself |

### 2.4 Why this means containers usually run *inside* VMs in the cloud (not instead of them)

Because container isolation is weaker than VM isolation (§2.3), and because a cloud provider must isolate **mutually distrusting customers** from each other, AWS does not run different customers' containers as bare processes on a shared host kernel. Instead, the standard cloud pattern is a **two-level isolation stack**: the hypervisor (Nitro) provides the strong, hardware-enforced tenant-to-tenant isolation boundary, and containers provide **cheap, fast, dense multiplexing *within* a single tenant's own VM (or Fargate's per-task micro-VM, §9.2)**. This is precisely the reasoning behind Fargate's architecture in §9 — it is not "containers replacing VMs," but "one more automated layer of VM provisioning hidden underneath the container abstraction."

---

## 3. Docker — Container Image and Runtime Model

### 3.1 Image layering — a copy-on-write filesystem, structurally parallel to EBS snapshots

A Docker image is built from a sequence of **read-only layers**, each corresponding to one instruction in the Dockerfile (`FROM`, `RUN`, `COPY`, etc.). When a container is started from an image, the runtime adds one **thin, writable layer** on top (via a union filesystem, e.g., OverlayFS), and any file the running container modifies is copy-on-write duplicated up into that top writable layer, leaving the underlying image layers untouched and shareable across every container started from the same image.

This layered, copy-on-write, incremental model is the **same architectural pattern** as EBS snapshots (EC2 notes §7.3) and S3/RDS incremental backups (Storage notes §9.1): a full base state, plus a chain of deltas, materialized lazily — the specific mechanism (union filesystem vs block-level snapshot chain) differs, but the underlying "store deltas, not full copies, and reconstruct the full state on demand" reasoning recurs across nearly every AWS storage abstraction covered in this document series.

### 3.2 Image registry — Amazon ECR

**Amazon ECR (Elastic Container Registry)** is AWS's managed Docker/OCI image registry — the storage and distribution service for container images, playing the same role for container images that **S3 plays for arbitrary objects** (Storage notes §2): both are durable, versioned, access-controlled, Regional storage for immutable artifacts, and ECR is in fact **backed by S3** for its underlying layer storage, making it a direct, literal application of the object storage concepts already covered.

---

## 4. Container Runtime and Orchestration — Why Orchestration Is Needed at All

A single Docker host can run many containers, but production systems need answers to questions a single host cannot provide on its own: *which* host should a new container run on (bin-packing/scheduling), what happens when a host or container dies (self-healing, echoing EC2 notes §11.2's ASG health-check discussion), how do containers discover and reach each other across hosts (service discovery, load balancing), and how are rolling updates performed without downtime. An **orchestrator** is the control plane that answers these questions — structurally the same role the **EC2 control plane** plays for VM placement (EC2 notes §3.2) and the same role a hypervisor's **vCPU scheduler** plays for a single host (Hypervisor notes §6), but at the level of *many hosts and many containers* rather than one host and many vCPUs.

---

## 5. Amazon ECS (Elastic Container Service)

### 5.1 Core objects

| ECS object | Definition | Rough VM-world analogue |
|---|---|---|
| **Task Definition** | A JSON/YAML blueprint: container image(s), CPU/memory allocation, networking mode, IAM role, environment variables | An AMI + launch configuration (EC2 notes §4, §11) |
| **Task** | A running instantiation of a Task Definition — one or more co-located containers sharing network namespace | A running EC2 instance |
| **Service** | Maintains a desired count of running Tasks, replacing failed ones automatically, and integrates with a Load Balancer | An Auto Scaling Group (EC2 notes §11) |
| **Cluster** | A logical grouping of the compute capacity (EC2 instances or Fargate) that Tasks run on | Analogous to a Region/AZ pool of available hosts |

### 5.2 ECS Launch Types — comparison table

| Property | EC2 Launch Type | Fargate Launch Type |
|---|---|---|
| Underlying host | Customer-managed EC2 instances (registered as **container instances** in the cluster) | Fully AWS-managed, no visible EC2 instance at all |
| Bin-packing responsibility | ECS scheduler places Tasks onto customer's EC2 instances; **customer must right-size the underlying instance fleet** | AWS provisions exactly the compute a Task needs, per-Task, no customer capacity planning |
| Billing granularity | Per EC2 instance-hour (regardless of container utilization — the same "you pay for the whole instance" model as raw EC2, EC2 notes §10) | Per-Task, billed by **vCPU-seconds and GB-seconds actually requested by the Task** (§9.1 formula) |
| Customer control | Full (SSH access to the host, custom AMIs, DaemonSets-equivalent host-level agents) | None (no host access at all — the host is a hidden abstraction) |
| Cold-start / scale-out latency | Bound by EC2 instance launch time if cluster capacity is insufficient (tens of seconds to minutes) | Task-level startup only (typically tens of seconds; no separate instance launch step visible to the customer) |
| Best for | Workloads needing GPU instances, custom AMIs, Spot cost optimization at the instance level, or maximizing bin-packing density to minimize cost | Workloads prioritizing zero infrastructure management, spiky/unpredictable load, or where instance-level bin-packing effort is not worth the operational savings |

---

## 6. Kubernetes — Formal Architecture

### 6.1 Why Kubernetes, given ECS already exists

ECS is an AWS-proprietary orchestrator. **Kubernetes (K8s)**, originated at Google (drawing on Google's internal Borg system) and open-sourced in 2014, is a **portable, vendor-neutral orchestration API** — the same workload definition (a set of YAML manifests) can, in principle, run unmodified on AWS (EKS), Google Cloud (GKE), Azure (AKS), or on-premises. This portability is the primary reason large organizations often choose Kubernetes over a cloud-proprietary orchestrator like ECS, at the cost of materially higher conceptual and operational complexity.

### 6.2 Control plane components

| Component | Role | Rough analogue elsewhere in this document set |
|---|---|---|
| **kube-apiserver** | The single entry point for all cluster state changes and queries — every other component talks *only* to the API server, never directly to each other | The EC2 control plane's API endpoint (EC2 notes §3.2) |
| **etcd** | A distributed, strongly consistent key-value store holding **all** cluster state (the single source of truth) | Structurally a quorum-replicated store — the same $R+W>N$ strong-consistency reasoning from Storage notes §8 applies to etcd's underlying Raft consensus protocol |
| **kube-scheduler** | Decides which **Node** a newly created **Pod** should run on, based on resource requests, affinity/anti-affinity rules, and taints/tolerations | The ECS scheduler (§5.1) and the EC2 control plane's host-placement algorithm (EC2 notes §3.2) |
| **kube-controller-manager** | Runs reconciliation control loops (e.g., the ReplicaSet controller continuously ensures the actual running Pod count matches the desired count) | The **same declarative control-loop pattern** as Auto Scaling Groups (EC2 notes §11) and S3 Lifecycle policies (Storage notes §2.3) |
| **kubelet** (on each Node) | The per-node agent that starts/stops containers per the scheduler's instructions and reports node/pod health back to the API server | The hypervisor's per-host role (Hypervisor notes §6) — a local enforcement agent taking instructions from a higher-level control plane |

### 6.3 Workload objects — the abstraction hierarchy

```
Pod            — smallest deployable unit; one or more tightly-coupled containers sharing a network namespace and IP
    |
ReplicaSet     — ensures N identical Pod replicas are always running (a reconciliation control loop)
    |
Deployment     — manages ReplicaSets to enable declarative, versioned rolling updates and rollbacks
    |
Service        — a stable virtual IP + DNS name load-balancing traffic across a dynamic set of Pods (Pods are ephemeral and get new IPs on every restart)
```

A **Pod**, not a container, is Kubernetes' smallest schedulable unit — this is a deliberate design choice mirroring the observation that some containers are tightly coupled enough (e.g., an application container plus a co-located logging "sidecar" container) that they should always be scheduled onto the same Node and share a network namespace, exactly as ECS's Task Definition allows multiple containers within a single Task (§5.1).

---

## 7. ECS vs EKS — Comparison Table

| Property | ECS | EKS (managed Kubernetes) |
|---|---|---|
| Control plane | AWS-proprietary, fully hidden, no separate billing for the control plane itself | Managed control plane (etcd, API server, etc.) — AWS runs it across multiple AZs for HA, billed **per cluster-hour** separately from worker compute |
| API / workload definition | ECS Task Definitions (AWS-specific JSON) | Kubernetes API objects (Pods, Deployments, Services) — portable across cloud providers |
| Ecosystem | Smaller, AWS-native tooling only | Massive open-source ecosystem (Helm charts, Prometheus/Grafana, service meshes like Istio, hundreds of CNCF projects) |
| Learning curve | Lower — fewer concepts, tightly integrated with other AWS services by default | Higher — full Kubernetes conceptual surface (namespaces, RBAC, CRDs, admission controllers, etc.) |
| Compute options | EC2 launch type or Fargate (§5.2) | EC2 (self-managed or Managed Node Groups) or **Fargate for EKS** |
| Networking | Uses standard ENI-per-Task (or awsvpc mode) | Uses the **VPC CNI plugin** (§14) to assign a real VPC-routable IP per Pod, an AWS-specific choice most other Kubernetes distributions do not make by default |
| Vendor lock-in | Higher (ECS concepts are AWS-specific) | Lower (workload manifests are largely cloud-portable) |
| Typical adopter profile | Teams fully committed to AWS, prioritizing simplicity and tight native-service integration | Teams needing multi-cloud portability, or already possessing Kubernetes expertise/tooling investment |

---

## 8. Autoscaling in the Container World — Extending EC2 notes §11

### 8.1 Horizontal Pod Autoscaler (HPA) — the Kubernetes analogue of Target Tracking

HPA adjusts the **replica count** of a Deployment based on an observed metric (commonly average CPU utilization per Pod), using a control formula that is the direct Kubernetes-world restatement of the EC2 notes' Target Tracking Auto Scaling formula (EC2 notes §11.1):

$$\text{desiredReplicas} = \left\lceil \text{currentReplicas} \times \frac{\text{currentMetricValue}}{\text{desiredMetricValue}} \right\rceil$$

This is, almost verbatim, the same proportional-control formula given for EC2 ASG Target Tracking in the EC2 notes — the "thermostat" control-loop pattern recurs identically at the Pod level.

### 8.2 Cluster Autoscaler / Karpenter — scaling the underlying Nodes

HPA scales **Pod count**, but if the underlying Nodes (EC2 instances) do not have enough spare capacity to schedule the new Pods, the Pods remain stuck in a `Pending` state. **Cluster Autoscaler** (and its more modern, faster-provisioning successor **Karpenter**) solves this second-order problem: it watches for unschedulable Pods and **adds EC2 instances** (or, conversely, removes underutilized ones) to the cluster's Node pool — structurally, this is Cluster Autoscaler triggering an **Auto Scaling Group** scale-out (EC2 notes §11) as its underlying mechanism, meaning container autoscaling in EKS is a **two-level control loop**: HPA (Pod-level) sitting on top of Cluster Autoscaler (Node/EC2-level), each independently reconciling toward its own target.

### 8.3 The bin-packing problem — formalized

Given $m$ Nodes each with capacity $(C_{cpu}, C_{mem})$ and $n$ Pods each requesting $(r_{cpu,i}, r_{mem,i})$, the scheduler must solve a **multi-dimensional bin-packing problem**: assign every Pod to some Node such that, for every Node $j$:

$$\sum_{i \in \text{Pods}(j)} r_{cpu,i} \leq C_{cpu}, \qquad \sum_{i \in \text{Pods}(j)} r_{mem,i} \leq C_{mem}$$

Bin-packing (in either dimension) is NP-hard in general; Kubernetes' scheduler and Cluster Autoscaler use greedy heuristics (e.g., "most allocated" or "least waste" scoring) rather than an optimal solver, meaning **real-world cluster utilization is always somewhat below the theoretical maximum** — the practical consequence of this is the numerical problem in §16.3.

---

## 9. AWS Fargate — Serverless Containers

### 9.1 Formal definition and billing model

Fargate removes the EC2 instance layer from container orchestration entirely: instead of the customer provisioning a fleet of EC2 instances for ECS/EKS to schedule Tasks/Pods onto (§5.2), **each Task/Pod runs on its own right-sized, isolated compute environment provisioned instantaneously by AWS**, billed by the **exact vCPU and memory requested by that Task**, per second (1-minute minimum).

$$\text{Fargate cost per Task-hour} = (\text{vCPU requested} \times \text{Rate}_{vCPU}) + (\text{Memory (GB) requested} \times \text{Rate}_{mem})$$

**Key economic contrast with EC2 launch type (§5.2)**: on the EC2 launch type, a customer pays for the **whole instance**, whether or not all of its capacity is used by scheduled Tasks (bin-packing efficiency directly determines cost-efficiency, per §8.3); on Fargate, the customer pays for **exactly** the CPU/memory each Task requests, with **no bin-packing responsibility or waste exposure at all** — AWS absorbs the bin-packing problem on its own infrastructure, at a materially higher **per-vCPU-hour** price than raw EC2, which is the standard "pay more per unit, in exchange for zero operational/capacity-planning burden" tradeoff seen repeatedly across this document series (NAT Gateway vs NAT Instance, VPC notes §5.2; RDS vs self-managed database on EC2, Storage notes §5.1).

### 9.2 Fargate's underlying isolation mechanism — the direct Nitro/Hypervisor connection

Fargate's per-Task isolation is **not** achieved via ordinary shared-kernel containers alone (§2.4's isolation concern) — each Fargate Task runs inside its own **lightweight micro-VM**, using **AWS Firecracker**, a purpose-built open-source VMM (Virtual Machine Monitor) that AWS developed specifically for this use case. Firecracker is architecturally the most direct possible bridge between this document and the Hypervisor notes:

- It is a **Type-1-adjacent hypervisor**, built on **KVM** (the same KVM discussed as an "architecturally hybrid" case in Hypervisor notes §2.1), stripped down to an extremely minimal device model — conceptually the **same "minimize the software attack surface, offload everything possible" design philosophy** as the Nitro Hypervisor (EC2 notes §2.2, Hypervisor notes §8.3).
- A Firecracker microVM boots in **under 125 milliseconds** and has a memory overhead of **under 5 MB per microVM**, achieved by discarding almost all of the legacy device emulation a general-purpose hypervisor like QEMU carries (Hypervisor notes §5, "full device emulation" row) — Firecracker implements only a minimal virtio block/net device model and nothing else.
- **Net effect**: Fargate gives every Task the **strong, hardware-virtualization-backed isolation of a VM** (satisfying the same isolation concern raised in §2.4) while still exposing the **fast-start, per-Task granularity of a container** to the customer — it resolves the VM-vs-container tradeoff table in §2.3 by making the VM boot time and memory tax small enough to be operationally indistinguishable from a container, rather than by weakening the isolation boundary.

---

## 10. AWS Lambda — Function-as-a-Service (FaaS)

### 10.1 Formal definition and the next abstraction step

Lambda removes not just the EC2 instance (as Fargate does) but the **container/Task lifecycle** itself from the customer's model entirely: the customer supplies **only a function** (a handler with a defined input event and output), and AWS is responsible for provisioning, invoking, scaling, and tearing down the execution environment per-request. This is the final step in the abstraction ladder opened in §1: EC2 (manage the OS) → Fargate (manage the container image, not the host) → Lambda (manage only application code, not even the container lifecycle).

### 10.2 Execution environment — also Firecracker-based

Lambda's execution environments are, like Fargate Tasks, isolated using **Firecracker microVMs** (§9.2) — this is the same underlying mechanism, exposed at a finer invocation granularity. This is a useful, concrete exam point: **Fargate and Lambda are not architecturally distinct virtualization technologies; they are the same Firecracker-microVM isolation mechanism, exposed at two different abstraction granularities** (Task/Pod-lifetime vs single-invocation-lifetime).

### 10.3 Cold start vs warm start

| State | What happens | Latency impact |
|---|---|---|
| **Cold start** | No existing execution environment for this function version is available; Lambda must provision a new Firecracker microVM, download/mount the function code and layers, initialize the language runtime, and run any code outside the handler (module-level imports, SDK client construction) before the handler itself executes | Added latency, ranging from tens of milliseconds (lightweight runtimes) to low seconds (large deployment packages, JVM-based runtimes, VPC-attached functions needing an ENI, §14) |
| **Warm start** | A previously-initialized execution environment is reused for a subsequent invocation (kept "warm" by AWS for a period after the last invocation, environment-dependent, typically minutes) | Minimal — only handler execution time |

**Provisioned Concurrency** pre-initializes a specified number of execution environments ahead of invocation traffic, eliminating cold starts for that pre-warmed pool at an additional standing hourly cost — directly analogous to paying for Reserved/On-Demand EC2 capacity ahead of need (EC2 notes §10) rather than relying on pure Spot-style on-demand provisioning.

### 10.4 Lambda concurrency model — the quantitative core

**Concurrency** = the number of in-flight (simultaneously executing) invocations of a function at any instant. Given:
- $\lambda$ = incoming request rate (requests/second)
- $d$ = average function duration (seconds)

By **Little's Law** (a general queueing-theory relationship, directly reusable from any prior queueing/throughput reasoning in this document set):

$$\text{Concurrent executions required} = \lambda \times d$$

**Worked example**: a function is invoked at $\lambda = 200$ requests/sec, with average duration $d = 300\text{ ms} = 0.3\text{ s}$:

$$\text{Concurrency required} = 200 \times 0.3 = 60 \text{ concurrent executions}$$

If the account/function's concurrency limit is below this figure, **throttling** occurs — this is structurally identical to the DynamoDB hot-partition throttling reasoning in the Storage notes (§7.2): a hard ceiling on a specific resource dimension causes request rejection even if *aggregate* account-level compute capacity is nowhere near exhausted.

### 10.5 Lambda billing formula

$$\text{Cost} = (\text{Number of requests} \times \text{Price per request}) + \left(\text{Memory allocated (GB)} \times \text{Duration (s)} \times \text{Price per GB-second}\right)$$

Memory allocation in Lambda **also determines proportional vCPU allocation** (AWS scales CPU share linearly with configured memory, up to a defined maximum), meaning increasing memory can, up to a point, **decrease** total billed cost by reducing duration $d$ enough to offset the higher per-GB-second rate — this produces a genuine optimization problem, formalized in the problem bank (§16.4).

---

## 11. Fargate vs Lambda — Comparison Table

| Property | Fargate | Lambda |
|---|---|---|
| Unit of deployment | A container image (Task/Pod) | A function handler (code + dependencies, up to a package size limit) |
| Execution lifetime | Long-running (as long as the Task/Service is desired to run) | Short-lived, per-invocation (max 15 minutes per invocation) |
| Underlying isolation | Firecracker microVM per Task | Firecracker microVM per execution environment |
| Scaling unit | Task count (via ECS/EKS Service desired count, or HPA) | Individual invocations, governed by concurrency (§10.4) |
| Billing granularity | Per-second, by vCPU/memory **requested** for the Task's full lifetime | Per-request + per-GB-second **actually consumed** during execution only |
| Idle cost | Non-zero if the Task is kept running with no traffic (billed continuously) | Zero — no charge when there are no invocations |
| State/long-lived connections | Well-suited (e.g., a Task can hold a persistent DB connection pool) | Poorly suited without extra care (each execution environment may be torn down; DB connection pooling requires patterns like RDS Proxy) |
| Best for | Steady-state or predictably-varying workloads, long-running processes, workloads needing full container control | Event-driven, bursty, or infrequent workloads; glue code between AWS services; workloads where near-zero idle cost matters most |

---

## 12. Container Networking — Direct Extension of the VPC Notes

### 12.1 The `awsvpc` network mode and VPC CNI — giving every Task/Pod a real VPC IP

By default, Docker containers on a single host communicate via a host-internal virtual bridge network, with only the host's own IP exposed externally (port-mapping-based). AWS's `awsvpc` network mode (for ECS) and the **VPC CNI (Container Network Interface) plugin** (for EKS) instead give **every Task/Pod its own first-class ENI and VPC-routable private IP address**, drawn from the subnet's CIDR block exactly as an EC2 instance's primary ENI is (VPC notes §9) — meaning Security Groups (VPC/EC2 notes §7, EC2 notes §8.2) can be attached **per-Task/per-Pod**, not just per-host, and VPC Flow Logs (VPC notes §12) capture Task/Pod-level traffic directly.

### 12.2 ENI density — a direct numerical constraint carried over from EC2

Because each Pod consumes a real ENI (or a secondary IP on a shared ENI, depending on the specific VPC CNI mode), the **maximum number of Pods schedulable per EC2 worker Node is bounded by that instance type's ENI/secondary-IP limit** — the exact same per-instance-type ENI count table referenced in the VPC notes (§9) and EC2 notes (§8.1) directly caps Kubernetes Pod density on EC2-backed EKS Nodes, independent of the Node's CPU/memory capacity. This is a frequently-surprising practical constraint: a Node can have abundant free CPU/memory yet be unable to schedule another Pod because it has exhausted its ENI/IP allocation — a networking-layer bin-packing ceiling layered on top of the CPU/memory bin-packing problem in §8.3.

**Approximate Pods-per-Node formula** (for the ENI-based CNI, `IPv4 prefix delegation` disabled, simplified):

$$\text{Max Pods} = (\text{Number of ENIs} \times (\text{IPs per ENI} - 1)) + 2$$

(the $-1$ per ENI accounts for each ENI's primary IP being reserved for the Node itself, not schedulable to a Pod, and the $+2$ accounts for two Pods — typically the CNI's own components — reserved regardless of ENI count; the exact constant varies slightly by CNI version, but the multiplicative structure — ENI count × IPs-per-ENI — is the durable, exam-relevant relationship).

---

## 13. Container Storage — Direct Extension of the Storage Notes

Containers are, by default, **ephemeral** — a container's writable layer (§3.1) is destroyed when the container stops, exactly as an EC2 Instance Store volume is wiped on termination (EC2 notes §7.1). For workloads needing persistent or shared storage:

| Storage need | ECS/EKS mechanism | Underlying AWS service |
|---|---|---|
| Persistent block storage, single-Task/Pod | EBS volume attached to the Task/Pod | EBS (EC2 notes §7.2) — same volume type tradeoffs apply directly |
| Shared storage across many Tasks/Pods concurrently | EFS-backed volume mount | EFS (Storage notes §3) — the same "concurrent, POSIX, multi-writer" gap EFS fills for EC2 applies identically to containers |
| Fargate-specific persistent storage | EFS only (Fargate Tasks cannot directly attach EBS in the way EC2-launch-type Tasks can, since there is no visible underlying instance to attach a block device to) | EFS — this is a direct architectural consequence of Fargate hiding the host (§9.1): a block device needs a host to attach to, while a network filesystem does not |

---

## 14. Cross-Reference Summary — How This Document Connects to EC2, Hypervisor, VPC, and Storage Notes

| Concept in this document | Connects to |
|---|---|
| Namespaces + cgroups as OS-level isolation | Hypervisor notes §1, §3–4 (hardware-level isolation) — direct contrast: same isolation *goal*, different mechanism and strength |
| Docker image layering (copy-on-write) | EC2 notes §7.3 (EBS snapshot deltas), Storage notes §9.1 — same "store deltas, reconstruct lazily" pattern |
| ECR backed by S3 | Storage notes §2 — literal reuse of the object storage layer |
| Kubernetes reconciliation control loops (ReplicaSet, HPA) | EC2 notes §11 (Auto Scaling target tracking), Storage notes §2.3 (Lifecycle policies) — same declarative control-loop pattern, three layers of the stack |
| Cluster Autoscaler triggering ASG scale-out | EC2 notes §11 — literal reuse; container-level scaling is a control loop layered on top of EC2's own control loop |
| Fargate/Lambda Firecracker microVMs | Hypervisor notes §2.1 (KVM), §3.4 (VT-x/AMD-V), §8.3 (TCB minimization); EC2 notes §2 (Nitro Hypervisor) — same minimal-hypervisor design lineage, explicitly named |
| Lambda concurrency throttling | Storage notes §7.2 (DynamoDB hot-partition throttling) — same "hard per-resource ceiling causes rejection despite spare aggregate capacity" pattern |
| ECS/EKS `awsvpc`/VPC CNI giving Tasks/Pods real ENIs | VPC notes §9 (ENI), EC2 notes §8.1 — literal reuse of the ENI/Security Group model at finer granularity |
| ENI-based Pods-per-Node ceiling | VPC notes §9, EC2 notes §8.1 (per-instance-type ENI limits) — same constraint table, now capping Pod density instead of just multi-homing |
| Container ephemeral storage vs EBS/EFS-backed persistence | EC2 notes §7.1 (Instance Store ephemerality), Storage notes §3 (EFS's concurrent-access role) — same durability/persistence reasoning, one layer up |

---

## 15. Suggested Numerical / Formula-Based Problem Bank

1. **Fargate vs EC2 launch type cost crossover**: Given a Task's vCPU/memory requirements, Fargate's per-vCPU-second and per-GB-second rates, and an EC2 instance's hourly rate with a stated achievable bin-packing density (Tasks per instance), compute the monthly cost under each launch type, and find the bin-packing density at which the EC2 launch type becomes cheaper than Fargate for the same aggregate workload.
2. **Lambda concurrency and throttling**: Given an incoming request rate $\lambda$ (requests/sec) and average duration $d$ (seconds), compute the required concurrency using Little's Law; given a configured reserved concurrency limit below this figure, compute the fraction of requests expected to be throttled, and the minimum concurrency limit needed to avoid throttling entirely.
3. **Lambda memory/cost optimization**: Given a function's duration as a (stated or assumed inversely-related) function of allocated memory, and Lambda's per-GB-second billing rate, compute total cost at two or more candidate memory settings, and identify the memory allocation that minimizes total cost for a fixed number of monthly invocations.
4. **Kubernetes bin-packing / cluster sizing**: Given $n$ Pods each with CPU/memory requests, and a Node type with fixed CPU/memory capacity, compute the theoretical minimum number of Nodes required (an idealized bin-packing lower bound), and compare it against the number of Nodes actually required under a stated greedy-heuristic packing efficiency (e.g., 80% effective utilization).
5. **Pods-per-Node ENI ceiling**: Given an EC2 instance type's maximum ENI count and IPs-per-ENI (from the standard AWS limits table), compute the maximum schedulable Pods per Node using the formula in §12.2, and determine whether a given target Pod density is CPU/memory-bound or ENI-bound for that instance type.

---

## 16. Summary Table — Containers, Orchestration & Serverless Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Isolation mechanism | Namespaces + cgroups vs hypervisor + EPT/NPT | Containers isolate at the OS-kernel layer (visibility + consumption limits); VMs isolate at the hardware layer — weaker but far cheaper/faster isolation |
| Image/registry | Docker layers, ECR | Copy-on-write layered images, same delta-storage pattern as EBS snapshots; ECR is S3 underneath |
| Orchestration | ECS vs Kubernetes/EKS | AWS-proprietary simplicity vs portable, ecosystem-rich complexity; both solve scheduling, self-healing, service discovery |
| Compute options | EC2 launch type vs Fargate vs Lambda | Increasing abstraction: manage the host → manage the container image only → manage only the function code |
| Serverless isolation | Firecracker microVMs | The literal architectural bridge to the Hypervisor notes — a minimal, KVM-based hypervisor giving Fargate/Lambda VM-grade isolation at container-grade speed |
| Autoscaling | HPA (Pod-level) + Cluster Autoscaler (Node-level) | Two-level control loop; HPA reuses the EC2 ASG target-tracking formula verbatim, one layer up |
| Concurrency math | Little's Law ($\text{Concurrency} = \lambda \times d$) | Governs Lambda throttling and general system sizing; same "hard ceiling causes rejection despite spare aggregate capacity" pattern as DynamoDB hot partitions |
| Networking | `awsvpc`/VPC CNI, ENI-per-Pod | Direct reuse of ENI/Security Group model at Task/Pod granularity; introduces an ENI-count-driven Pod density ceiling independent of CPU/memory |
| Storage | Ephemeral container FS vs EBS/EFS-backed volumes | Same ephemeral-vs-persistent reasoning as Instance Store vs EBS; Fargate's hidden host forces EFS (network-attached) over EBS (host-attached) for persistence |
| Economics | Pay-for-instance (EC2) vs pay-for-request (Fargate) vs pay-for-execution (Lambda) | Same recurring "higher per-unit price in exchange for zero operational/capacity-planning burden" tradeoff seen throughout this document series |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 15 reserved as a follow-up tutorial/assignment. Designed to be read as the fifth document in this set — Section 9.2 in particular should be read as the single most direct bridge back to the Hypervisor notes, since Firecracker is a named, concrete descendant of exactly the KVM/minimal-hypervisor lineage discussed there.*
