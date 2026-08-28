# Hypervisors — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Companion document to: "Amazon EC2 — Technical Lecture Notes"

---

## 1. Definition and Position in the Stack

A **hypervisor** (also called a Virtual Machine Monitor, VMM) is a software/firmware layer that creates and manages **virtual machines (VMs)** by abstracting and multiplexing physical hardware resources — CPU, memory, storage, and network — among multiple isolated guest operating systems running concurrently on the same physical host.

Formally, a hypervisor must satisfy the three properties defined by **Popek and Goldberg (1974)** for a piece of software to qualify as a Virtual Machine Monitor:

1. **Equivalence (Fidelity)**: A program running under the VMM should behave identically to how it would behave running directly on the physical hardware (barring timing differences).
2. **Resource control (Safety)**: The VMM must be in complete control of the virtualized resources — a guest must never be able to directly access hardware in a way that bypasses the VMM.
3. **Efficiency (Performance)**: A statistically dominant fraction of guest instructions must execute directly on the physical CPU without VMM intervention (i.e., virtualization overhead must be low; the VMM should not be a full instruction-by-instruction interpreter/emulator for typical instructions).

This is the formal definition your hypervisor unit should already trace back to — everything below is a discussion of *how* real hypervisors achieve these three properties.

---

## 2. Type-1 vs Type-2 Hypervisors

