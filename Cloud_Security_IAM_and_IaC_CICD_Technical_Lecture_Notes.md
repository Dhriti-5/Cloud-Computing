# Cloud Security, IAM & Infrastructure-as-Code / CI-CD — Technical Lecture Notes
### BE IV — Computer Science and Engineering | Cloud Computing
### Companion document to: "Amazon EC2", "Hypervisors", "Cloud Networking (VPC Deep Dive)", "Cloud Storage & Database Services", and "Containers, Orchestration & Serverless" — Technical Lecture Notes
### Prerequisite recap assumed: Nitro Security Chip (Hypervisor/EC2 notes), IAM Instance Profiles & IMDSv2 (EC2 notes §13), Security Groups/NACLs (VPC notes §7), all prior documents' control-loop/declarative-reconciliation pattern

---

## 1. Positioning Security, IAM, and IaC/CI-CD in the Stack

Every prior document in this set covered a **resource** (compute, network, storage, containers) and how AWS provisions/scales it. This document covers two **cross-cutting layers** that sit *around* every resource discussed so far, rather than being a resource in their own right:

```
                     Security & IAM (WHO can do WHAT, to WHICH resource, under WHAT conditions)
                              |
   applies uniformly across: EC2 | VPC | S3/RDS/DynamoDB | ECS/EKS/Lambda   (all four prior documents)
                              |
             IaC & CI/CD (HOW resources are declared, versioned, and safely changed over time)
```

**Formal framing**: IAM answers an **access-control** question (authorization); IaC/CI-CD answers a **change-management** question (how state transitions are proposed, validated, and applied). Both are, at their core, the same kind of problem already seen repeatedly in this document series — a **declarative target state**, reconciled against **actual state**, via an automated control loop (EC2 notes §11 Auto Scaling, Storage notes §2.3 S3 Lifecycle, Containers notes §6.2 Kubernetes controllers) — IaC tools are simply that same reconciliation pattern applied to the **entire infrastructure definition**, not just one resource's capacity.

### 1.1 The AWS Shared Responsibility Model

| Responsibility | AWS ("security **of** the cloud") | Customer ("security **in** the cloud") |
|---|---|---|
| Physical data center security | AWS | — |
| Hypervisor/Nitro System isolation | AWS (Hypervisor notes §8, EC2 notes §2.2 Nitro Security Chip) | — |
| Host OS patching (managed services: RDS, Fargate, Lambda) | AWS | — |
| Guest OS patching (EC2, self-managed on EC2) | — | Customer |
| Network configuration (VPC, Security Groups, NACLs) | Provides the primitives | Customer configures them (VPC notes §7) |
| IAM policy configuration | Provides the service | Customer defines least-privilege policies (this document, §2–5) |
| Data encryption | Provides KMS/encryption mechanisms | Customer decides what to encrypt and manages key policy (§7) |
| Application-layer security | — | Customer (input validation, dependency management, etc.) |

This table is the formal generalization of a distinction already made implicitly in every prior document: AWS secures the **mechanism** (Nitro, EBS replication, S3 durability), the customer secures the **configuration and usage** of that mechanism — the line moves further toward "AWS responsible" as a service becomes more managed (compare: raw EC2 vs Fargate vs Lambda, Containers notes §11), which is itself a re-statement of the "managed convenience vs customer control" tradeoff recurring throughout this series.

---

## 2. IAM — Formal Model

### 2.1 Definition

**AWS Identity and Access Management (IAM)** is a Regionless (global), free AWS service answering exactly one question for every API call made to any AWS service: **"Is this specific principal authorized to perform this specific action, on this specific resource, under these specific conditions, right now?"**

Formally, an authorization decision is a function over a 4-tuple:

$$\text{Decision} = f(\text{Principal}, \text{Action}, \text{Resource}, \text{Condition Context})$$

