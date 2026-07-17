# The Azure Kubernetes (AKS) Bible
*A decision-first reference for architecting, deploying, and operating serious workloads on AKS.*
*Last verified against Azure docs/release notes: July 2026.*

> **How to use this document:** Section 14 ("Pre-Flight Checklist for High-Intensity Workloads") is the one to open first if you're about to ship something complex to production. Everything before it is the reference material that checklist points back to.

---

## Table of Contents

1. [AKS vs. Everything Else](#1-aks-vs-everything-else)
2. [Cluster Foundations](#2-cluster-foundations)
3. [Networking](#3-networking)
4. [Compute & Autoscaling](#4-compute--autoscaling)
5. [AI/ML & GPU Workloads](#5-aiml--gpu-workloads)
6. [Storage](#6-storage)
7. [Identity & Security](#7-identity--security)
8. [Observability](#8-observability)
9. [CI/CD & GitOps](#9-cicd--gitops)
10. [Multi-Cluster, High Availability & Disaster Recovery](#10-multi-cluster-high-availability--disaster-recovery)
11. [Databases & Messaging: In-Cluster vs. Managed](#11-databases--messaging-in-cluster-vs-managed)
12. [Cost Optimization](#12-cost-optimization)
13. [Common Pitfalls & Anti-Patterns](#13-common-pitfalls--anti-patterns)
14. [Pre-Flight Checklist for High-Intensity Workloads](#14-pre-flight-checklist-for-high-intensity-workloads)
15. [Quick-Reference Decision Tables](#15-quick-reference-decision-tables)

---

## 1. AKS vs. Everything Else

Before committing to AKS, confirm it's actually the right layer. AKS wins when you need fine-grained control over networking, scheduling, multi-tenancy, service mesh, or a large heterogeneous set of microservices/batch/AI workloads on one substrate. It's the wrong tool if you just need to run a container.

| If you need... | Use instead of AKS |
|---|---|
| One or a few stateless containers, no orchestration control needed | **Azure Container Apps** (built on KEDA + Dapr + Envoy, serverless-ish) |
| A single container, simplest possible deploy | **Azure Container Instances (ACI)** |
| Classic web app + built-in CI/CD, no container expertise wanted | **Azure App Service** (with container support) |
| Event-driven, bursty microservices without wanting to manage a cluster | **Container Apps** |
| Full control of scheduling, custom CRDs/operators, service mesh, complex multi-tenant platform, GPU fleets, strict compliance/network isolation | **AKS** |
| You already run OSS Kubernetes and want lowest possible operational lift | **AKS Automatic** (see §2.1) |

**Rule of thumb:** if your team is asking about admission controllers, custom schedulers, service mesh, node pools, or GPU bin-packing — you're in AKS territory. If you're asking "how do I just run this image," you probably don't need Kubernetes at all.

---

## 2. Cluster Foundations

### 2.1 AKS Automatic vs. AKS Standard

Microsoft now ships two real "editions." This decision should be made first — it shapes almost everything downstream.

| | **AKS Automatic** (GA Oct 2025) | **AKS Standard** |
|---|---|---|
| Node provisioning | Node Autoprovisioning (Karpenter-based) always on, pre-configured | You choose: Cluster Autoscaler or opt into NAP |
| Networking defaults | Azure CNI Overlay + Cilium dataplane, hardened by default | Any supported CNI/dataplane combo |
| Security defaults | Image integrity, deployment safeguards, Workload Identity, private-by-default patterns pre-wired | You configure manually |
| Observability | Managed Prometheus + Grafana + Container Insights on by default | Opt-in, à la carte |
| Ingress | App Routing add-on pre-enabled | You choose |
| Control level | Opinionated — some knobs deliberately hidden | Full control over every primitive |
| SLA | Uptime SLA + pod-readiness SLA included by default | Uptime SLA only on Standard/Premium tier |
| Best for | Teams that want production-grade defaults fast, standard microservice/AI workloads | Unusual networking, deep customization, regulated environments needing explicit control, bare-metal/edge, legacy migrations |

**Guidance:** For most new, high-complexity production workloads, **start with AKS Automatic** unless you have a concrete, named reason you need Standard's control (e.g., custom CNI plugin, non-standard node OS, specific compliance tooling that assumes full control). Test your failure modes on Automatic before assuming you need Standard — the "opinionated" parts are usually opinions you'd have landed on anyway.

### 2.2 Node Pools

- **System node pool**: runs core add-ons (CoreDNS, metrics-server, konnectivity). Keep it small, dedicated, and untouched by application workloads (use taints). Minimum 3 nodes across zones for production.
- **User node pools**: your actual workloads. Segment by hardware profile (CPU-optimized, memory-optimized, GPU) and by workload class (latency-sensitive vs. batch), not by team/app name — that leads to node-pool sprawl.
- Use **taints + tolerations + node affinity** to keep GPU/expensive nodes exclusive to the workloads that need them.
- Prefer **NAP (node autoprovisioning)** over manually predefined node pools for variable workloads — it picks optimal VM SKUs per pending-pod shape and bin-packs better than static pools. Requires Azure CNI Overlay + Cilium dataplane.
- Cluster Autoscaler is still the right choice if you need tight, predictable control over exactly which SKUs can appear (e.g., hard compliance requirement to only ever provision from an approved SKU list) — NAP picks from a broader eligible set.

### 2.3 Kubernetes Version & Upgrade Strategy

- AKS supports the latest ~3 minor versions under standard support; **Long Term Support (LTS)** is available for an extra 1 year of coverage on a pinned minor version — use it for workloads that can't tolerate frequent forced upgrades, not as a substitute for having an upgrade process at all.
- Set an **auto-upgrade channel** (`node-image`, `patch`, `stable`, `rapid`) rather than upgrading manually and inconsistently — falling behind on AKS versions is one of the most common causes of painful, high-risk "big bang" upgrades later.
- Use **planned maintenance windows** so upgrades/reimages land outside business-critical hours.
- For node pool upgrades, use **Capacity-Based Surge** (MaxSurge + MaxUnavailable) to control blast radius during rolling upgrades — critical for high-traffic clusters where losing capacity mid-upgrade would page someone.

---

## 3. Networking

This is the area with the most "it depends" — and the one most likely to bite you late if under-planned. Also the area with the most churn recently, so pay attention to the ingress section specifically.

### 3.1 CNI Plugin Choice

| Option | When to use |
|---|---|
| **Azure CNI Overlay + Cilium dataplane** | **Default recommendation for new clusters.** Best IP address efficiency (no VNet IP exhaustion), best performance, native eBPF network policy, required for NAP. |
| **Azure CNI Overlay (non-Cilium)** | If you need overlay's IP efficiency but aren't ready for Cilium-specific features. |
| **Azure CNI (legacy, flat/VNet-routed)** | Only if you have a hard requirement for pods to get routable VNet IPs directly (e.g., certain legacy on-prem integration patterns). Burns IP address space fast at scale — plan subnets generously or avoid.
| **kubenet** | Legacy/deprecated path. Don't choose it for new clusters. |

### 3.2 Network Policy

- Use **Cilium's native network policy** (comes free with the Cilium dataplane) for L3/L4/L7 policy enforcement — it's faster and more capable than the Azure NPM or Calico options.
- If you're not on Cilium dataplane, Azure Network Policy Manager or Calico are the fallbacks.
- Treat network policy as **default-deny** at the namespace level for any multi-tenant or compliance-sensitive cluster; allow explicitly.

### 3.3 Ingress & Gateway — Read This Carefully (Time-Sensitive)

The ingress landscape on AKS changed materially in the last year and is mid-migration right now:

- The **upstream community Ingress-NGINX project was retired in March 2026.**
- AKS's **managed NGINX-based "application routing" add-on** is in a support wind-down: critical security patches only through **November 2026**, after which it receives no further Azure support.
- **You need a migration plan if you're currently on the NGINX app-routing add-on or standalone Ingress-NGINX.**

Your two supported forward paths:

| Path | What it is | Best for |
|---|---|---|
| **Application Gateway for Containers (AGC)** | Evolution of AGIC. Lives outside the cluster data plane, terminates public TLS, does L7 routing, integrates WAF, supports Gateway API v1. Can be composed with the Istio add-on for north-south → mesh routing. | Production workloads wanting a fully managed, Azure-native edge with WAF, that don't want ingress logic running as cluster pods. |
| **App Routing add-on — Gateway API mode** (`--enable-app-routing-istio`) | A *lightweight* Istio control plane managing Gateway API resources, without full service mesh (no sidecar injection, no mesh CRDs). Successor to the NGINX-based add-on. | Teams that want a managed, in-cluster Gateway API implementation without adopting a full mesh. |
| **Istio-based service mesh add-on (full)** | Full Istio control plane + sidecars, mTLS everywhere, traffic shaping, supports both Istio's native Gateway and Kubernetes Gateway API for ingress. | Workloads that need service-to-service mTLS, fine-grained traffic policy, circuit breaking, canary/traffic-splitting at the mesh level — i.e., genuinely complex microservice topologies. |

**Decision rule:**
- Need WAF + Azure-native edge + don't want ingress components living in-cluster → **AGC**.
- Need a simple, supported Gateway API ingress without adopting a mesh → **App Routing Gateway API mode**.
- Need service mesh capabilities (mTLS between services, retries/circuit-breaking, fine-grained L7 policy between your own microservices) → **Istio add-on**, optionally fronted by **AGC** for north-south.
- Still running the NGINX app-routing add-on or self-managed Ingress-NGINX → **start migrating now**; November 2026 is not far off for anything nontrivial.

Note: App Routing Gateway API mode and the full Istio service mesh add-on **cannot run simultaneously** — pick one.

### 3.4 Service Mesh

Only take on a service mesh if you have a concrete need: mTLS between services, fine-grained traffic splitting/canary at the network layer, or consistent retry/circuit-breaker policy across many services written in different languages. If your traffic patterns are simple, skip it — a mesh is real operational overhead (sidecar resource cost, upgrade cadence, added failure surface).

- **Istio add-on (Microsoft-supported)** is the default recommendation if you need a mesh on AKS — it's a tested distribution of upstream Istio with canary-based control-plane upgrades.
- Open Service Mesh is retired — do not build new designs on it.
- Linkerd remains a viable self-managed alternative if you want a lighter-weight mesh and are comfortable operating it yourself (not Microsoft-managed).

### 3.5 Cluster Access & Isolation

- **Private clusters** (API server has no public IP) are the default posture for anything handling sensitive data — pair with Azure Private Link / Private Endpoint for the API server.
- Use **authorized IP ranges** at minimum if you can't go fully private.
- Segment egress with **Azure Firewall** or NAT Gateway + explicit egress rules — don't leave clusters with unrestricted outbound.
- For **regulatory/data-sovereignty needs**, combine private clusters with **Azure Policy** to block resources outside approved regions.

---

## 4. Compute & Autoscaling

| Layer | Tool | Use for |
|---|---|---|
| Node count | **Node Autoprovisioning (NAP)** | Default for variable workloads; picks optimal VM SKU per pending pod, GA and Karpenter-based. |
| Node count | **Cluster Autoscaler** | When you need tight control over an explicit, fixed list of allowed VM SKUs (compliance-driven). |
| Pod replica count (CPU/memory) | **Horizontal Pod Autoscaler (HPA)** | Standard scale-out on resource metrics. |
| Pod replica count (custom/event metrics) | **KEDA** (managed add-on) | Scale on queue depth, Kafka lag, Service Bus messages, Prometheus metrics, cron schedules — essential for event-driven and bursty workloads. Also drives scale-to-zero. |
| Pod resource sizing | **Vertical Pod Autoscaler (VPA)**, add-on | Right-sizing requests/limits automatically; use in recommendation-only mode first before "Auto" mode in production, since VPA restarts pods to resize. |
| Cost-sensitive/interruptible workloads | **Spot node pools** | Batch jobs, CI runners, dev/test, stateless workloads tolerant of eviction. Never for stateful primary replicas without a resilience strategy. |
| Multi-tenant hard isolation | **Kata Containers / confidential VM node pools** | Untrusted or regulated multi-tenant workloads needing VM-level isolation between pods, not just namespace/cgroup isolation. |

**High-intensity workload note:** for latency-sensitive, high-throughput services, combine HPA/KEDA (reactive) with **pre-provisioned minimum replica counts and PodDisruptionBudgets** — pure autoscaling reacts *after* load arrives; it won't save you from a traffic spike faster than your scale-up time. Load-test your actual scale-up latency, don't assume it.

---

## 5. AI/ML & GPU Workloads

This is now a first-class AKS scenario, not a bolt-on, and the tooling has consolidated fast. If your "high-intensity" workload is AI inference/training, this section matters as much as networking.

| Need | Tool |
|---|---|
| Deploy/manage open-weight LLMs on your own GPU nodes with minimal ops | **KAITO** (Kubernetes AI Toolchain Operator, CNCF Sandbox, Microsoft-backed managed add-on). Auto-provisions right-sized GPU node pools per model, integrates **vLLM** as the inference engine. |
| Higher-level model selection/cost estimation/deployment workflow | **AI Runway** (Kubernetes-native, built on top of KAITO) |
| Distributed training / hyperparameter tuning at scale | **Ray**, now available as **Anyscale on Azure** (managed Ray, public preview as of mid-2026) |
| Standard model-serving abstraction across runtimes (vLLM, Triton, TF Serving) | **KServe** |
| GPU/batch job scheduling and queuing fairness across teams | **Kueue** |
| LLM-specific request routing (token-aware, model-aware load balancing) | Gateway API **Inference Extension** + an agent-aware gateway (e.g., agentgateway) — young/fast-moving, pin versions |

**GPU node pool practices:**
- Configure **ephemeral NVMe disks** on GPU VM families as scratch/cache space for model weights and datasets — don't rely on them for anything durable (data is lost on deallocation/redeploy).
- Use **Azure Container Storage** to auto-orchestrate ephemeral NVMe disks for workloads that need fast local storage without manual disk management.
- **Separate training, batch, and real-time inference workloads** onto different node pools/namespaces even within one cluster — resource contention between a training job and a latency-sensitive inference endpoint is a common self-inflicted incident.
- Keep GPU node OS images current — GPU driver updates ship through AKS node image releases and matter for both performance and CVEs.
- Set GPU node pool **minimum count to zero** and let NAP/KAITO provision on demand for cost control, unless you have a hard latency SLA that requires warm capacity.
- AMD GPU support (ROCm) is available alongside NVIDIA — confirm driver/tooling compatibility per workload before committing to a SKU family.

---

## 6. Storage

| Need | Use |
|---|---|
| Block storage for a single pod (databases, general PVCs) | **Azure Disk CSI driver** — Premium SSD v2 or Ultra Disk for high-IOPS/low-latency needs |
| Shared storage across many pods (ReadWriteMany) | **Azure Files CSI driver** (SMB/NFS) |
| Object storage mounted into pods | **Blob Storage CSI driver** |
| Extreme low-latency, high-throughput shared file storage (HPC, large ML training sets, EDA/rendering) | **Azure NetApp Files** |
| Local NVMe orchestration for GPU/cache workloads | **Azure Container Storage** |
| High-performance block storage decoupled from compute, SAN-like | **Elastic SAN** — good fit where you need zone-redundant storage independent of VM lifecycle |
| Fast ephemeral scratch space | Local ephemeral NVMe (no CSI, node-local, non-durable) |

**Rules:**
- Match storage class to actual IOPS/throughput needs — over-provisioning Premium SSD v2 or Ultra Disk everywhere is a common cost leak; under-provisioning for a database is a common performance incident.
- For StatefulSets with real durability requirements, always confirm your **zone-redundant storage** story — a zonal outage shouldn't silently orphan your data.
- Back up PV data explicitly (see §10) — CSI snapshots are not a substitute for an actual backup/retention strategy.

---

## 7. Identity & Security

| Concern | Tool |
|---|---|
| Pods authenticating to Azure resources (Key Vault, Storage, DB) | **Microsoft Entra Workload Identity** (federated OIDC, the only supported path now — AAD Pod Identity is fully retired, don't build on it) |
| Cluster access control | **Azure RBAC for Kubernetes Authorization** integrated with Entra ID groups — prefer this over managing raw Kubernetes RBAC manifests for human access; keep native K8s RBAC for workload-to-workload/service-account scoping |
| Policy enforcement (org guardrails) | **Azure Policy for AKS** (built on Gatekeeper/OPA) — enforce things like "no privileged containers," "images must come from approved registry," required labels |
| Runtime threat detection | **Microsoft Defender for Containers** — vulnerability scanning of images in ACR, runtime threat detection, attack-path analysis |
| Secrets | **Azure Key Vault Provider for Secrets Store CSI Driver** — mount secrets as volumes, avoid storing raw secrets in Kubernetes Secrets objects where possible; enable auto-rotation |
| Pod-level security posture | **Pod Security Admission** (baseline/restricted profiles) — replaces deprecated PodSecurityPolicy |
| Image supply chain | **ACR with content trust/signing**, vulnerability scanning gate in CI before deploy, restrict clusters to pull only from approved registries via Azure Policy |
| Network-level isolation | Private clusters, NSGs, Azure Firewall, default-deny network policy (§3.2) |
| Hard multi-tenant isolation | Kata/confidential VM node pools (§4) — namespace isolation alone is not a security boundary against a genuinely untrusted tenant |
| Compliance/regulated workloads | Confidential Computing node pools (confidential VMs with Azure Linux) where data-in-use protection matters; FIPS-validated node pools where mandated |

**High-complexity note:** for anything genuinely high-stakes, don't treat these as a checklist to tick once — wire Defender findings and Azure Policy compliance into your actual deployment pipeline so violations block promotion rather than get discovered after the fact.

---

## 8. Observability

| Need | Tool |
|---|---|
| Cluster/pod-level metrics, dashboards | **Azure Monitor managed Prometheus + Grafana** (includes built-in GPU metrics by default now) |
| Logs, container insights | **Container Insights** (Azure Monitor) |
| Application-level tracing/APM | **Application Insights** or self-hosted **OpenTelemetry Collector** exporting to your backend of choice |
| Cost visibility per namespace/team | Azure Cost Management + Kubernetes cost allocation, or **Kubecost** for finer-grained show-back/charge-back |
| GPU utilization | Managed Prometheus GPU metrics, or self-managed **NVIDIA DCGM exporter** for deeper detail |

**Guidance:** wire alerting to business-relevant SLOs (error budget burn, p99 latency, queue depth) rather than just infra signals (CPU%) — for high-intensity apps, infra can look "healthy" while the app is failing user requests (e.g., thread pool exhaustion, downstream timeouts).

---

## 9. CI/CD & GitOps

| Need | Tool |
|---|---|
| Declarative, pull-based continuous deployment from Git | **Flux** (Microsoft-supported AKS extension) — recommended default for GitOps on AKS |
| Alternative GitOps controller with a stronger UI/multi-cluster app-of-apps pattern | **ArgoCD** (self-managed, well supported on AKS, common in larger platform teams) |
| CI pipelines | **GitHub Actions** or **Azure DevOps Pipelines** — build, test, scan, push to ACR |
| Scaffolding new services with Kubernetes manifests/Helm/Dockerfiles quickly | **Draft** |
| Progressive delivery (canary, blue/green) | Flagger (works with Istio/Gateway API) or native Argo Rollouts if on ArgoCD |

**Rule:** for high-complexity, multi-service systems, GitOps (pull-based) beats push-based `kubectl apply` pipelines — it gives you an auditable, reconciled source of truth and makes drift detection/rollback trivial. Treat this as close to mandatory once you're past a handful of services.

---

## 10. Multi-Cluster, High Availability & Disaster Recovery

### 10.1 Within a region
- Spread nodes across **availability zones** — AKS Standard/Premium tier with zones gives 99.95% API server SLA.
- Use **PodDisruptionBudgets** and **topology spread constraints** so a zone or node-pool maintenance event doesn't take out all replicas of a critical service at once.

### 10.2 Across regions

| Pattern | Description | Best for |
|---|---|---|
| **Active-passive** | Secondary cluster mirrors primary config/data but doesn't serve traffic until failover via Azure Front Door | Workloads tied to a primary-region database/state store |
| **Active-active** | Two+ regions actively serve traffic simultaneously | Stateless services, or services fronting globally-distributed data (Cosmos DB multi-region writes, etc.) |
| **Deployment stamps / geodes** | Independent regional stamps, any region can serve any request | Large-scale SaaS with strict latency/data-locality requirements |

- **Azure Kubernetes Fleet Manager** coordinates version/upgrade rollout across clusters and now supports **cross-cluster networking via managed Cilium cluster mesh** — global service discovery and routing across member clusters without hand-rolled VPNs. Use it once you're running more than a couple of clusters, or need coordinated multi-region failover.
- Route global traffic with **Azure Front Door** (L7, WAF, health-probe-driven failover) or **Azure Traffic Manager** (DNS-based) depending on whether you need L7 features.
- Replicate container images with **ACR geo-replication** so a regional Azure outage doesn't also take out your image pulls.

### 10.3 Backup
- **Azure Backup for AKS** (native, GA) — cluster state + persistent volume backup with scheduled policies, restore to same or different cluster.
- **Velero** remains a valid self-managed alternative, especially for teams already standardized on it across non-Azure clusters too.
- Back up etcd/cluster state *and* PV data *and* your GitOps repo (the last one you probably already have, but confirm it's actually the full source of truth, not partially hand-patched in-cluster).

---

## 11. Databases & Messaging: In-Cluster vs. Managed

**Default answer: use Azure's managed PaaS data services, not self-hosted-in-cluster, unless you have a specific reason not to.** Running a real production database inside Kubernetes (via an operator) adds operational burden that's rarely worth it unless you need portability across clouds or already have deep in-house Kubernetes-database-operator expertise.

| Need | Recommended | When to consider in-cluster instead |
|---|---|---|
| Relational DB | **Azure Database for PostgreSQL/MySQL Flexible Server**, Azure SQL | Rarely — only for cloud-portability requirements or very specific extension needs unavailable in managed offering |
| Globally distributed, multi-model | **Cosmos DB** | Not applicable — no good in-cluster equivalent |
| Caching | **Azure Cache for Redis** | Simple, low-stakes caching where an in-cluster Redis is genuinely disposable |
| Message queue / pub-sub | **Azure Service Bus** (queues/topics), **Event Hubs** (high-throughput streaming, Kafka-compatible endpoint) | Self-hosted Kafka/RabbitMQ on AKS if you need capabilities Event Hubs' Kafka surface doesn't cover, or multi-cloud portability |
| Event-driven scaling trigger | KEDA scalers exist for both Service Bus and Event Hubs — pair scaling with whichever messaging backend you pick |

---

## 12. Cost Optimization

- **NAP/Karpenter-based bin-packing** beats static over-provisioned node pools by default — let it consolidate.
- **Spot node pools** for anything interruption-tolerant (CI, batch, dev/test, some ML training with checkpointing).
- **GPU nodes at min=0**, scale on demand via KAITO/NAP — idle GPU nodes are the single most common AKS cost leak in AI workloads.
- **Reserved Instances / Azure Savings Plans** for your steady-state baseline capacity once you understand your floor usage — don't reserve before you know your baseline.
- **VPA in recommendation mode** to catch over-provisioned requests/limits across the fleet.
- **Kubecost or Azure Cost Management** namespace/label-based allocation so cost ownership is visible to the teams generating it — visibility alone typically drives real optimization.
- **ACR geo-replication and image size**: smaller images pull faster and cost less to replicate/store — don't ignore image hygiene at scale.

---

## 13. Common Pitfalls & Anti-Patterns

- **Skipping a real ingress/gateway decision** and defaulting to whatever's already deprecated (see §3.3) — you will be forced to migrate later, under more pressure.
- **No PodDisruptionBudgets** — a routine node upgrade or zone rebalance takes down all replicas of a "highly available" service at once.
- **Autoscaling as your only capacity strategy** for latency-sensitive services — scale-up isn't instant; a spike can outrun HPA/NAP.
- **Running stateful, non-replicated primaries on Spot nodes.**
- **Treating namespaces as a security boundary** for genuinely untrusted multi-tenant workloads — they're not; use confidential/Kata isolation when it matters.
- **Manual `kubectl apply` pipelines** for complex multi-service systems instead of GitOps — leads to drift no one can explain during an incident.
- **No default-deny network policy** — flat networking inside the cluster means one compromised pod can reach everything.
- **Ignoring Kubernetes version upgrades** until forced — batching years of skipped upgrades into one high-risk jump.
- **Over-provisioning GPU nodes with min > 0** "just in case" — the most common AI-workload cost leak.
- **No tested DR/backup restore path** — a backup you've never restored from is a hypothesis, not a plan.
- **Mixing training and real-time inference on the same GPU node pool** without isolation — contention shows up as production latency incidents.
- **Secrets stored as plain Kubernetes Secrets** instead of Key Vault CSI-mounted — base64 is not encryption.

---

## 14. Pre-Flight Checklist for High-Intensity Workloads

Before you move a genuinely complex, high-intensity application to production on AKS, walk through this in order:

**Architecture**
- [ ] Chose AKS Automatic vs. Standard deliberately, not by default (§2.1)
- [ ] CNI is Azure CNI Overlay + Cilium unless there's a documented reason otherwise (§3.1)
- [ ] Ingress/gateway path decided with the November 2026 NGINX deprecation in mind (§3.3)
- [ ] Service mesh adopted only if there's a concrete mTLS/traffic-policy need (§3.4)
- [ ] Private cluster + network segmentation matches your data sensitivity (§3.5)

**Scaling & Resilience**
- [ ] Load-tested actual scale-up latency under realistic traffic spikes, not assumed it
- [ ] HPA/KEDA configured with sane min replicas — not scaling from zero for anything latency-critical
- [ ] PodDisruptionBudgets and topology spread constraints set on every critical Deployment/StatefulSet
- [ ] Multi-zone at minimum; multi-region if your SLA requires surviving a regional outage
- [ ] Chosen and tested a DR pattern (active-active / active-passive / stamps) — and actually tested failover, not just designed it

**Data**
- [ ] Storage class matched to real IOPS/throughput needs, not defaulted
- [ ] Zone-redundant storage for anything with durability requirements
- [ ] Backup strategy covers cluster state + PVs + GitOps repo, and has been **restored from at least once**
- [ ] Managed data services used by default; in-cluster databases justified explicitly if chosen

**Security**
- [ ] Workload Identity in place, no long-lived credentials in pods
- [ ] Default-deny network policy at namespace level
- [ ] Azure Policy guardrails enforced in pipeline (not just documented)
- [ ] Defender for Containers enabled and findings wired into CI/CD gating
- [ ] Secrets via Key Vault CSI, not raw Kubernetes Secrets
- [ ] Multi-tenant isolation model matches actual trust boundaries (namespace vs. Kata/confidential)

**Compute (if AI/GPU workload)**
- [ ] Training/inference/batch workloads isolated onto separate node pools
- [ ] GPU node pools min=0 with NAP/KAITO-driven scale, unless a hard latency SLA says otherwise
- [ ] Ephemeral NVMe usage understood as non-durable

**Operations**
- [ ] GitOps (Flux/ArgoCD) is the actual deployment path, no hand-applied manifests
- [ ] Auto-upgrade channel configured with a maintenance window, not manual/ad hoc upgrades
- [ ] Observability wired to business SLOs (latency, error budget), not just infra metrics
- [ ] Cost allocation visible per team/namespace before scale makes it painful to retrofit
- [ ] On-call runbooks exist for: node pool exhaustion, zone failure, failed upgrade rollback, storage exhaustion

If you can't check most of these boxes, that's not necessarily a blocker — but each unchecked box is a named risk you're choosing to accept, and it should be a conscious choice, not an oversight.

---

## 15. Quick-Reference Decision Tables

**"Which ingress/gateway do I use?"**
- Need WAF + Azure-native edge → AGC
- Need simple Gateway API, no mesh → App Routing Gateway API mode
- Need mTLS + traffic policy between your services → Istio add-on

**"Which autoscaler?"**
- Nodes, general case → NAP
- Nodes, fixed SKU allowlist required → Cluster Autoscaler
- Pods, CPU/memory → HPA
- Pods, queue/event-driven → KEDA
- Right-sizing requests/limits → VPA (recommendation mode first)

**"Where does my data live?"**
- Relational/general → Managed PaaS (Postgres/MySQL Flexible Server, Azure SQL)
- Global scale/multi-model → Cosmos DB
- Block storage → Disk CSI (Premium SSD v2/Ultra for high IOPS)
- Shared file access → Files CSI or NetApp Files (extreme performance)
- Object → Blob CSI

**"How do I run AI workloads?"**
- Serve an open-weight LLM → KAITO + vLLM
- Higher-level model ops workflow → AI Runway
- Distributed training → Ray / Anyscale on Azure
- Multi-runtime model serving abstraction → KServe
- GPU job queuing/fairness → Kueue

**"Do I need a service mesh?"**
- Only if you need mTLS between services, fine-grained traffic splitting, or consistent retry/circuit-break policy across many polyglot services. Otherwise, skip it.

---

*This document reflects the AKS platform as of July 2026. Azure ships AKS updates roughly bi-weekly (see the AKS GitHub release notes) — re-verify anything with a hard external deadline (like the NGINX ingress retirement) closer to your deployment date.*