| Property | Type-1 (Bare-Metal / Native) | Type-2 (Hosted) |
|---|---|---|
| Runs on | Directly on physical hardware | On top of a conventional host OS |
| Examples | Xen, VMware ESXi, Microsoft Hyper-V, KVM (architecturally hybrid — see §2.1) | VMware Workstation/Fusion, Oracle VirtualBox, Parallels Desktop |
| Privilege level | Runs at the highest privilege level (Ring 0 / VMX root mode), directly manages hardware | Runs as a process/kernel module under a general-purpose OS, which itself manages hardware |
| Performance | Higher — no intermediary OS scheduling/resource contention | Lower — subject to host OS scheduling, an additional software layer between guest and hardware |
| Primary use case | Data centers, cloud providers (AWS, Azure, GCP), enterprise server virtualization | Desktop virtualization, developer sandboxes, running a second OS on a personal machine |
| I/O path | Direct device driver ownership (or delegated to a privileged guest, e.g., Xen's Dom0) | Routed through the host OS's existing driver stack |

### 2.1 KVM — an architecturally hybrid case (important nuance)

KVM (Kernel-based Virtual Machine) is frequently mis-classified. Technically:

- KVM is a **Linux kernel module** that turns the Linux kernel itself into a Type-1 hypervisor by exposing `/dev/kvm` and using CPU hardware virtualization extensions directly.
- Because the "host OS" (Linux) and the "hypervisor" are the same running kernel, KVM is best described as Type-1 **integrated into a general-purpose OS kernel**, rather than a Type-2 hypervisor bolted on top of one.
- QEMU is used alongside KVM to provide device emulation (BIOS, virtual disk controllers, virtual NICs) for the guest, while KVM itself handles CPU/memory virtualization via hardware extensions.

This KVM+QEMU pairing is the direct ancestor of the **Nitro Hypervisor** discussed in the EC2 document — AWS took KVM and stripped away nearly all of QEMU's device-emulation responsibility by moving it into the Nitro Cards (hardware), leaving KVM to do essentially only CPU/memory virtualization.

---

## 3. CPU Virtualization

### 3.1 The privilege-ring problem

x86 CPUs define four protection rings (0–3), where Ring 0 has full hardware privilege (kernel mode) and Ring 3 is the least privileged (user mode). A conventional OS kernel expects to run in Ring 0. This creates the **fundamental problem of x86 virtualization**: if multiple guest OS kernels all expect Ring 0, but only one entity can actually hold Ring 0 on the real hardware, something has to mediate.

Historically there were three solutions, and this progression is the central narrative of CPU virtualization:

### 3.2 Full Virtualization via Binary Translation

- The hypervisor runs at Ring 0. The guest kernel is demoted to run at a lower privilege ring (e.g., Ring 1) — a technique called **ring deprivileging**.
- **Problem**: some x86 instructions (17 identified "critical instructions" in the original x86 ISA, e.g., `SGDT`, `SIDT`, `POPF` under certain conditions) behave differently depending on privilege level but do **not** trap when executed at a non-Ring-0 level — they silently fail to raise the exception the VMM needs in order to intervene. This violates the Popek-Goldberg requirement that all "sensitive" instructions must be "privileged" (trap when misused). Native x86 (pre-2005) fails this requirement, meaning naive trap-and-emulate is **not sufficient** on x86.
- **Solution (VMware's original approach)**: **Binary Translation (BT)**. The hypervisor scans guest kernel code blocks at runtime and rewrites (translates) any sensitive/non-trapping instruction into a safe equivalent sequence that does trap into the VMM, while leaving ordinary user-level instructions to run natively at full speed. Translated code is cached (translation cache) to amortize the translation cost across repeated execution.
- **Advantage**: Works on **unmodified guest OSes** (no guest awareness of virtualization required).
- **Disadvantage**: Translation overhead, code cache management overhead, added complexity.

### 3.3 Paravirtualization

- The guest OS **kernel source code is modified** to replace sensitive/privileged instructions with explicit **hypercalls** — direct, cooperative calls into the hypervisor (analogous to how a user process makes a syscall into the kernel).
- Pioneered by **Xen** (Xen PV mode). The guest is aware it is virtualized and cooperates actively.
- **Advantage**: Eliminates the need for binary translation or trap-and-emulate entirely for those instructions — lower overhead, since the guest directly asks the hypervisor to do the privileged operation rather than the hypervisor having to detect and intercept it.
- **Disadvantage**: Requires a modified guest kernel — cannot run unmodified proprietary OSes (this is precisely why early Windows guests on Xen required a different mode, and why paravirtualization has been largely superseded now that hardware-assisted virtualization exists).

### 3.4 Hardware-Assisted (Hardware Virtual Machine, HVM) Virtualization

- Introduced by Intel as **VT-x** (Intel Virtualization Technology) and AMD as **AMD-V**, starting ~2005–2006.
- Adds a **new CPU execution mode** distinct from the ring model: **VMX root mode** (hypervisor) and **VMX non-root mode** (guest). The guest kernel can now genuinely run at what it believes is Ring 0, inside VMX non-root mode, and the CPU hardware itself automatically traps all sensitive instructions via **VM Exit** transitions back to root mode — no software ring-deprivileging or binary translation needed.
- Key hardware primitives:
  - **VMCS (Virtual Machine Control Structure)** on Intel / **VMCB (Virtual Machine Control Block)** on AMD — a hardware-defined data structure per vCPU that stores guest CPU state, exit reasons, and controls which events cause a VM Exit.
  - **VM Entry / VM Exit**: hardware-managed transitions between root mode (hypervisor) and non-root mode (guest), replacing software trap-and-emulate.
- This is what makes **unmodified guest OSes run efficiently without binary translation** — the modern default. Both KVM and Xen's HVM mode, VMware ESXi, and Hyper-V all rely on VT-x/AMD-V today; the Nitro Hypervisor is fundamentally a VT-x/AMD-V consumer.

### 3.5 Comparative summary

| Technique | Guest modification required? | Mechanism | Overhead source | Still used today? |
|---|---|---|---|---|
| Full virtualization (Binary Translation) | No | Runtime rewriting of sensitive instructions | Translation + code cache management | Largely deprecated in favor of HVM |
| Paravirtualization | Yes (modified kernel) | Explicit hypercalls | Hypercall dispatch (lower than BT, no translation) | Mostly deprecated for CPU virtualization; paravirtualized *drivers* (see §5) still common |
| Hardware-assisted (HVM) | No | Hardware VM Exit on sensitive instructions | VM Exit/Entry transition latency | **Dominant technique today** (VT-x/AMD-V universal baseline) |

---

## 4. Memory Virtualization

Memory virtualization must solve an additional indirection problem beyond CPU virtualization: a guest OS believes it manages **physical** memory directly, but it is actually managing **guest-physical** memory that the hypervisor must further map onto **host-physical** (real) memory.

This creates a **two-level address translation problem**:

$$\text{Guest Virtual Address (GVA)} \xrightarrow{\text{guest page tables}} \text{Guest Physical Address (GPA)} \xrightarrow{\text{hypervisor-managed mapping}} \text{Host Physical Address (HPA)}$$

### 4.1 Shadow Page Tables (software approach, pre-hardware-assist)

- The hypervisor maintains a **shadow page table** per guest process — a hypervisor-owned page table that directly maps GVA → HPA, bypassing the intermediate GPA step at translation time.
- The guest's own page tables (GVA→GPA) are kept read-only/trapped so the hypervisor can intercept every guest page table modification, propagate the change into the shadow table, and keep the shadow synchronized.
- **Overhead**: every guest page table write, and every TLB miss requiring a page-table walk, potentially causes a VM Exit to keep the shadow table consistent — expensive, especially for workloads with heavy page-table churn (process creation, `fork()`, memory-mapped I/O-heavy workloads).

### 4.2 Hardware-Assisted Memory Virtualization (Nested/Extended Paging)

- Intel **EPT (Extended Page Tables)** and AMD **NPT/RVI (Nested Page Tables / Rapid Virtualization Indexing)**.
- The CPU's Memory Management Unit (MMU) is extended to walk **two** page tables in hardware on every memory access: the guest's own GVA→GPA table (unmodified, guest-managed, no hypervisor trapping needed) and a second, hypervisor-managed GPA→HPA table.
- The CPU hardware performs the full **two-dimensional page walk** natively — no VM Exit required for ordinary guest page table updates, since the hypervisor no longer needs to intercept and shadow every guest page table write.
- **Overhead source shifts** from "VM Exit per page table update" (software shadowing) to "TLB miss cost," since a two-dimensional page walk on a TLB miss is more expensive than a single-dimensional walk (up to a multiplicative increase in the number of memory accesses needed to resolve a miss: a 4-level guest table combined with a 4-level hypervisor table can require up to 24 memory accesses in the worst case for a single TLB miss, versus 4 for native). This is mitigated in practice by large TLBs, **huge pages** (2 MB/1 GB pages drastically reduce the frequency of TLB misses), and dedicated EPT/NPT caching structures on modern CPUs.

### 4.3 Memory Overcommitment Techniques

Hypervisors frequently allow the **sum of guest-configured memory** to exceed physical host RAM, relying on the statistical likelihood that not all guests use their full allocation simultaneously. Key techniques:

| Technique | Mechanism |
|---|---|
| **Ballooning** | A "balloon driver" installed inside the guest is instructed by the hypervisor to allocate ("inflate") memory inside the guest OS, forcing the guest's own memory manager to reclaim/swap out its least-needed pages. The hypervisor then reclaims the physical pages backing the balloon for use elsewhere. This works *with* the guest OS's own memory management policy rather than second-guessing it. |
| **Transparent Page Sharing (TPS) / Kernel Same-page Merging (KSM)** | The hypervisor scans physical pages across all guests, hashes their content, and merges identical pages (common when many guests run the same OS/binaries) into a single copy-on-write physical page. |
| **Hypervisor swapping** | As a last resort, the hypervisor swaps a guest's physical pages out to disk directly, without guest cooperation — highest performance penalty since the hypervisor has no visibility into which guest pages are "hot" vs "cold" (unlike ballooning, which lets the guest itself choose). |

**Overcommitment ratio formula:**
$$\text{Memory Overcommit Ratio} = \frac{\sum \text{Configured Guest Memory}}{\text{Physical Host RAM}}$$

A ratio > 1 indicates overcommitment; sustained high utilization across all guests simultaneously at ratio > 1 risks ballooning/swapping-induced performance degradation.

---

## 5. I/O Virtualization

I/O (storage, network) virtualization has historically been the highest-overhead component, and is the direct subject of the Nitro System discussion in the EC2 document. Four approaches, in increasing order of performance and decreasing order of generality:

| Approach | Mechanism | Performance | Guest awareness needed |
|---|---|---|---|
| **Full device emulation** | Hypervisor (typically via QEMU) presents a fully emulated legacy hardware device (e.g., emulated Intel e1000 NIC, emulated IDE controller); every guest I/O operation traps and is emulated in software | Lowest — every operation is a trap + software emulation | None (works with any unmodified guest driver) |
| **Paravirtualized I/O (virtio)** | Guest installs a **paravirtualized driver** (e.g., `virtio-net`, `virtio-blk`) that communicates with the hypervisor via an efficient shared-memory ring buffer protocol instead of emulating real hardware register-level behavior | Medium-high — avoids emulating slow legacy hardware semantics, but still crosses the guest/hypervisor boundary per I/O batch | Yes (virtio driver in guest) |
| **SR-IOV (Single Root I/O Virtualization)** | A physical PCIe device (typically a NIC) exposes multiple lightweight **Virtual Functions (VFs)**, each of which can be directly assigned to a guest, bypassing the hypervisor's software I/O stack almost entirely for the data path | Near-native | Yes (VF driver in guest) — but no hypervisor mediation needed per-packet |
| **Hardware I/O offload (Nitro Cards model)** | I/O virtualization logic moves off the CPU entirely onto dedicated offload silicon; guest sees a device (via ENA driver, NVMe driver) that talks to purpose-built hardware, not a shared PCIe device on the same die | Near-native, with **zero host CPU tax**, unlike even SR-IOV which still consumes some host resources for VF management | Yes (ENA/NVMe drivers) |

This progression — full emulation → paravirtualized drivers → SR-IOV → dedicated offload hardware — is the direct evolutionary path that leads to the Nitro System covered in the EC2 lecture: Nitro Cards are best understood as a purpose-built, vertically-integrated evolution beyond generic SR-IOV, owned end-to-end by AWS rather than a general PCIe-SIG standard implemented by third-party NIC vendors.

---

## 6. Scheduling in the Hypervisor

The hypervisor's **CPU scheduler** decides which virtual CPU (vCPU) from which guest runs on which physical CPU (pCPU) core at any given time — structurally analogous to an OS process scheduler, but scheduling vCPUs instead of processes/threads.

| Concept | Meaning |
|---|---|
| **vCPU overcommit** | Configuring more total vCPUs across all guests than there are physical CPU cores available. Ratio = $\dfrac{\sum \text{Configured vCPUs}}{\text{Physical Cores}}$. Ratios significantly above 1 increase **CPU ready time** (time a vCPU is runnable but waiting for a physical core). |
| **CPU Ready Time / Steal Time** | The duration a vCPU is queued and ready to execute but cannot be scheduled onto a physical core because all cores are busy with other vCPUs. Reported in Linux guests as `%steal` in tools like `top`. High steal time is the primary observable symptom of CPU overcommitment. |
| **Co-scheduling (Relaxed Co-scheduling)** | For multi-vCPU VMs, ideally all vCPUs of the same VM should be scheduled to run simultaneously (since guest OS code often assumes SMP cores make forward progress together, e.g., spinlocks). Strict co-scheduling (all-or-nothing) can waste physical CPU cycles; most modern hypervisors use *relaxed* co-scheduling, which tolerates limited skew between sibling vCPUs. |
| **NUMA-aware scheduling** | On multi-socket hosts, the hypervisor scheduler tries to keep a VM's vCPUs and memory pinned within a single NUMA (Non-Uniform Memory Access) node, since cross-node memory access latency is significantly higher than local-node access. |

---

## 7. Live Migration

Live migration moves a running VM from one physical host to another with minimal (ideally sub-second, imperceptible) downtime — a capability with no equivalent in physical-server administration and a core value proposition of virtualization for data center operators (maintenance, load balancing, hardware failure avoidance).

### 7.1 Pre-copy live migration (dominant technique)

1. **Iterative memory copy phase**: The VM continues running on the source host while its entire memory image is copied to the destination host over the network. Because the VM is still executing, some pages are modified ("dirtied") during the copy — these are tracked via a **dirty page bitmap** (maintained using the same page-table dirty bits used for standard OS memory management, intercepted by the hypervisor).
2. **Iteration**: After the first full copy pass, the hypervisor re-copies only the pages dirtied during that pass. This repeats, with each iteration copying a (hopefully shrinking) working set of dirtied pages.
3. **Stop-and-copy phase**: Once the dirty set is small enough (below a convergence threshold) or a maximum iteration count is reached, the VM is briefly paused, the final small delta of dirty pages plus CPU/device state is copied, and execution resumes on the destination host.

**Convergence condition (informal):**
$$\text{Migration converges if } \text{Dirty Page Rate} < \text{Network Transfer Rate}$$

If a workload dirties memory faster than the network can transfer pages, pre-copy will not converge, and the hypervisor must fall back to a forced stop-and-copy (longer downtime) or abort the migration.

### 7.2 Post-copy live migration (alternative)

- The VM is paused briefly on the source, minimal CPU/device state plus an empty memory mapping is transferred to the destination, and execution **resumes immediately** on the destination.
- Memory pages are then fetched **on demand** from the source host as the resumed VM touches them (page fault triggers a network fetch), with a background process also proactively pushing remaining pages.
- **Tradeoff vs pre-copy**: Lower total data transferred and a much shorter *initial* pause, but higher per-page-fault latency during the transition window, and a **correctness risk**: if the network link fails mid-migration, the VM state is now split across two hosts with no complete copy on either — pre-copy's independent, most-recent full copy on the source remains valid until the very last step, whereas post-copy has no such safety fallback.

---

## 8. Hypervisor Security

### 8.1 Isolation guarantee and VM Escape

The central security promise of a hypervisor is **isolation**: a compromise inside one guest VM must not allow an attacker to affect the hypervisor itself, the physical host, or any other co-resident guest VM. A **VM escape** vulnerability is any flaw that allows guest-level code to break out of this isolation boundary and execute code at hypervisor or host privilege level. Historically these have most often originated in the **emulated device layer** (e.g., emulated floppy/serial/USB controllers in QEMU), since device emulation code parses complex, guest-controlled input — precisely the class of attack surface that hardware I/O offload (Nitro Cards, SR-IOV) reduces by removing shared software emulation code from the trusted computing base entirely.

### 8.2 Side-channel attacks across co-resident tenants

Because multiple mutually distrusting tenants' VMs may share the same physical CPU cache hierarchy, hypervisors must also consider **microarchitectural side-channel attacks**, where isolation is violated not through a software bug but through observable timing differences in shared hardware resources:

| Attack class | Shared resource exploited |
|---|---|
| Cache timing (Prime+Probe, Flush+Reload) | Shared L2/L3 CPU cache between co-resident vCPUs |
| Spectre / Meltdown family | Speculative execution units, branch predictors shared across privilege boundaries |
| Rowhammer-style | Shared DRAM row buffers |

Mitigations include CPU microcode patches, hypervisor-level scheduling isolation (avoiding co-locating mutually distrusting workloads on sibling hyperthreads — **hyperthread-aware scheduling**), and, at the extreme, dedicating entire physical hosts to a single tenant (AWS's **Dedicated Hosts**, offered as an EC2 purchasing option precisely for this threat model).

### 8.3 Reducing the Trusted Computing Base (TCB)

A recurring theme across modern hypervisor design (and the direct throughline to Nitro): **security improves as the amount of privileged software is reduced**. A monolithic hypervisor with a large device-emulation codebase running at the highest privilege level has a large attack surface. The trend — visible in Xen's move to disaggregate driver domains, in microkernel-style hypervisor design, and taken furthest by Nitro's move of nearly all I/O logic into fixed-function hardware — is to minimize the amount of code running with the ability to violate isolation, since a hardware ASIC executing a fixed function has no exploitable general-purpose instruction execution surface the way software does.

---

## 9. Suggested Numerical / Formula-Based Problem Bank

1. **vCPU overcommit**: Given a host with $N$ physical cores running $k$ guest VMs each configured with $v_i$ vCPUs, compute the overcommit ratio, and given an observed average `%steal` value, discuss whether the host is under scheduling pressure.
2. **Memory overcommit**: Given physical host RAM and a list of guest-configured memory sizes, compute the overcommit ratio; given a target maximum ratio, compute how many additional guests of a given size can be safely admitted.
3. **EPT/NPT page walk cost**: Given a 4-level guest page table and a 4-level hypervisor (nested) page table, compute the worst-case number of memory accesses required to resolve a single TLB miss under two-dimensional paging, and compare against native (single-level walk) and shadow-paging (single effective walk, but with VM Exit cost per guest page table write) — as a cost-per-access-pattern comparison exercise.
4. **Live migration convergence**: Given total guest memory size, initial full-copy transfer rate, and a page dirty rate (pages/second) with a fixed average page size, compute whether pre-copy converges within $n$ iterations, and if so, the total data transferred and the final stop-and-copy downtime.
5. **I/O virtualization overhead comparison**: Given per-operation overhead (in microseconds) for full emulation, paravirtualized I/O, and SR-IOV/hardware-offload for a fixed I/O operation rate (ops/sec), compute total CPU time consumed per second by each approach and the resulting host CPU capacity left available for guest compute.

---

## 10. Summary Table — Hypervisor Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Definition | Popek & Goldberg conditions | Equivalence, Resource Control, Efficiency — the formal bar any VMM must clear |
| Architecture | Type-1 vs Type-2 | Bare-metal privileged control vs hosted-on-general-OS; KVM is an architecturally hybrid Type-1-in-kernel case |
| CPU virtualization | Binary Translation → Paravirtualization → Hardware-Assisted (VT-x/AMD-V) | x86's non-trapping sensitive instructions forced software workarounds until hardware added a dedicated root/non-root execution mode |
| Memory virtualization | Shadow Page Tables → EPT/NPT | Software-maintained shadow tables trade VM Exit frequency for a more expensive two-dimensional hardware page walk |
| Memory scaling | Ballooning, KSM/TPS, hypervisor swap | Techniques to safely oversubscribe physical RAM across guests, ranked by how much they cooperate with guest-level memory management |
| I/O virtualization | Full emulation → virtio → SR-IOV → Nitro-style hardware offload | Same "move work out of software and off the CPU" trajectory that defines the entire hypervisor evolution story |
| Scheduling | vCPU scheduling, ready/steal time, NUMA-awareness | Structurally an OS scheduler problem, one level up, with added SMP co-scheduling and NUMA locality concerns |
| Mobility | Pre-copy vs Post-copy live migration | Pre-copy is safer (always has a valid full copy) but may not converge under high dirty rates; post-copy trades this safety for lower upfront transfer |
| Security | VM escape, side channels, TCB minimization | Isolation is the hypervisor's core promise; the entire architectural trend is toward shrinking privileged software surface area |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 9 reserved as a follow-up tutorial/assignment. Designed to be read as a direct prerequisite companion to the "Amazon EC2 — Technical Lecture Notes" document — Sections 1, 3, 5, and 8 map directly onto the Nitro System discussion there.*