- **Principal**: the identity making the request (IAM User, IAM Role, federated identity, or an AWS service acting on the customer's behalf)
- **Action**: the specific API operation (e.g., `s3:GetObject`, `ec2:RunInstances`)
- **Resource**: the specific AWS resource ARN(s) the action targets
- **Condition Context**: request-time attributes (source IP, MFA presence, time of day, tags, encryption requirements, etc.)

### 2.2 IAM Users, Groups, Roles — comparison table

| Entity | Has long-term credentials? | Assumed or logged into? | Typical use |
|---|---|---|---|
| **IAM User** | Yes (password and/or access keys) | Logged in directly | A specific human or a legacy application needing long-term credentials (increasingly discouraged) |
| **IAM Group** | No (not a principal itself) | N/A — a collection of Users | Attaching a common policy set to many Users at once |
| **IAM Role** | **No** — only ever **temporary** credentials, issued via STS (§3) | **Assumed** — by a human, an application, an EC2 instance, or another AWS service | The AWS-recommended default for **any** machine identity (EC2 Instance Profiles, EC2 notes §13; ECS Task Roles, Containers notes §5.1; Lambda execution roles) |

**Architectural reasoning — why Roles are preferred over Users for workloads:** long-term IAM User access keys, once created, are a static secret that must be manually rotated and can be exfiltrated and reused indefinitely if leaked. This is the exact same class of problem the EC2 notes' IMDSv2 discussion addresses at the credential-delivery layer (EC2 notes §13): a Role's temporary credentials are **automatically rotated** (typically hourly) by STS, meaning a leaked credential has a bounded, short blast-radius window — this is the identity-layer instance of the same "minimize the value/lifetime of any single leaked secret" principle that motivates Nitro's hardware-enforced TCB minimization (Hypervisor notes §8.3) and IMDSv2's token-based session model (EC2 notes §13).

### 2.3 Policy documents — structure

An IAM policy is a JSON document composed of one or more **statements**, each with:

```json
{
  "Effect": "Allow" | "Deny",
  "Action": ["service:ActionName", ...],
  "Resource": ["arn:aws:...", ...],
  "Condition": { "ConditionOperator": { "ConditionKey": "Value" } }
}
```

### 2.4 Policy types — comparison table

| Policy type | Attached to | Scope | Typical use |
|---|---|---|---|
| **Identity-based policy** | IAM User, Group, or Role | Grants what that identity can do | The default, most common policy type |
| **Resource-based policy** | The resource itself (e.g., an S3 bucket policy, a KMS key policy) | Grants who can access that specific resource, **including principals from other AWS accounts** | Cross-account access without needing the other account to assume a Role |
| **Permissions boundary** | An IAM User or Role | A **ceiling** — the *maximum* permissions that identity can ever have, regardless of what identity-based policies grant it | Delegated administration — allow a team to create their own Roles, but cap what those self-created Roles can ever do |
| **Service Control Policy (SCP)** | An AWS Organizations OU or account | A ceiling applied **organization-wide**, across every IAM entity in every account under that OU | Guardrails at the organization level (e.g., "no account in this OU may ever disable CloudTrail," §9.1) |
| **Session policy** | Passed at `AssumeRole` time (§3) | A further ceiling for just that one temporary session | Narrowing an already-broad Role's permissions for one specific, time-boxed task |

---

## 3. Policy Evaluation Logic — the Formal Algorithm

This is the single most quantitatively/logically rich, and most frequently misunderstood, part of IAM — directly analogous in spirit to the VPC notes' NACL evaluation-order discussion (VPC notes §6.1, §7.2), but with a stricter, non-order-dependent precedence rule.

### 3.1 The evaluation algorithm

For a given request, AWS evaluates **every applicable policy of every applicable type** (identity-based, resource-based, permissions boundary, SCP, session policy) and combines them using this precedence, in order:

1. **Explicit Deny anywhere → final decision is Deny.** (An explicit Deny in *any* applicable policy — identity-based, resource-based, SCP, boundary, or session policy — cannot be overridden by an Allow anywhere else.)
2. **Organization SCP must Allow (or have no explicit Deny)** — an SCP that does not explicitly allow the action results in an implicit Deny at the organization boundary, regardless of what the account-level identity policy says.
3. **Permissions boundary / session policy must Allow** — if present, they must intersect (logical AND) with the identity-based Allow.
4. **At least one identity-based OR resource-based policy must explicitly Allow.**
5. **Default: implicit Deny** — if no statement anywhere explicitly allows the action, the request is denied by default (IAM is **default-deny**, the same "deny all inbound by default" posture as a default Security Group, VPC notes §7.1, but applied to *every* action by *every* principal, not just network traffic).

**Formalized as a boolean expression**, letting $D$ = "any explicit Deny exists," $A_{id}$ = "an identity-based policy Allows," $A_{res}$ = "a resource-based policy Allows," $B$ = "no permissions boundary present, OR boundary Allows," $S$ = "no SCP present, OR SCP Allows," $P$ = "no session policy present, OR session policy Allows":

$$\text{Access Granted} = \neg D \;\wedge\; S \;\wedge\; B \;\wedge\; P \;\wedge\; (A_{id} \vee A_{res})$$

**Exam-relevant consequence**: because an explicit Deny **always wins**, the safest and most common enterprise guardrail pattern is an SCP with a small number of high-value explicit Deny statements (e.g., "Deny `iam:CreateAccessKey` outside an approved condition") — this single Deny cannot be overridden by *any* combination of Allow statements anywhere else in the account, including an account administrator's own broad `AdministratorAccess` identity policy.

### 3.2 Least privilege — formalized as a permission-surface minimization problem

**Least privilege** is the principle that a principal should hold the **minimum set of Actions × Resources** necessary for its function, and no more. This can be stated as a formal minimization objective: given a required task set $T$ (the actions a principal *must* be able to perform to do its job) and a granted permission set $G$ (what its attached policies actually allow), the **excess permission surface**:

$$\text{Excess Surface} = |G| - |G \cap T| = |G \setminus T|$$

A well-designed policy minimizes $|G \setminus T|$ toward zero — every permission granted beyond what is strictly required is pure, unused **attack surface**, structurally the same reasoning as the Hypervisor notes' TCB-minimization principle (Hypervisor notes §8.3) and the general "reduce the amount of code/privilege that *could* be misused, even if it currently isn't" theme running through this entire document series.

---

## 4. AWS Security Token Service (STS) — Temporary Credentials

### 4.1 `AssumeRole` mechanics

When a principal calls `sts:AssumeRole` against a target Role, STS validates the Role's **trust policy** (a resource-based policy on the Role itself, specifying *who* is allowed to assume it — a distinct document from the Role's *permissions* policy, which specifies what the assumed session can *do*), and, if permitted, issues a **temporary credential set**: an Access Key ID, a Secret Access Key, and a **Session Token**, all with a bounded expiry.

