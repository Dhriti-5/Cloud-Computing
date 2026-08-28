# Cloud Networking — VPC Deep Dive — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Companion document to: "Amazon EC2 — Technical Lecture Notes" and "Hypervisors — Technical Lecture Notes"
### Prerequisite recap assumed: Hypervisor I/O virtualization (§5, Hypervisor notes), Nitro System and ENI/Security Groups (§2, §8, EC2 notes), basic IPv4 addressing and subnetting

---

## 1. Positioning VPC in the AWS Stack

A **Virtual Private Cloud (VPC)** is AWS's **software-defined, logically isolated network** that customers provision inside a Region. If EC2 answers "how do I get a virtual machine," VPC answers "how do those virtual machines — and every other AWS service that needs an IP address — talk to each other, to the internet, and to my on-premises network, while remaining isolated from every other customer's traffic on the same physical infrastructure."

Formally, extending the layering diagram from the EC2 notes:

```
Physical Data Center (AWS-owned hardware, Regions/Availability Zones)
        |
Nitro System (hardware offload + lightweight hypervisor)        <-- hypervisor layer
        |
Nitro Card for VPC (packet encapsulation, mapping service lookups, SG/NACL enforcement)  <-- network data-plane layer
        |
VPC (software-defined network: CIDR block, subnets, route tables, gateways) <-- network control-plane / IaaS layer
        |
ENI (per-instance virtual NIC) --- EC2 instance
```

**Key conceptual point carried over from the Hypervisor notes:** a VPC is not "a physical network you can point a cable at." It is a **control-plane abstraction** realized as an **overlay network** on top of AWS's physical Clos-topology data center fabric. Every packet a VPC "sees" is, at the physical layer, encapsulated, routed by AWS's internal mapping service, and de-encapsulated at the destination Nitro Card — the same encapsulate/de-encapsulate pattern you saw with GENEVE for Gateway Load Balancer (EC2 notes, §12) is the general mechanism, not a special case. This is why two instances in the same VPC but different Availability Zones can communicate at low latency without the customer ever configuring physical routing: the "network" the customer configures (route tables, subnets, CIDR blocks) is a **logical policy specification**, and AWS's mapping service + Nitro Cards are the **enforcement mechanism**, in exactly the same sense that a hypervisor's shadow/nested page tables (Hypervisor notes, §4) are the enforcement mechanism behind a guest's "physical" memory abstraction. Both are cases of a thin, customer-facing abstraction backed by a hardware-accelerated translation layer.

---

## 2. Why VPC Exists — Historical/Architectural Reasoning

### 2.1 EC2-Classic (2006–2013 rollout, retired 2022)

EC2 originally launched with **no VPC concept at all**. Every EC2 instance was placed on a **flat, shared network** spanning a Region:

- All EC2-Classic instances of all customers in a Region drew private IPv4 addresses from a single large, AWS-managed address space.
- There was no customer-defined subnetting, no customer-controlled route table, and no way to establish private, non-internet-routed connectivity to on-premises infrastructure.
- Isolation between customers relied entirely on Security Groups (instance-level, stateful firewall) — there was no network-layer segmentation a customer could design themselves.
- Customers could not choose their own private IP ranges, which made **hybrid cloud** (extending an on-premises network into AWS via VPN) fundamentally difficult: on-premises RFC 1918 ranges could easily collide with the AWS-assigned range, and there was no mechanism to route private traffic between an on-prem network and specific EC2 instances at all.

### 2.2 The shift to VPC (introduced 2009, made default 2013)

VPC was introduced to give customers the same design freedom they had in a traditional on-premises data center — the ability to define their own IP address plan, subnet topology, routing policy, and network-layer access control — **while the underlying physical multi-tenant hardware stayed exactly the same.** This is the same architectural motivation you have seen twice already in this course:

| Layer | Problem | AWS's abstraction-based solution |
|---|---|---|
| Compute (Hypervisor notes) | One physical CPU/host must run many mutually distrusting guests | Hypervisor multiplexes CPU/memory/I/O into isolated VMs |
| Compute economics (EC2 notes) | Physical servers are slow to provision, cannot be resized elastically | EC2 control plane turns physical capacity into an elastic, API-driven resource |
| **Networking (this document)** | **One physical network fabric must carry many mutually distrusting customers' traffic, each wanting their own IP plan** | **VPC overlay network gives each customer a private, customer-defined address space and topology on shared physical wiring** |

