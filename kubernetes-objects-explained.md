# Kubernetes Objects Explained

*A practical reference to every major Kubernetes object: what it is, when to use it, and how it relates to the others.*

---

## Table of Contents

1. [The Mental Model](#1-the-mental-model)
2. [Workload Objects](#2-workload-objects)
3. [Networking Objects](#3-networking-objects)
4. [Configuration & Storage](#4-configuration--storage)
5. [Scaling & Availability](#5-scaling--availability)
6. [Cluster, Identity & Access Control](#6-cluster-identity--access-control)
7. [Extensibility](#7-extensibility)
8. [Quick Decision Table](#8-quick-decision-table)

---

## 1. The Mental Model

Kubernetes objects fall into six functional groups. Almost everything you'll ever touch is one of these:

| Group | Answers the question | Examples |
|---|---|---|
| **Workload** | "What runs?" | Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob |
| **Networking** | "How does traffic reach it?" | Service, Ingress, Gateway/HTTPRoute, NetworkPolicy |
| **Config & Storage** | "What data/config does it need?" | ConfigMap, Secret, PersistentVolume, PersistentVolumeClaim, StorageClass |
| **Scaling & Availability** | "How does it grow/shrink and stay up?" | HorizontalPodAutoscaler, VerticalPodAutoscaler, PodDisruptionBudget |
| **Cluster & Access** | "Who can do what, where?" | Namespace, ServiceAccount, Role/ClusterRole, RoleBinding/ClusterRoleBinding, ResourceQuota |
| **Extensibility** | "How do I teach Kubernetes new object types?" | CustomResourceDefinition, Operators, Admission Webhooks |

A useful rule as you read: **you almost never create a bare Pod directly in production** — you create a higher-level controller (Deployment, StatefulSet, Job, etc.) that creates and manages Pods for you.

---

## 2. Workload Objects

### Pod
The smallest deployable unit — one or more containers that share network namespace and storage, always scheduled together on the same node.

- **Use when:** you're learning, debugging, or running a genuine one-off (rare in production).
- **Don't use directly for:** anything that needs to survive a node failure or restart automatically — a bare Pod that dies stays dead.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
```

### ReplicaSet
Ensures a specified number of identical Pod replicas are running at all times. Replaces Pods that die.

- **Use when:** almost never directly — it's the mechanism *underneath* a Deployment. You interact with Deployments, and Deployments manage ReplicaSets for you.

### Deployment
Manages ReplicaSets on your behalf and adds **rolling updates, rollbacks, and declarative versioning** on top.

- **Use for:** stateless applications — web servers, APIs, workers — where any replica is interchangeable and none holds unique identity or storage.
- **Key feature:** change the Pod template (e.g., new image tag) and the Deployment controller rolls out the change gradually, replacing old Pods with new ones, and can roll back if something goes wrong.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
      - name: web
        image: myapp:2.0
```

### StatefulSet
Like a Deployment, but for workloads where **each replica needs a stable, unique identity** — a predictable network name (`pod-0`, `pod-1`, ...) and its own persistent storage that follows it across rescheduling.

- **Use for:** databases, message queues, anything clustered (Kafka, Cassandra, Elasticsearch, ZooKeeper) — anything where "which specific instance is this" matters, and where replica N should always reattach to the storage volume it had before.
- **Key differences from Deployment:** ordered, predictable Pod creation/deletion (0, 1, 2... and reverse on scale-down); stable network identity via a headless Service; per-Pod PersistentVolumeClaims that persist even if the Pod is rescheduled.

### DaemonSet
Ensures **exactly one copy of a Pod runs on every node** (or a selected subset of nodes) in the cluster.

- **Use for:** node-level infrastructure — log collectors (Fluent Bit), monitoring agents (Prometheus Node Exporter, Datadog agent), CNI plugins, storage drivers (CSI node plugins), security agents.
- **Rule of thumb:** if the question is "does every node need one of these running on it," it's a DaemonSet, not a Deployment.

### Job
Runs Pods to completion — a specified number of Pods must **finish successfully** rather than run forever. Retries on failure up to a configured limit.

- **Use for:** one-off or finite tasks — database migrations, batch data processing, report generation, a task that should run once and exit.
- **Parallelism** can be configured to run multiple Pods concurrently to process a workload faster.

### CronJob
A Job that runs on a **recurring schedule**, using standard cron syntax.

- **Use for:** scheduled maintenance tasks, nightly backups, periodic report generation, cleanup jobs.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
          restartPolicy: OnFailure
```

---

## 3. Networking Objects

### Service
A stable network identity (virtual IP + DNS name) that load-balances traffic across a dynamic set of Pods, selected by label. Solves the problem that Pods are ephemeral and their IPs change constantly.

| Service type | What it does | Use when |
|---|---|---|
| **ClusterIP** (default) | Internal-only virtual IP, reachable from within the cluster | Backend services, internal APIs — the vast majority of Services |
| **NodePort** | Exposes the Service on a static port on every node's IP | Simple external access without a cloud load balancer, dev/test, on-prem edge cases |
| **LoadBalancer** | Provisions a cloud provider load balancer pointing at the Service | Directly exposing a service to the internet without a separate Ingress layer |
| **ExternalName** | Maps the Service to an external DNS name (no proxying) | Referencing an external system (e.g., an external database) using in-cluster service discovery conventions |
| **Headless** (`clusterIP: None`) | No virtual IP; DNS returns individual Pod IPs directly | StatefulSets, or any case where clients need to address individual Pods rather than a load-balanced set |

### Ingress
Routes **external HTTP/HTTPS traffic** into the cluster based on hostname/path, to different Services, typically with TLS termination — all through a single external entry point instead of one LoadBalancer per Service.

- **Requires an Ingress Controller** (e.g., NGINX, cloud-managed controllers) actually running in the cluster to do anything — the Ingress object alone is just a routing spec.
- **Use for:** classic HTTP(S) routing — host-based and path-based rules to multiple backend Services.
- **Being succeeded by:** the **Gateway API** (see below) — new designs increasingly favor Gateway API for its more expressive, role-oriented model.

### Gateway API (Gateway, HTTPRoute, GRPCRoute, TLSRoute, etc.)
The modern, more expressive successor to Ingress. Separates concerns between infrastructure operators (who define a `Gateway` — the actual listener/load balancer) and application teams (who define `HTTPRoute`/similar objects attaching to it).

- **Use for:** anything Ingress used to handle, plus more advanced routing (traffic splitting/weighting, header-based routing, gRPC routing) and cleaner multi-team ownership boundaries.
- **Use when starting fresh** — it's the direction the ecosystem is moving, with broader protocol support than Ingress ever had.

### NetworkPolicy
Firewall rules **for pod-to-pod traffic** inside the cluster — controls which Pods can talk to which, on which ports, by label selector.

- **Use for:** enforcing least-privilege network access between workloads (e.g., "only the `frontend` label can talk to `backend` on port 8080; deny everything else"). Requires a CNI plugin that supports it (not all do out of the box).
- **Default behavior without any NetworkPolicy:** all Pods can talk to all Pods — flat, open networking. Add default-deny policies for anything sensitive.

### Endpoints / EndpointSlice
The actual list of Pod IPs backing a Service — mostly **managed automatically for you**, rarely created by hand. EndpointSlice is the newer, more scalable version (Endpoints doesn't scale well to Services with thousands of backing Pods).

---

## 4. Configuration & Storage

### ConfigMap
Stores **non-sensitive configuration data** as key-value pairs, injectable into Pods as environment variables, command-line args, or mounted files.

- **Use for:** application config, feature flags, config files — anything you'd want to change without rebuilding your container image.

### Secret
Like a ConfigMap, but intended for **sensitive data** (passwords, tokens, keys). Base64-encoded, not encrypted, by default — that's *encoding*, not security.

- **Use for:** credentials, API keys, TLS certificates.
- **Important:** base64 is trivially reversible. For real security, enable encryption at rest on your cluster's etcd, and consider an external secrets manager (e.g., a cloud KMS/Key Vault integration) rather than relying on raw Kubernetes Secrets alone for sensitive production data.

### PersistentVolume (PV)
A piece of actual storage in the cluster (a disk, NFS share, cloud volume) — provisioned either manually by an admin or dynamically by a StorageClass. Exists independently of any Pod's lifecycle.

### PersistentVolumeClaim (PVC)
A **request** for storage made by a Pod/StatefulSet — "I need 10Gi, ReadWriteOnce, fast SSD." Kubernetes binds it to a matching PersistentVolume.

- **Relationship:** Pods reference PVCs, PVCs bind to PVs. This indirection lets application manifests stay portable — they ask for storage by characteristics, not by a specific physical volume.

### StorageClass
Defines **how** dynamic PersistentVolumes get provisioned — which storage backend, what performance tier, what reclaim policy.

- **Use for:** letting PVCs auto-provision matching PVs on demand instead of requiring an admin to pre-create every volume by hand. Almost always used in modern clusters — static PV provisioning is the exception now, not the rule.

### VolumeSnapshot
A point-in-time snapshot of a PersistentVolume's data, used for backup or cloning a volume into a new PVC.

---

## 5. Scaling & Availability

### HorizontalPodAutoscaler (HPA)
Automatically adjusts the **number of Pod replicas** on a Deployment/StatefulSet based on observed metrics (CPU, memory, or custom/external metrics).

- **Use for:** scaling stateless workloads in/out with load.

### VerticalPodAutoscaler (VPA)
Automatically adjusts a Pod's **CPU/memory requests and limits** based on observed usage, rather than replica count.

- **Use for:** right-sizing workloads that are hard to scale horizontally, or catching chronically over/under-provisioned resource requests.
- **Caution:** in "Auto" mode it restarts Pods to resize them — start in recommendation-only mode in production.

### PodDisruptionBudget (PDB)
Limits how many Pods of a given application can be voluntarily taken down at once (during node drains, upgrades, cluster scaling) — protects availability during planned disruptions.

- **Use for:** any workload where losing too many replicas simultaneously during routine maintenance would cause an outage. Set `minAvailable` or `maxUnavailable` on anything you actually care about staying up.

---

## 6. Cluster, Identity & Access Control

### Namespace
A logical partition within a cluster — scopes names, resource quotas, and access control. Doesn't provide network or security isolation by itself (that needs NetworkPolicy + RBAC on top).

- **Use for:** separating environments (dev/staging/prod on one cluster), separating teams/products, applying different quotas or policies to different groups of workloads.

### ServiceAccount
An identity that **Pods use** to authenticate to the Kubernetes API (and, via federation, often to cloud provider APIs too — e.g., workload identity patterns).

- **Use for:** giving a workload only the permissions it actually needs, instead of using broad, shared credentials. Every Pod runs as some ServiceAccount, `default` if none is specified — don't leave workloads that touch the API on `default` with broad permissions.

### Role / ClusterRole
Defines a **set of permissions** — which verbs (get, list, create, delete...) are allowed on which resource types.

- **Role:** scoped to a single Namespace.
- **ClusterRole:** cluster-wide, or reusable across namespaces (e.g., "view" permissions granted per-namespace via multiple bindings).

### RoleBinding / ClusterRoleBinding
**Grants** a Role or ClusterRole to a specific user, group, or ServiceAccount.

- **RoleBinding:** grants permissions within one namespace (can reference either a Role or a ClusterRole, scoped to that namespace).
- **ClusterRoleBinding:** grants permissions cluster-wide.

**How these four fit together:** a Role/ClusterRole defines *what's allowed*; a RoleBinding/ClusterRoleBinding defines *who gets it and where*. This is standard Kubernetes RBAC — separate from any cloud provider's own IAM/RBAC layer, which typically governs access to the cluster's control plane itself rather than in-cluster resources.

### ResourceQuota
Caps the **total** amount of resources (CPU, memory, object counts like number of Pods/Services) a Namespace can consume.

- **Use for:** preventing one team/namespace from starving others of cluster capacity in a shared/multi-tenant cluster.

### LimitRange
Sets **default and min/max** resource requests/limits for individual Pods/containers within a Namespace, so workloads that don't specify their own limits still get sane defaults.

---

## 7. Extensibility

### CustomResourceDefinition (CRD)
Lets you define **entirely new object types** that behave like native Kubernetes resources — `kubectl get`, `kubectl apply`, and the API all just work on them.

- **Use for:** representing domain-specific concepts declaratively (e.g., a `Certificate` object for cert-manager, a `VirtualService` for Istio, a `Workspace` for an AI model-serving operator).
- CRDs by themselves are just data — they need a **controller** watching them to actually *do* anything.

### Operator (pattern, not a single object)
A controller (usually built on a CRD) that encodes **operational knowledge** for running a specific piece of software — e.g., an operator for a database that handles backups, failover, and version upgrades automatically by watching a custom resource and reconciling real-world state to match it.

- **Use for:** running complex stateful software (databases, message brokers) on Kubernetes with automated day-2 operations, if you've decided to run it in-cluster rather than use a managed cloud equivalent.

### Admission Webhooks (ValidatingWebhookConfiguration / MutatingWebhookConfiguration)
Intercept API requests **before** an object is persisted, to validate it against policy or mutate it automatically (e.g., inject a sidecar container, enforce required labels, block privileged Pods).

- **Use for:** enforcing organizational policy (often via tools like Gatekeeper/OPA or Kyverno, which are themselves built on this mechanism), and for automatic injection patterns (service mesh sidecars, Key Vault secret injection).

---

## 8. Quick Decision Table

| "I need to..." | Use |
|---|---|
| Run a stateless app that should self-heal and support rolling updates | **Deployment** |
| Run a database or clustered app needing stable identity + storage per replica | **StatefulSet** |
| Run something on every node (logging, monitoring agent) | **DaemonSet** |
| Run a task once to completion | **Job** |
| Run a task on a schedule | **CronJob** |
| Give a set of Pods a stable internal address | **Service (ClusterIP)** |
| Expose something directly to the internet without a routing layer | **Service (LoadBalancer)** |
| Route HTTP(S) by hostname/path to multiple backends, with TLS | **Ingress** or **Gateway API** |
| Control which Pods can talk to which | **NetworkPolicy** |
| Inject config that isn't sensitive | **ConfigMap** |
| Inject credentials/keys | **Secret** (+ external secrets manager for real security) |
| Request durable storage for a Pod | **PersistentVolumeClaim** (backed by a **StorageClass**) |
| Auto-scale replica count with load | **HorizontalPodAutoscaler** |
| Auto-right-size CPU/memory requests | **VerticalPodAutoscaler** |
| Protect availability during node drains/upgrades | **PodDisruptionBudget** |
| Separate environments/teams on one cluster | **Namespace** |
| Give a workload scoped API/cloud permissions | **ServiceAccount** (+ Role/RoleBinding) |
| Cap resource usage per team/namespace | **ResourceQuota** |
| Teach Kubernetes a new kind of object | **CustomResourceDefinition** + a controller/Operator |
| Enforce policy or auto-inject things at object-creation time | **Admission Webhook** |

---

*This is a conceptual reference, not exhaustive API documentation — field-level specifics change between Kubernetes versions, so check `kubectl explain <object>` or the official API reference for exact schema on the version you're running.*