$$\text{Credential validity window} = [\text{Issuance Time},\ \text{Issuance Time} + \text{Requested Duration}]$$

subject to $\text{Requested Duration} \leq \text{MaxSessionDuration}$ (a Role-level configurable ceiling, default 1 hour, configurable up to 12 hours).

### 4.2 Where this mechanism is *already* used elsewhere in this document series

STS `AssumeRole` is not a standalone feature students need to learn in isolation — it is the **literal mechanism underneath** several constructs already covered:

| Prior mechanism | How it uses STS/AssumeRole |
|---|---|
| EC2 Instance Profile / IMDS credential delivery (EC2 notes §13) | The Nitro-exposed IMDS endpoint is, under the hood, periodically calling STS on the instance's behalf to refresh temporary credentials before they expire |
| ECS Task Role / EKS IAM Roles for Service Accounts (Containers notes §5.1) | Each Task/Pod's credentials are STS-issued temporary credentials scoped to that Task's Role |
| Lambda execution role (Containers notes §10) | Every Lambda invocation's AWS SDK calls are authorized using STS-issued temporary credentials tied to the function's execution role |
| Cross-account access | An account B administrator assumes a Role in account A (trust policy in A names account B as a trusted principal) |

### 4.3 Federation

STS also issues temporary credentials for **federated identities** — users authenticated by an external identity provider (SAML 2.0, OIDC, or AWS IAM Identity Center) rather than an IAM User — meaning a corporate directory (e.g., Active Directory, Okta) can grant AWS access without ever creating a long-term IAM User at all, closing the same "avoid long-term static credentials wherever possible" gap discussed in §2.2.

---

## 5. Encryption — KMS and the Envelope Encryption Model

### 5.1 Why envelope encryption, not direct data-key encryption

Encrypting a large object (e.g., a multi-GB S3 object, or an entire EBS volume) directly with a KMS-managed key would require sending the **full plaintext** to KMS for every encrypt/decrypt operation — infeasible at scale and a KMS API throughput bottleneck (§5.3). AWS KMS instead uses **envelope encryption**:

1. The calling service (S3, EBS, RDS) requests a **data key** from KMS, specifying which **Customer Master Key (CMK)** to use.
2. KMS generates a random data key, returns **two versions**: the **plaintext data key** (used immediately, then discarded from memory) and a **KMS-encrypted (wrapped) copy of that same data key**.
3. The plaintext data key encrypts the actual data **locally**, at the service's own infrastructure — no further KMS API call needed for the bulk encryption operation itself.
4. Only the small, wrapped data key (not the bulk data) is stored alongside the encrypted data.
5. On decrypt, the wrapped data key is sent back to KMS (a small, fast call), KMS unwraps it using the CMK, returns the plaintext data key, which is then used locally to decrypt the bulk data.

This is structurally the same "amortize an expensive/centralized operation by doing the bulk of the work locally, only touching the centralized authority for the small, security-critical step" pattern as Aurora's redo-log-only replication (Storage notes §6.2, replicating only the log stream rather than full data pages) — both minimize the volume of data that must cross a slower/more centralized boundary.