By 2013 VPC became the default (and, since 2022, the *only*) provisioning model for new AWS accounts — EC2-Classic's full retirement in August 2022 is the formal end of the flat-network era.

---

## 3. VPC — Formal Definition and CIDR Fundamentals

### 3.1 Definition

A VPC is a **Regional** resource (it does not span Regions) defined primarily by an **IPv4 CIDR block** (and optionally one or more IPv6 CIDR blocks). All subnets, route tables, gateways, and ENIs that belong to the VPC draw their addressing from, and are governed by policy attached to, this VPC.

- **CIDR block size**: between `/16` (65,536 addresses) and `/28` (16 addresses) for IPv4, primary + up to 4 additional secondary CIDR blocks (subject to region-specific limits) can be associated with a single VPC to allow later address-space expansion without recreating the VPC.
- A VPC **cannot span more than one Region**, but it **does span all Availability Zones within that Region** — this is the direct networking counterpart of the EC2 notes' point that an ENI (and hence a subnet) is tied to exactly one AZ, while the VPC as a whole is Regional.

### 3.2 CIDR address-count formula

For an IPv4 CIDR block with prefix length $p$:

$$\text{Total addresses in block} = 2^{(32-p)}$$

**AWS reserves exactly 5 IP addresses in every subnet** (not just the VPC) — this is a critical, frequently-tested detail:

