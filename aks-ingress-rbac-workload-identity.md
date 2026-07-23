# Ingress, IAM/RBAC, and OIDC Workload Identity in AKS

A practical guide to three AKS building blocks — how traffic gets in, who can do what, and how pods get Azure credentials safely — and how they combine in one real deployment.

---

## 1. Ingress — getting traffic into the cluster

Ingress is a Kubernetes object that routes external HTTP(S) traffic to internal Services, based on hostname/path. It needs an **Ingress Controller** running in the cluster to actually do the routing (e.g. NGINX Ingress Controller, or Azure's Application Gateway Ingress Controller — AGIC).

```
Internet → Load Balancer → Ingress Controller (pod) → Service → Pod(s)
```

**Example Ingress resource:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

Ingress only handles **routing**. It doesn't know or care who's allowed to deploy it, or what credentials the app behind it uses — that's RBAC and Workload Identity's job.

---

## 2. IAM & RBAC — two separate layers

This is the part people mix up most: **Azure IAM** and **Kubernetes RBAC** are different systems controlling different things.

| | Azure IAM (RBAC) | Kubernetes RBAC |
|---|---|---|
| Controls | Access to the **AKS resource itself** (Azure control plane) | Access to **objects inside the cluster** (pods, secrets, deployments) |
| Example question | "Can this user start/stop/delete the AKS cluster?" | "Can this user create a pod in namespace `prod`?" |
| Assigned via | Azure role assignments (e.g. `Azure Kubernetes Service Cluster Admin Role`) | `Role`/`ClusterRole` + `RoleBinding`/`ClusterRoleBinding` |
| Scope | Subscription / resource group / cluster resource | Namespace or whole cluster |

**Azure IAM example** — grant a user permission to get cluster credentials:

```bash
az role assignment create \
  --assignee user@example.com \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<cluster-name>
```

**Kubernetes RBAC example** — inside the cluster, let a group only view pods in `prod`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: prod
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

AKS can even bridge the two using **Azure RBAC for Kubernetes authorization**, where Kubernetes permissions are managed as Azure role assignments instead of Role/RoleBinding YAML — but the two layers (resource access vs. in-cluster access) remain conceptually distinct.

---

## 3. OIDC Issuer & Workload Identity — pods getting Azure credentials

The problem: a pod often needs to call Azure services (Key Vault, Storage, SQL). The old way was storing a Service Principal secret in a Kubernetes Secret — a long-lived credential that could leak.

**Workload Identity Federation** solves this with no stored secrets:

1. AKS exposes an **OIDC issuer** (a public endpoint that can issue verifiable tokens for Kubernetes ServiceAccounts).
2. You create a **federated identity credential** on an Azure Managed Identity, trusting tokens from that OIDC issuer for a specific `namespace:serviceaccount`.
3. A pod using that ServiceAccount gets a short-lived Kubernetes-issued token automatically projected into it.
4. The Azure SDK inside the pod exchanges that token for a real Azure AD access token — via Azure AD trusting the AKS OIDC issuer, no secret ever stored.

```
Pod (ServiceAccount token) → Azure AD (validates via AKS OIDC issuer) → Azure AD access token → Key Vault/Storage/etc.
```

**Setup:**

```bash
# 1. Enable OIDC issuer + workload identity on the cluster
az aks update -g myRG -n myCluster \
  --enable-oidc-issuer --enable-workload-identity

# 2. Get the issuer URL
export AKS_OIDC_ISSUER=$(az aks show -g myRG -n myCluster \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# 3. Create a managed identity
az identity create -g myRG -n myAppIdentity

# 4. Federate it with the ServiceAccount
az identity federated-credential create \
  --name myAppFedCred \
  --identity-name myAppIdentity \
  --resource-group myRG \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:prod:myapp-sa \
  --audience api://AzureADTokenExchange
```

**ServiceAccount (annotated with the identity's client ID):**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: prod
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
```

**Pod using it:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels:
        app: myapp
        azure.workload.identity/use: "true"   # required to inject the token
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myacr.azurecr.io/myapp:v1
          ports:
            - containerPort: 8080
```

No secret, anywhere — the pod authenticates as itself.

---

## 4. How they combine — one deployment, one picture

```
                         ┌─────────────────────────────────────────┐
   Internet              │                AKS Cluster               │
      │                  │                                           │
      ▼                  │   ┌───────────────┐      ┌─────────────┐  │
  Ingress ───────────────┼──▶│ Ingress        │─────▶│  myapp      │  │
  (nginx)                │   │ Controller     │      │  Pod        │  │
                         │   └───────────────┘      │  (ServiceAcct)│ │
                         │                            └──────┬──────┘  │
                         └───────────────────────────────────┼─────────┘
                                                               │ workload identity token
                                                               ▼
                                                    Azure AD (validates via OIDC issuer)
                                                               │
                                                               ▼
                                                        Azure Key Vault
```

- **Ingress** decides *how traffic reaches* `myapp`.
- **Kubernetes RBAC** decides *which humans/CI pipelines* are allowed to create/edit that Ingress, Deployment, or ServiceAccount in the `prod` namespace.
- **Azure IAM** decides *who can touch the AKS cluster resource itself* (e.g. who can even run `az aks get-credentials`).
- **OIDC Workload Identity** decides *how the running pod itself* authenticates to Azure — no human, no static secret.

---

## 5. Simple end-to-end example

**Scenario:** `myapp` is exposed publicly via Ingress, reads a database connection string from Azure Key Vault using Workload Identity, and only the `prod-deployers` group is allowed to modify it in-cluster.

```yaml
# 1. Namespace-scoped RBAC: only prod-deployers can touch Deployments/Ingress in `prod`
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: deployer
rules:
  - apiGroups: ["apps", "networking.k8s.io"]
    resources: ["deployments", "ingresses"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prod-deployers-binding
  namespace: prod
subjects:
  - kind: Group
    name: prod-deployers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
---
# 2. ServiceAccount federated to an Azure Managed Identity (workload identity)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: prod
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
---
# 3. The app pod — pulls DB secret from Key Vault via Azure AD, no stored secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp, azure.workload.identity/use: "true" }
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myacr.azurecr.io/myapp:v1
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: prod
spec:
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 8080 }]
---
# 4. Ingress exposing it publicly
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port: { number: 80 }
```

On top of this, Azure IAM (outside the cluster) controls who can run `az aks get-credentials` to even fetch a kubeconfig for this cluster in the first place — the outermost gate before Kubernetes RBAC ever applies.

---

## 6. Quick summary

| Concept | Question it answers | Where it lives |
|---|---|---|
| **Ingress** | How does traffic reach my service? | Inside the cluster (routing) |
| **Kubernetes RBAC** | What can this user/service account do *in* the cluster? | Inside the cluster (API server) |
| **Azure IAM** | Who can manage the AKS *resource* itself? | Azure control plane |
| **OIDC / Workload Identity** | How does a pod prove its identity to Azure, without a secret? | Bridges cluster ↔ Azure AD |

Together, they give you: controlled entry (Ingress), controlled cluster changes (RBAC + IAM), and secretless, auditable access to Azure services from your workloads (OIDC Workload Identity).