### 5.2 KMS Key types — comparison table

| Key type | Key material origin | Rotation | Typical use |
|---|---|---|---|
| **AWS owned key** | Fully AWS-managed, not visible in the customer's account at all | AWS-managed | Default encryption for some services with no customer key-management need (lowest control, zero operational burden) |
| **AWS managed key** (`aws/s3`, `aws/ebs`, etc.) | Created automatically the first time a service needs one, visible in the customer's KMS console | Automatic, annual, AWS-managed | Default "SSE-KMS with an AWS managed key" — encryption at rest with minimal setup |
| **Customer managed key (CMK)** | Customer explicitly creates it | Optional automatic annual rotation, or manual | Fine-grained key policy control, cross-account sharing, audit-specific requirements, ability to disable/schedule deletion |
| **Customer-provided key (SSE-C, S3 only)** | Customer supplies the raw key material on every request; AWS never stores it | Customer's full responsibility | Regulatory requirements mandating AWS never possess the key material at all |

### 5.3 KMS request quotas — the quantitative core

KMS API calls (`GenerateDataKey`, `Decrypt`, `Encrypt`) are subject to a **Region-level, key-type-dependent requests-per-second quota** (e.g., several thousand to tens of thousands of requests/second for symmetric CMKs, varying by KMS key type and Region) — this is the encryption-layer instance of the same **hard per-resource throughput ceiling** pattern seen at DynamoDB partitions (Storage notes §7.2) and Lambda concurrency (Containers notes §10.4): a workload performing envelope encryption/decryption at very high request rates (e.g., encrypting every individual small object in a high-throughput pipeline) can be **KMS-throttled** even though the underlying storage/compute service has ample capacity — exactly the same "hard ceiling on one dimension causes throttling despite spare aggregate capacity elsewhere" theme recurring for the fourth time across this document series.

**Effective throughput ceiling formula**, given a KMS quota $Q$ (requests/sec) and $c$ = KMS calls required per logical operation (1 for envelope-encrypt-once-per-large-object patterns; potentially 1-per-record for naive small-object/record-level encryption):

$$\text{Max sustainable operation rate} = \frac{Q}{c}$$

This is why S3 Bucket Keys (a KMS optimization reducing the *effective* number of KMS calls for S3 SSE-KMS by reusing a bucket-level data key for a short time window rather than calling KMS per-object) exist — they directly reduce $c$, raising the achievable operation rate under the same fixed KMS quota $Q$.

---

## 6. Network-Layer Security — Extending the VPC Notes

The VPC notes already cover Security Groups and NACLs in full (VPC notes §7). This section adds the **edge/application-layer** security services that sit in front of a VPC.