| Reserved address | Position in subnet (example: `10.0.0.0/24`) | Purpose |
|---|---|---|
| Network address | `10.0.0.0` | Identifies the subnet itself (standard IP networking convention) |
| VPC router | `10.0.0.1` | Reserved for the implicit VPC router (the "route table enforcer") |
| DNS | `10.0.0.2` | Reserved for the Amazon-provided DNS resolver (Route 53 Resolver, base of the VPC's DNS + 2) |
| Reserved for future use | `10.0.0.3` | AWS-reserved |
| Broadcast address | `10.0.0.255` (last address) | AWS does not support broadcast within a VPC, but the address is still reserved for consistency with standard IPv4 subnetting |

**Usable host addresses per subnet:**

$$\text{Usable addresses} = 2^{(32-p)} - 5$$

This is the direct networking analogue of the EC2 notes' gp2 IOPS formula ($3\times\text{GiB}$) and the Hypervisor notes' memory overcommit ratio: a clean closed-form relationship AWS imposes on top of a raw resource, ideal for numerical problems (see §16).

### 3.3 Worked example

A `/24` subnet: $2^{(32-24)} - 5 = 256 - 5 = 251$ usable addresses.
A `/28` subnet (smallest allowed): $2^{4} - 5 = 16 - 5 = 11$ usable addresses.

---

## 4. Subnets, Availability Zones, and the Public/Private Distinction

### 4.1 Subnet = CIDR sub-block + AZ binding + route table association

A **subnet** is a subdivision of the VPC's CIDR block, and — unlike the VPC itself — a subnet is bound to exactly **one Availability Zone**. This AZ binding is what makes subnet design the primary lever for the fault-isolation strategies discussed in the EC2 notes' Placement Groups section (§9): a Multi-AZ architecture is, at the networking layer, nothing more than "one subnet per AZ, each with its own route table entries."

**Critical conceptual correction (frequently misunderstood):** "Public" and "private" are **not** properties of a subnet in AWS's data model. A subnet is **public** purely as a *consequence* of its route table containing a route to an **Internet Gateway** (§5) for `0.0.0.0/0`. There is no separate "make this subnet public" flag — publicness is entirely derived from routing policy. This mirrors the EC2 notes' point about Security Groups vs NACLs: **AWS networking constructs are compositional** — a small number of primitives (CIDR blocks, route tables, gateways) combine to produce emergent properties (public/private) rather than the emergent property being a first-class configurable object itself.

### 4.2 Subnet types — comparison table

| Subnet type | Route table entry for `0.0.0.0/0` | Typical resource placement | Outbound internet path |
|---|---|---|---|
| **Public subnet** | → Internet Gateway (IGW) | Load balancers, NAT Gateways, bastion hosts | Direct via IGW |
| **Private subnet (NAT-routed)** | → NAT Gateway (itself in a public subnet) | Application/DB tier instances that need outbound internet (e.g., OS patching) but must not be inbound-reachable from the internet | Indirect, via NAT Gateway |
| **Private subnet (isolated)** | No `0.0.0.0/0` route at all | DB tier with no internet dependency at all, highest-sensitivity workloads | None |

---

## 5. Gateways — Internet Gateway, NAT Gateway, NAT Instance, Egress-Only IGW

### 5.1 Internet Gateway (IGW)

A horizontally scaled, redundant, highly available VPC component (one per VPC, no bandwidth constraint the customer needs to size — unlike a NAT Gateway) that performs **1:1 NAT** between an instance's private IP and its associated public IPv4 address/Elastic IP for internet-bound traffic, and serves as the target for the default route in any public subnet. The IGW has **no availability or throughput dimension for the customer to provision** — it is not billed per-hour and requires no capacity planning, which is the primary practical distinction from a NAT Gateway below.

### 5.2 NAT Gateway vs NAT Instance — comparison table

Private subnet instances that need **outbound-only** internet access (e.g., downloading OS patches) without being directly internet-reachable require **Network Address Translation (NAT)**, provided by either a managed NAT Gateway or a self-managed NAT Instance:

| Property | NAT Gateway (managed) | NAT Instance (self-managed EC2) |
|---|---|---|
| Underlying resource | AWS-managed service, placed in a public subnet | An ordinary EC2 instance running NAT software, `Source/Dest Check` disabled |
| Availability | Highly available **within its AZ only** — must deploy one NAT Gateway per AZ for full Multi-AZ resilience (a direct parallel to the EC2 notes' Multi-AZ Auto Scaling discussion) | Single point of failure unless customer builds their own failover (e.g., via a health-checked script + route table update) |
| Bandwidth | Scales automatically, burst up to tens of Gbps (baseline ~5 Gbps, bursts substantially higher depending on the current generation) | Capped by the underlying EC2 instance type's network performance (EC2 notes §5.1 instance family table) |
| Cost model | Hourly charge **+ per-GB data processing charge** | Only the underlying instance's compute cost (no separate per-GB charge) |
| Security Group support | **No** — cannot attach a Security Group directly to a NAT Gateway | **Yes** — an ordinary EC2 instance, so full SG/NACL control applies |
| Maintenance | Fully managed, no patching | Customer-managed OS, needs patching like any EC2 instance |
| Port forwarding / bastion use | Not supported | Supported (can double as a bastion) |

**Practical/exam-relevant framing:** NAT Gateway vs NAT Instance is structurally the same "managed convenience vs customer-controlled flexibility" tradeoff you saw between **gp3/EBS vs Instance Store** in the EC2 notes (§7.1) — a fully-managed, elastic, metered service versus a customer-operated resource with more control but more operational burden and a hard capacity ceiling.

### 5.3 Egress-Only Internet Gateway (IPv6-specific)

IPv6 addresses in a VPC are, by AWS design, **globally routable and not subject to NAT** (IPv6 has no concept of private RFC-1918-style translation the way IPv4 does). To give an IPv6 subnet the same "outbound yes, inbound no" property that a NAT Gateway provides for IPv4, AWS provides a distinct construct — the **Egress-Only Internet Gateway** — which is stateful and permits only connections *initiated* from inside the VPC, structurally analogous to how a Security Group is stateful with respect to return traffic (EC2 notes §8.2).

---

## 6. Route Tables — The VPC's Control Plane Object

### 6.1 Formal model

A route table is an **ordered set of (destination CIDR, target) rules**. Every subnet is associated with exactly one route table (the **main route table** by default, or a **custom route table** if explicitly associated). Route selection follows the **longest-prefix-match** rule — the same principle used in general IP routing (and directly analogous to the Hypervisor notes' point that NACL rules are evaluated in explicit priority order, §8.2 EC2 notes, rather than SG's "any match wins" model): the most specific (longest prefix) matching route wins, regardless of the order routes were added.

### 6.2 Local route (implicit, cannot be deleted)

Every route table contains an immutable **local route** for the VPC's own CIDR block (and any secondary CIDR blocks), targeting `local` — this is what allows all subnets within the VPC to reach each other by default, and it always has implicit highest priority as the most specific applicable route for intra-VPC traffic.

### 6.3 Route table targets — reference table

| Target type | Used for |
|---|---|
| `local` | Intra-VPC traffic (implicit, all subnets) |
| Internet Gateway (`igw-...`) | Public subnet default route |
| NAT Gateway (`nat-...`) | Private subnet default route (outbound only) |
| Virtual Private Gateway (`vgw-...`) | Traffic destined for an on-premises network over Site-to-Site VPN |
| Transit Gateway (`tgw-...`) | Traffic destined for other VPCs/on-prem networks attached to a Transit Gateway (§9) |
| VPC Peering Connection (`pcx-...`) | Traffic destined for a peered VPC's CIDR |
| Gateway VPC Endpoint | Traffic destined for S3/DynamoDB via a prefix-list route, without leaving the AWS network |
| Elastic Network Interface (`eni-...`) | Traffic routed through a specific appliance instance (e.g., a self-managed firewall/NAT instance) |

---

## 7. Security at the Network Layer — Extending the EC2 Notes

The Security Group vs NACL comparison table is already covered in full in the EC2 notes (§8.2) — Security Groups are stateful, instance/ENI-level, allow-only, enforced in Nitro Card hardware; NACLs are stateless, subnet-level, allow+deny, evaluated in numeric rule order. This section extends that discussion to VPC-wide design implications.

### 7.1 Defense-in-depth: why both layers exist simultaneously

A packet entering a subnet is evaluated by the **NACL first** (subnet boundary), then by the **Security Group** (instance boundary) if it is inbound, or SG-then-NACL if outbound. This two-layer model is the network-layer equivalent of the **TCB-minimization principle** from the Hypervisor notes (§8.3): rather than relying on a single enforcement point, AWS composes a coarse-grained, explicit-deny, stateless perimeter control (NACL) with a fine-grained, allow-only, stateful, hardware-enforced control (Security Group) — a failure or misconfiguration in one layer does not by itself eliminate isolation, the same layered-isolation reasoning that motivates Security Group enforcement happening in Nitro Card hardware in the first place (EC2 notes §8.2, §2.3).

### 7.2 Stateless NACL rule evaluation — worked example

Because NACLs are stateless, an engineer must explicitly permit **both directions** of any connection. For a web server subnet that must accept inbound HTTPS (443) and allow the corresponding ephemeral-port return traffic:

| Rule # | Type | Protocol | Port range | Source/Dest | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | 443 | `0.0.0.0/0` | ALLOW |
| 200 | Inbound | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW (ephemeral return ports for client-initiated connections the server itself makes, and for the client's ephemeral source port on the inbound leg) |
| * | Inbound | ALL | ALL | `0.0.0.0/0` | DENY (implicit final rule, cannot be deleted) |
| 100 | Outbound | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW (server's response back to client's ephemeral port) |
| * | Outbound | ALL | ALL | `0.0.0.0/0` | DENY (implicit final rule) |

Forgetting the ephemeral-port outbound rule is the single most common NACL misconfiguration in practice — the connection's SYN reaches the server, but the SYN-ACK is silently dropped, producing symptoms that look like a Security Group problem but are in fact a NACL statelessness problem.

---

## 8. Inter-VPC and Hybrid Connectivity

### 8.1 VPC Peering

A **1:1, non-transitive** private connection between two VPCs (same or different accounts, same or different Regions). Non-transitivity means if VPC A is peered with VPC B, and VPC B is peered with VPC C, **A cannot reach C** through B — each pair requires its own explicit peering connection and its own route table entries in both VPCs.

**Full-mesh connection-count formula:** for $n$ VPCs requiring full any-to-any connectivity via peering alone:

$$\text{Peering connections required} = \binom{n}{2} = \frac{n(n-1)}{2}$$

This quadratic growth is precisely the architectural motivation for Transit Gateway below — the same "point-to-point doesn't scale, introduce a hub" reasoning that motivates hub-and-spoke topologies in general distributed-systems design.

### 8.2 Transit Gateway (TGW)

A **Regional (with inter-Region peering support), transitive routing hub**. Each attached VPC (or VPN, or Direct Connect connection) requires only **one** attachment to the TGW, and the TGW's own route table(s) determine which attachments can reach which other attachments.

**Connection-count formula with a hub:**

$$\text{Attachments required} = n$$

instead of $\frac{n(n-1)}{2}$ — a direct instance of the classic hub-and-spoke vs full-mesh tradeoff, with the added benefit that TGW route tables allow **selective transitivity** (e.g., a shared-services VPC reachable by all spokes, while spoke VPCs cannot reach each other) — something pure VPC Peering's non-transitivity makes structurally impossible without a hub.

### 8.3 VPC Peering vs Transit Gateway — comparison table

| Property | VPC Peering | Transit Gateway |
|---|---|---|
| Transitivity | Non-transitive (strict 1:1) | Transitive (hub-and-spoke, policy-controlled via TGW route tables) |
| Scaling for $n$ VPCs (full mesh) | $O(n^2)$ connections | $O(n)$ attachments |
| Cost model | No hourly charge; billed only for inter-AZ/inter-Region data transfer | Hourly charge per attachment + per-GB data processing charge |
| Bandwidth | No AWS-imposed aggregate throughput ceiling (limited by instance/ENI bandwidth) | Up to 50 Gbps per VPC attachment (burstable, subject to per-flow limits) |
| Overlapping CIDRs | Not supported between peered VPCs | Not supported between VPCs attached to the same TGW route table |
| On-premises connectivity | Not directly (peering is VPC-to-VPC only) | Yes — VPN and Direct Connect can attach directly to a TGW, unifying cloud and on-prem routing |

### 8.4 PrivateLink and VPC Endpoints

Where peering/TGW connect **networks**, **VPC Endpoints** connect a VPC to a **specific AWS service** without traversing the public internet and, critically, without requiring the consuming VPC and the service's VPC to have non-overlapping CIDRs or any routing relationship at all.

| Endpoint type | Mechanism | Services | Route table impact |
|---|---|---|---|
| **Gateway Endpoint** | Adds a route (via a managed prefix list) to the route table, targeting the AWS service directly | S3, DynamoDB only | Requires an explicit route table entry |
| **Interface Endpoint (PrivateLink)** | Provisions an ENI with a private IP **inside your subnet**, backed by AWS PrivateLink; DNS is transparently redirected to this private IP | Most AWS services (EC2 API, Systems Manager, most others) and customer/third-party services published via PrivateLink | No route table change needed — resolved via DNS + ENI, billed hourly + per-GB |

**Architectural significance:** PrivateLink is the mechanism that lets AWS itself, and third-party SaaS vendors, expose a service to thousands of customer VPCs **without peering with any of them** — the service provider's VPC and the consumer's VPC never need route-level awareness of each other, only the interface endpoint's ENI does. This is the standard modern alternative to peering purely for "I need to reach one service," reserving peering/TGW for "I need general network reachability."

---

## 9. Elastic Network Interfaces, Private IPs, and Elastic IPs in VPC Context

This section extends EC2 notes §8.1 (ENI) from the compute perspective to the VPC design perspective.

- An ENI's IP address is drawn from its subnet's CIDR block, which is itself drawn from the VPC's CIDR block — the full addressing hierarchy is **VPC CIDR → subnet CIDR → ENI private IP**, directly mirroring the **VPC → subnet → instance** placement hierarchy.
- **Elastic IP (EIP)**: a static, VPC-scoped public IPv4 address that a customer can allocate and associate/disassociate with any ENI, decoupling the public identity of a service from the lifecycle of any particular instance — e.g., failing over a public-facing service to a replacement instance without any DNS propagation delay. AWS separately charges for EIPs that are allocated but **not** associated with a running instance, specifically to discourage hoarding a scarce public IPv4 resource.
- **Secondary private IPs** on a single ENI enable **multiple TLS certificates/hostnames bound to distinct IPs on one instance** — a legacy pattern from before SNI (Server Name Indication) was universally supported, still occasionally relevant for appliance-style workloads.

---

## 10. DNS Inside a VPC

### 10.1 Route 53 Resolver (the `.2` address from §3.2)

Every VPC has an implicit DNS resolver at the base of its CIDR block **+2** (e.g., `10.0.0.2` in a `10.0.0.0/24` subnet), reachable by all resources in the VPC by default (controlled by the `enableDnsSupport` VPC attribute). This resolver handles:
- Public DNS resolution (recursively, out to the internet)
- Private DNS resolution for VPC-internal names (e.g., instance private DNS hostnames, and private hosted zones in Route 53)
- Automatic resolution of PrivateLink Interface Endpoint private DNS names (§8.4), which is what allows application code to use a service's standard public hostname (e.g., an S3 or third-party SaaS hostname) unmodified while transparently resolving to the private ENI IP inside the VPC — no application-level reconfiguration needed to "go private."

### 10.2 Two independently controllable attributes

| VPC DNS attribute | Controls |
|---|---|
| `enableDnsSupport` | Whether the VPC's Route 53 Resolver answers DNS queries at all |
| `enableDnsHostnames` | Whether instances launched in the VPC are assigned public/private DNS hostnames |

Both must be enabled for private DNS resolution of PrivateLink interface endpoints and for instances to receive usable DNS hostnames — a common source of "PrivateLink endpoint created but still resolving to the public IP" misconfiguration when `enableDnsHostnames` is left off.

---

## 11. Hybrid Connectivity — VPN vs Direct Connect

| Property | Site-to-Site VPN | Direct Connect (DX) |
|---|---|---|
| Underlying medium | Encrypted IPsec tunnel over the **public internet** | Dedicated **private physical fiber** circuit from a customer/partner location to an AWS Direct Connect location |
| Bandwidth | Typically capped per tunnel (traffic exceeding a few hundred Mbps to low single-digit Gbps sustained, though multiple tunnels can be aggregated) | 50 Mbps to 100 Gbps dedicated, or hosted connections from partners at finer granularity |
| Latency/jitter | Variable — subject to public internet congestion | Consistent, predictable — dedicated circuit, no public internet contention |
| Setup time | Minutes to hours (fully API/console-driven) | Days to weeks (physical cross-connect provisioning required) |
| Encryption | Native (IPsec) | Not natively encrypted (a private circuit, not a VPN) — MACsec or an IPsec-over-DX overlay is layered on top if encryption in transit is required |
| Redundant/HA target | Two tunnels per VPN connection by default (different AWS endpoints) | Requires provisioning a second physical connection, ideally at a different DX location, for full redundancy |
| Cost model | Hourly charge + data transfer out | Port-hour charge + data transfer out (data transfer out rate is substantially lower than internet/VPN egress pricing) |
| Typical use case | Quick setup, lower/variable bandwidth needs, DR/backup path | Sustained high-bandwidth, latency-sensitive hybrid workloads (large data migrations, consistent low-latency hybrid applications) |

**VPN + Direct Connect together (common HA pattern):** DX as the primary path, with a Site-to-Site VPN as an automatic failover path if the physical DX circuit fails — combining DX's performance with VPN's rapid, internet-independent-of-physical-infrastructure failover.

---

## 12. VPC Flow Logs

**VPC Flow Logs** capture metadata (not packet payload) about IP traffic to/from ENIs at the VPC, subnet, or individual-ENI level, publishable to CloudWatch Logs, S3, or Kinesis Data Firehose. This is the network-layer counterpart of the EC2 notes' CloudWatch discussion (§14) — just as the hypervisor/Nitro layer cannot see inside guest OS memory without an agent (EC2 notes §14 note), Flow Logs cannot see inside encrypted payloads or application-layer content; they record only 5-tuple-style connection metadata (source/dest IP, source/dest port, protocol, packets, bytes, `ACCEPT`/`REJECT` action, and which layer — SG or NACL — made the accept/reject decision).

**Default flow log capture window: every packet is aggregated into records covering an approximately 60-second (up to 10-minute maximum) aggregation interval** rather than being emitted per-packet — a deliberate throughput/cost tradeoff, structurally the same "sampling interval vs granularity vs cost" tradeoff as EC2's Basic (5-minute) vs Detailed (1-minute) CloudWatch monitoring tiers (EC2 notes §14).

---

## 13. Multi-AZ and Multi-Region VPC Design Patterns

### 13.1 Why subnet-per-AZ is the baseline pattern

Since a subnet is bound to one AZ (§4.1), and an AZ is AWS's fault-isolation boundary (independent power, cooling, and network — directly analogous to the "Spread" Placement Group's fault-isolation goal in the EC2 notes §9), the standard resilient VPC design provisions **at least one public and one private subnet per AZ used**, each pair with independent route tables where policy differs (e.g., each private subnet routing to the NAT Gateway in its *own* AZ's public subnet, avoiding a cross-AZ data transfer charge and avoiding a single NAT Gateway becoming a cross-AZ single point of failure).

### 13.2 Reference 3-tier, 2-AZ VPC layout

| Tier | AZ-a subnet | AZ-b subnet | Route table target for `0.0.0.0/0` |
|---|---|---|---|
| Public (ALB, NAT GW) | `10.0.0.0/24` | `10.0.1.0/24` | IGW |
| Private/App | `10.0.10.0/24` | `10.0.11.0/24` | NAT GW in same-AZ public subnet |
| Private/DB (isolated) | `10.0.20.0/24` | `10.0.21.0/24` | No default route |

### 13.3 Cross-Region connectivity

A VPC cannot span Regions, so multi-Region architectures require either **VPC Peering across Regions** (non-transitive, as in §8.1, plus inter-Region data transfer charges) or **Transit Gateway inter-Region peering** (TGW-to-TGW, preserving the hub's transitive routing policy across Regions) — the same $O(n^2)$ vs $O(n)$ reasoning from §8.1–8.2 applies at Region scope as well.

---

## 14. VPC Sizing and Capacity Planning — Quantitative Core

This is the most quantitatively rich section of this document, structurally parallel to the EC2 notes' T-family CPU Credit model (§6) and the Hypervisor notes' overcommit ratio formulas (§4.3, §6) — a clean formula governing a finite, plannable resource.

### 14.1 Address exhaustion planning

Given a VPC CIDR of prefix $p_{vpc}$ subdivided into $k$ equally-sized subnets of prefix $p_{sub}$:

$$k_{max} = 2^{(p_{sub} - p_{vpc})}$$

$$\text{Total usable addresses across all subnets} = k \times \left(2^{(32 - p_{sub})} - 5\right)$$

Note the **5-address-per-subnet tax compounds with subnet count**: a `/16` VPC ($65{,}536$ raw addresses) split into sixteen `/20` subnets loses $16 \times 5 = 80$ addresses to reservation overhead, whereas the same `/16` split into 256 `/24` subnets loses $256 \times 5 = 1{,}280$ addresses — finer-grained subnetting directly trades away usable address space for isolation/blast-radius granularity, a direct numerical instance of the same "more isolation boundaries cost more overhead" theme as Spread/Partition Placement Groups (EC2 notes §9).

### 14.2 NAT Gateway cost model

$$\text{NAT Gateway monthly cost} = (\text{Hourly rate} \times 730) + (\text{Data processed in GB} \times \text{Per-GB rate})$$

(730 ≈ average hours per month, the same constant used implicitly in Reserved Instance breakeven calculations in the EC2 notes §10.)

### 14.3 Transit Gateway vs full-mesh peering — cost/complexity crossover

Combining §8.1 and §8.2's connection-count formulas with per-connection cost, a natural numerical question (see §16) is: at what number of VPCs $n$ does Transit Gateway's flat per-attachment cost become cheaper/simpler than full-mesh Peering's quadratically growing connection (and route-table-entry) management overhead — even before accounting for the fact that Peering itself has no attachment-hour charge, making the crossover purely an **operational-complexity** argument (route table entries: $n(n-1)$ total directional entries for full mesh vs $n$ for a hub) rather than a pure-cost one in most real deployments.

---

## 15. Cross-Reference Summary — How This Document Connects to EC2 and Hypervisor Notes

| Concept in this document | Connects to |
|---|---|
| Nitro Card for VPC as the network data-plane enforcer | Nitro System, EC2 notes §2 — same hardware family that also handles EBS I/O |
| VPC overlay network / mapping service | Hypervisor notes §4 (GVA→GPA→HPA two-level translation) — both are thin logical abstractions backed by hardware-accelerated translation |
| GENEVE encapsulation (Gateway Load Balancer) | EC2 notes §12 — same encapsulation protocol family used for the general VPC overlay data path |
| Security Groups (stateful, hardware-enforced) | EC2 notes §8.2 — fully defined there; this document only extends to NACL interaction |
| ENI, Elastic IP | EC2 notes §8.1 — this document adds the VPC-level addressing hierarchy context |
| Subnet-per-AZ resilience pattern | EC2 notes §9 (Placement Groups: Spread strategy) — same fault-isolation-vs-overhead tradeoff, applied to networking instead of compute placement |
| NAT Gateway vs NAT Instance (managed vs self-operated) | EC2 notes §7.1 (EBS vs Instance Store) — same "managed/elastic/metered vs customer-controlled/capped" tradeoff shape |
| Full-mesh vs hub-and-spoke ($O(n^2)$ vs $O(n)$) | General distributed-systems reasoning; also echoes the EC2 notes' Auto Scaling discussion of moving from manual (per-instance) to policy-driven (control-loop) management at scale |
| VPC Flow Logs aggregation interval | EC2 notes §14 (Basic vs Detailed CloudWatch monitoring) — same sampling-granularity-vs-cost tradeoff |

---

## 16. Suggested Numerical / Formula-Based Problem Bank

1. **Subnet address planning**: Given a VPC CIDR block and a required number of subnets each needing at least $N$ usable host addresses, compute the minimum subnet prefix length required (accounting for the 5 reserved addresses per subnet), and determine the maximum number of such subnets the VPC CIDR can support without overlap.
2. **Peering vs Transit Gateway crossover**: Given $n$ VPCs requiring full any-to-any connectivity, a per-peering-connection route-table management overhead in entries, and a Transit Gateway's flat per-attachment hourly + per-GB cost, compute the connection/entry count under each topology for a given $n$, and find the smallest $n$ at which Transit Gateway's attachment count is less than half of the full-mesh peering connection count.
3. **NAT Gateway cost comparison**: Given expected monthly outbound data volume (GB) for a private subnet, NAT Gateway hourly + per-GB rates, and the compute cost of an EC2 instance type large enough to self-host a NAT Instance at the same throughput, compute the monthly cost of each option and identify the data-volume breakeven point.
4. **Address exhaustion under VPC secondary CIDR growth**: Given an initial `/20` VPC fully subnetted into equally sized `/24` subnets across 2 AZs, compute current usable address utilization; given a required 40% headroom for future growth, compute whether a secondary CIDR block is needed and the minimum secondary CIDR prefix required.
5. **Multi-AZ NAT Gateway data transfer**: Given instance count per AZ, average outbound data volume per instance per day, and a scenario where all AZs share a single NAT Gateway in AZ-a (misconfigured) versus one NAT Gateway per AZ (correct pattern), compute the additional inter-AZ data transfer cost incurred purely by the misconfiguration, using standard inter-AZ data transfer pricing.

---

## 17. Summary Table — VPC Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Foundational | EC2-Classic → VPC | Flat shared network → customer-defined logical overlay network; same physical hardware, richer software-defined abstraction |
| Addressing | CIDR block, 5 reserved IPs/subnet | Usable addresses $= 2^{(32-p)} - 5$; finer subnetting trades usable address space for isolation granularity |
| Topology | Subnet = CIDR + single AZ + route table | Public/private is a derived property of routing (IGW route present or not), not a first-class flag |
| Routing | Route table, longest-prefix-match | Local route always implicit and highest-priority for intra-VPC traffic |
| Egress | IGW vs NAT Gateway vs NAT Instance vs Egress-Only IGW | Managed/elastic/metered vs self-operated/capped — same tradeoff shape as EBS vs Instance Store |
| Security | Security Group (stateful, instance) + NACL (stateless, subnet) | Defense-in-depth composition of two primitives, same TCB-minimization philosophy as Nitro/hypervisor security |
| Inter-VPC | Peering ($O(n^2)$, non-transitive) vs Transit Gateway ($O(n)$, transitive hub) | Classic full-mesh-vs-hub scaling argument, plus TGW's policy-controlled selective transitivity |
| Service access | Gateway Endpoint (S3/DynamoDB) vs Interface Endpoint/PrivateLink | Route-table-based vs ENI+DNS-based; PrivateLink avoids any route-level coupling between provider and consumer VPCs |
| DNS | Route 53 Resolver at CIDR+2 | `enableDnsSupport` + `enableDnsHostnames` both required for private endpoint resolution to work correctly |
| Hybrid | Site-to-Site VPN vs Direct Connect | Fast-to-provision encrypted internet path vs dedicated, predictable, higher-bandwidth private circuit |
| Observability | VPC Flow Logs | Metadata-only, aggregated-interval capture — same sampling/cost tradeoff as CloudWatch Basic vs Detailed monitoring |
| Resilience | Subnet-per-AZ, NAT Gateway-per-AZ | Fault isolation at the networking layer, directly mirroring Spread Placement Groups at the compute layer |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 16 reserved as a follow-up tutorial/assignment. Designed to be read as the third document in this set — Sections 1, 7, 8, 9, and 12 map directly onto the Nitro System, Security Group, and CloudWatch discussions in the EC2 notes, and Section 1 maps onto the overlay/translation-layer reasoning in the Hypervisor notes' memory virtualization section (§4).*