| Service | OSI Layer / scope | Function |
|---|---|---|
| **AWS WAF (Web Application Firewall)** | Layer 7 (HTTP/HTTPS), attached to an ALB, API Gateway, or CloudFront distribution | Rule-based filtering of HTTP requests (SQL injection patterns, XSS patterns, rate-based rules, geographic restrictions) — the Layer-7-aware counterpart to a Security Group, which only inspects Layer 3/4 headers (VPC notes §7) |
| **AWS Shield Standard** | Layer 3/4, automatic, free, applied to all AWS customers | Automatic mitigation of common, high-volume network/transport-layer DDoS patterns (SYN floods, UDP reflection) |
| **AWS Shield Advanced** | Layer 3/4 and Layer 7 | Paid tier: enhanced detection, 24/7 DDoS Response Team access, cost protection against scaling charges incurred *during* an attack, deeper integration with WAF for automatic rule deployment |
| **AWS Network Firewall** | Layer 3–7, deployed inline within a VPC (via Gateway Load Balancer, VPC notes §5's GENEVE-encapsulated appliance-insertion pattern) | Stateful, deep-packet-inspection firewall with intrusion-detection/prevention rule support — the VPC notes' Gateway Load Balancer discussion (§12 of the EC2 notes' ELB section, and VPC notes' general appliance-insertion reasoning) is the literal mechanism Network Firewall is built on |

**Layered defense summary, extending the VPC notes' defense-in-depth reasoning (VPC notes §7.1):** WAF/Shield/Network Firewall sit **in front of** the Security Group/NACL layers already covered, giving a full stack: DDoS mitigation (Shield) → application-layer filtering (WAF) → deep packet inspection (Network Firewall) → subnet-level stateless filtering (NACL) → instance-level stateful filtering (Security Group) → OS/application-level controls — each layer independently enforced, so a bypass or misconfiguration at any one layer does not eliminate the others, precisely the TCB-minimization-adjacent "no single point of enforcement failure" reasoning already established for Nitro (Hypervisor notes §8.3) and IAM's explicit-Deny precedence (§3.1).

---

## 7. Audit, Detection, and Compliance

| Service | Function | Rough analogue elsewhere in this document set |
|---|---|---|
| **AWS CloudTrail** | Records **every API call** made in the account (who, what, when, from where, success/failure) as an immutable event log | The IAM/API-layer equivalent of VPC Flow Logs (VPC notes §12) — metadata-only, aggregated-and-durable record of activity, not the request/response payload itself |
| **AWS Config** | Continuously records the **configuration state** of resources over time, and evaluates them against declared compliance rules (e.g., "no S3 bucket may be public") | The reconciliation-loop pattern again — Config is a *read-only, alerting* control loop (detects and reports drift) rather than a *corrective* one (unlike Auto Scaling or Kubernetes controllers, which actively correct drift) |
| **Amazon GuardDuty** | ML/threat-intelligence-based anomaly detection across CloudTrail, VPC Flow Logs, and DNS logs (e.g., detecting IAM credential exfiltration patterns, cryptomining network signatures, unusual API call sequences) | Analogous to an intrusion-detection layer sitting *above* all the raw log sources already discussed, correlating across them rather than replacing any single one |
| **AWS Security Hub** | Aggregates findings from GuardDuty, Config, Inspector, and third-party tools into a single, prioritized dashboard, scored against standard compliance frameworks (CIS AWS Foundations Benchmark, PCI-DSS, etc.) | A single-pane-of-glass aggregation layer over the other services in this table |

---

## 8. Infrastructure as Code (IaC) — Formal Definition and Historical Reasoning

### 8.1 The three eras of infrastructure provisioning

| Era | Method | Core problem |
|---|---|---|
| **Manual** | Console clicks, SSH-and-configure by hand | Not repeatable, not auditable, "snowflake" servers that drift from each other over time, cannot be version-controlled |
| **Imperative scripting** | Shell scripts, Ansible-style step sequences (`RunInstances`, then `CreateVolume`, then `AttachVolume`, in a fixed order) | Repeatable, but **not idempotent by default** — re-running the same script against an already-provisioned environment may error out or create duplicate resources, since the script describes a *sequence of actions*, not a *target state* |
| **Declarative IaC** | Terraform, CloudFormation, CDK — a file describing the **desired end state**; the tool computes and applies whatever *diff* of actions is needed to reach it | Idempotent by construction: running the same declaration twice against an already-correct environment is a no-op, and the tool — not the human — computes the action sequence |

This progression (manual → imperative → declarative) is the **infrastructure-provisioning-layer instance of the exact same reconciliation-loop shift** already seen at the resource level throughout this series: EC2 Auto Scaling (EC2 notes §11), S3 Lifecycle policies (Storage notes §2.3), and Kubernetes controllers (Containers notes §6.2) are all "declare the target, let a control loop continuously reconcile toward it" systems operating on a *single resource type*; declarative IaC is the same pattern generalized to the **entire infrastructure graph** across every service in this document series simultaneously.

### 8.2 State — the essential IaC concept

A declarative IaC tool must track **what it believes it last provisioned** (its "state") in order to compute a diff against the desired configuration on the next run. This state file is itself a critical, sensitive artifact (it may contain resource IDs, and in some tooling, even sensitive attribute values) requiring its own durability and locking strategy — a direct, literal application of concepts from the Storage notes: state is typically stored in **S3** (durability, Storage notes §2.4) with **DynamoDB** used as a **distributed lock** (Storage notes §7, preventing two concurrent `terraform apply` runs from corrupting the state file through a race condition) — this is a genuinely common real-world Terraform-on-AWS backend pattern, and a clean illustration of composing two previously-covered services to solve a new problem.

### 8.3 CloudFormation vs Terraform vs CDK — comparison table

| Property | AWS CloudFormation | Terraform | AWS CDK (Cloud Development Kit) |
|---|---|---|---|
| Provider scope | AWS only | Multi-cloud (AWS, Azure, GCP, hundreds of providers) via a plugin model | AWS only (also supports CDK for Terraform / cdktf as a hybrid) |
| Configuration language | Declarative JSON/YAML | Declarative HCL (HashiCorp Configuration Language) | **Imperative, general-purpose languages** (TypeScript, Python, Java, etc.) that **synthesize** a declarative CloudFormation template underneath |
| State management | Managed entirely by AWS (no separate state file the customer must store/secure) | Customer-managed state file (commonly S3 + DynamoDB lock, §8.2) | Delegates entirely to CloudFormation's state management (inherits CFN's model) |
| Drift detection | Native (`DetectStackDrift`) | Available (`terraform plan` detects drift against real infrastructure) | Inherits CloudFormation's drift detection |
| Rollback on failure | Automatic (CloudFormation rolls the stack back to its last known-good state on failure) | Not automatic by default — a failed `apply` can leave infrastructure in a partially-changed state requiring manual intervention | Inherits CloudFormation's automatic rollback |
| Learning curve / expressiveness | Verbose for complex logic (loops/conditionals are awkward in pure declarative YAML) | Full HCL expressiveness (loops, conditionals, modules) while remaining declarative | Full general-purpose programming language expressiveness (loops, classes, package management, unit testing the infrastructure code itself) |
| Best for | Teams fully AWS-committed, wanting zero external state-management burden | Multi-cloud environments, teams preferring a single tool across providers | Teams wanting to define infrastructure using the same language/tooling as their application code, with strong type-checking and abstraction/reuse |

---

## 9. CI/CD — AWS-Native Pipeline Services

### 9.1 The pipeline stages and their AWS services

| Stage | AWS service | Function |
|---|---|---|
| Source | CodeCommit (or GitHub/Bitbucket via integration) | Version-controlled source of truth, triggers the pipeline on commit |
| Build | **CodeBuild** | Compiles code, runs unit tests, builds container images (pushing to ECR, Containers notes §3.2), all inside a fully-managed, ephemeral build environment (itself a container/Fargate-style execution environment — a direct reuse of the Containers notes' serverless-compute reasoning, §9–10) |
| Test | CodeBuild (or a dedicated stage) | Integration tests, security/dependency scanning |
| Deploy | **CodeDeploy** | Orchestrates the actual rollout to EC2 instances, ECS Services, or Lambda functions, per a configured deployment strategy (§9.2) |
| Orchestration | **CodePipeline** | The overall workflow engine tying Source → Build → Test → Deploy stages together, with manual approval gates optionally inserted between any stages |

### 9.2 Deployment strategies — comparison table, with formalized blast-radius reasoning

| Strategy | Mechanism | Downtime | Blast radius on failure | Infra cost during deployment |
|---|---|---|---|---|
| **In-place (rolling)** | Existing instances/Tasks updated in batches, old version stopped before new version starts on each | Brief, per-batch | Partial — a bad deployment affects whichever batch is mid-rollout when the failure is detected | No extra (same fleet reused) |
| **Blue/Green** | A complete **second, parallel environment** ("Green") is provisioned with the new version; traffic is cut over (often instantly, via DNS or load balancer target group swap) once Green is validated; old environment ("Blue") is kept briefly for instant rollback | Zero (traffic cutover is near-instantaneous) | Zero for a failed *deployment* (Green is validated before any traffic shifts) — but requires validation to actually catch the failure before cutover | **2×** — both environments run simultaneously during the transition window |
| **Canary** | A small percentage $p$ of traffic is shifted to the new version first; if healthy, traffic is ramped up in further increments until 100% | Zero | **Bounded to $p$** of total traffic/users for the canary window — the core quantitative advantage over in-place rolling | Slightly elevated only during the canary window (both versions serve production traffic briefly) |

### 9.3 Blast radius — formalized

For a canary deployment shifting an initial fraction $p$ of traffic to a new version, and given a total request volume $R$ over the canary observation window:

$$\text{Requests exposed to a bad deployment} = p \times R$$

This is the deployment-layer version of the general "bound the impact of a failure to the smallest possible subset before committing further" principle already seen in this series in a different form at the Storage notes' quorum-availability reasoning (Storage notes §8.3, tolerating partial replica loss) and the VPC notes' Spread Placement Group fault-isolation goal (EC2 notes §9, referenced there) — canary deployment is essentially **blast-radius limitation applied to code changes**, the same way Spread Placement Groups apply it to hardware failures.

### 9.4 Statistical validity of a canary — minimum sample size

A canary's purpose is to detect whether the new version's error rate is meaningfully worse than the old version's, **before** committing to a full rollout. Given a baseline error rate $p_0$, a minimum detectable increase $\delta$ worth flagging, and a desired statistical confidence (Type I error rate $\alpha$, typically 0.05, and power $1-\beta$, typically 0.8), the required sample size per arm (approximate, using the standard two-proportion z-test sample-size formula) is:

$$n \approx \frac{\left(z_{\alpha/2}\sqrt{2\bar{p}(1-\bar{p})} + z_{\beta}\sqrt{p_0(1-p_0) + p_1(1-p_1)}\right)^2}{\delta^2}$$

where $p_1 = p_0 + \delta$ and $\bar p = (p_0+p_1)/2$. The practical, exam-relevant takeaway (rather than the derivation): **a canary window sized to too few requests cannot statistically distinguish "the new version is genuinely worse" from "normal random variation,"** meaning canary percentage $p$ and observation-window duration must be jointly large enough, given the traffic volume $R$, to accumulate at least this minimum sample size before a promote/rollback decision is trustworthy — this is the direct probabilistic analogue of the Storage notes' durability-formula reasoning (Storage notes §2.4): both convert a real operational question into an explicit, computable statistical quantity rather than a qualitative judgment call.

---

## 10. Secrets Management

| Service | Rotation | Fine-grained versioning | Cost model | Typical use |
|---|---|---|---|---|
| **AWS Secrets Manager** | Native automatic rotation (via a Lambda rotation function, e.g., for RDS credentials) | Yes, full version history with staging labels (`AWSCURRENT`, `AWSPREVIOUS`) | Per-secret monthly charge + per-API-call charge | Database credentials, API keys needing scheduled rotation |
| **AWS Systems Manager Parameter Store** (Standard tier) | No native automatic rotation (must be built manually, e.g., via EventBridge + Lambda) | Basic versioning | **Free** for Standard tier parameters | Configuration values, feature flags, and secrets where automatic rotation is not required |
| **Parameter Store (Advanced tier)** | Same manual-rotation caveat as Standard | Higher size/throughput limits, supports parameter policies (e.g., expiration) | Per-parameter monthly charge | Larger-scale secret/config management needing higher throughput than Standard tier |

Both integrate with KMS (§5) for encryption at rest — the secret's plaintext value is itself encrypted using the same envelope-encryption model described in §5.1, meaning Secrets Manager is architecturally a thin, purpose-built application layer on top of KMS + a versioned datastore, rather than a wholly separate encryption mechanism.

---

## 11. Cross-Reference Summary — How This Document Connects to All Four Prior Documents

| Concept in this document | Connects to |
|---|---|
| IAM Roles / temporary credentials via STS | EC2 notes §13 (Instance Profiles, IMDSv2) — this document formalizes the general mechanism EC2 notes introduced narrowly |
| Explicit-Deny-wins policy evaluation | VPC notes §6.1/§7.2 (NACL rule ordering) — a related but distinct precedence model, contrasted explicitly in §3.1 |
| Least privilege / permission-surface minimization | Hypervisor notes §8.3 (TCB minimization) — same "minimize what *could* be misused" principle, applied to IAM instead of hypervisor code surface |
| Envelope encryption (KMS) | Storage notes §6.2 (Aurora's log-only replication) — same "keep the bulk operation local, touch the centralized/trusted authority only for the small critical step" pattern |
| KMS request quota throttling | Storage notes §7.2 (DynamoDB hot partitions), Containers notes §10.4 (Lambda concurrency) — the third instance of "hard per-resource ceiling causes throttling despite spare aggregate capacity" |
| CloudTrail as metadata-only audit log | VPC notes §12 (VPC Flow Logs) — same "record connection/call metadata, not payload" design |
| AWS Config as a read-only reconciliation/detection loop | EC2 notes §11 (Auto Scaling), Storage notes §2.3 (Lifecycle), Containers notes §6.2 (K8s controllers) — same control-loop pattern, but detect-only rather than corrective |
| Network Firewall via Gateway Load Balancer | VPC notes' appliance-insertion / GENEVE encapsulation discussion — literal reuse of the same mechanism |
| IaC state file in S3 + DynamoDB lock | Storage notes §2 (S3 durability) and §7–8 (DynamoDB, quorum consistency) — a direct, practical composition of two previously-covered services |
| CodeBuild's ephemeral, managed build environment | Containers notes §9–10 (Fargate/Lambda serverless execution model) — architecturally the same "AWS provisions the exact compute a job needs, on demand" pattern |
| Blue/Green deployment's 2× infra cost during cutover | EC2 notes §10 (pricing models) — the same kind of explicit cost-vs-risk tradeoff reasoning applied to deployment strategy instead of purchasing model |
| Canary blast-radius formula | Storage notes §8.3 (quorum availability under partial failure), EC2 notes §9 (Placement Group fault isolation) — same "bound the impact of a failure to a known subset" principle, at the deployment layer |

---

## 12. Suggested Numerical / Formula-Based Problem Bank

1. **IAM policy evaluation**: Given a principal with an identity-based policy allowing an action, a resource-based policy on the target resource, a permissions boundary, and an SCP at the OU level — with one of these containing an explicit Deny — determine the final access decision, and identify which single policy change would be required to grant access (if possible at all).
2. **Least privilege excess surface**: Given a Role's granted permission set $G$ (listed as a set of Action×Resource pairs) and the actual task-required set $T$, compute the excess permission surface $|G \setminus T|$, and determine the minimal set of statements to remove from the policy to achieve $G = T$.
3. **KMS throughput ceiling**: Given a data pipeline needing to encrypt $N$ small records/second, a chosen encryption pattern (per-record KMS call vs a bucket/batch-key pattern reducing calls by a factor of $k$), and the Region's KMS requests-per-second quota, determine whether the pipeline will be throttled under each pattern, and the minimum batching factor $k$ required to avoid throttling.
4. **Canary sample size and blast radius**: Given a baseline error rate $p_0$, a minimum detectable increase $\delta$ worth flagging, standard $\alpha=0.05$/power$=0.8$ assumptions, and a known request rate $R$ (requests/sec), compute the required canary sample size using the two-proportion sample size formula, the resulting minimum canary observation window duration at a chosen canary percentage $p$, and the number of users exposed to a bad deployment during that window.
5. **Blue/Green cost vs in-place risk**: Given the hourly cost of the production fleet, the typical cutover window duration for a Blue/Green deployment, and an estimated cost-of-a-bad-in-place-deployment (in terms of affected users × average revenue per user × incident duration), compute the total cost of running Blue/Green for the cutover window versus the expected cost of an in-place rollout's failure risk, and determine at what estimated failure probability the two strategies break even.

---

## 13. Summary Table — Security, IAM & IaC/CI-CD Concept Map for Revision

| Layer | Concept | Key idea to retain |
|---|---|---|
| Foundational | Shared Responsibility Model | AWS secures the mechanism; the customer secures configuration/usage — the line shifts toward AWS as services become more managed |
| Identity | Users vs Groups vs Roles | Roles + temporary STS credentials are the default for any workload identity — bounded blast radius via automatic rotation |
| Policy | Identity-based, resource-based, boundary, SCP, session policy | Five policy types combine via one algorithm: explicit Deny always wins, default is implicit Deny |
| Least privilege | Excess permission surface $\|G \setminus T\|$ | Same TCB-minimization philosophy as Nitro/hypervisor security, applied to IAM |
| Credentials | STS / AssumeRole | The literal mechanism underneath EC2 Instance Profiles, ECS Task Roles, and Lambda execution roles |
| Encryption | KMS envelope encryption | Bulk data encrypted locally with a plaintext data key; only the small wrapped key touches the centralized KMS authority |
| Encryption limits | KMS request quotas | Same "hard per-resource throughput ceiling" pattern as DynamoDB partitions and Lambda concurrency |
| Network edge | WAF / Shield / Network Firewall | Layer 7 / DDoS / deep-packet-inspection layers sitting in front of Security Groups and NACLs — defense in depth extended outward |
| Audit | CloudTrail, Config, GuardDuty, Security Hub | API-call log (metadata-only, like Flow Logs) + config-drift detection (read-only control loop) + anomaly correlation + aggregation dashboard |
| IaC philosophy | Manual → Imperative → Declarative | Declarative IaC is the reconciliation-loop pattern (ASG, Lifecycle, K8s controllers) generalized to the whole infrastructure graph |
| IaC tooling | CloudFormation vs Terraform vs CDK | AWS-native/no external state vs multi-cloud/customer-managed state vs general-purpose-language-authored declarative templates |
| CI/CD | CodePipeline/CodeBuild/CodeDeploy | Managed, ephemeral build environments reuse the Fargate/Lambda serverless-execution pattern |
| Deployment strategy | Rolling vs Blue/Green vs Canary | Downtime vs cost vs blast-radius tradeoff; canary bounds blast radius to $p \times R$ requests, statistically validated via sample-size formulas |
| Secrets | Secrets Manager vs Parameter Store | Automatic rotation + versioning vs free/simple config storage; both are a thin layer over KMS envelope encryption |

---

*End of lecture notes. Structured for a single 60–75 minute technical lecture session, with Section 12 reserved as a follow-up tutorial/assignment. Designed to be read as the sixth and final document in this set — Section 11 indexes every connection back to the EC2, Hypervisor, VPC, Storage, and Containers notes explicitly, closing the loop on this series' recurring themes: hardware/software TCB minimization, declarative reconciliation control loops, managed-convenience-vs-customer-control economics, and hard per-resource throughput ceilings.*
