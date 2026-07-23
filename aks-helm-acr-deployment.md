# Deploying a Customer Container to AKS via Helm + Argo CD, Using an Image from ACR

A practical walkthrough for this exact scenario: you have a container image sitting in **Azure Container Registry (ACR)**, you package it as a **Helm chart**, and you deploy it to **AKS** using **Argo CD** (GitOps) instead of running `helm install` by hand.

---

## 1. The scenario, in one picture

```
Git repo (Helm chart + values.yaml)
        │  Argo CD watches this repo
        ▼
   Argo CD (running in AKS) ──▶ runs "helm template/install" for you ──▶ Deployment/Service/Ingress
        ▲
        │  pulls image
  Azure Container Registry (ACR) ◀── Customer's container image pushed here
        │
        ▼
   Running Pods, serving traffic
```

Four moving pieces:
1. **ACR** — holds the image.
2. **AKS ↔ ACR trust** — lets the cluster pull images without a manually managed secret.
3. **Helm chart** — the template that turns "an image + some config" into real Kubernetes objects (Deployment, Service, etc.), stored in Git.
4. **Argo CD** — watches the Git repo and continuously reconciles the cluster to match it, instead of anyone running `helm install` from a laptop.

---

## 2. Step 1 — Get the image into ACR

If the customer already gave you a built image, push it in:

```bash
az acr login --name myRegistry

docker tag customer-app:1.0 myregistry.azurecr.io/customer-app:1.0
docker push myregistry.azurecr.io/customer-app:1.0
```

Confirm it's there:

```bash
az acr repository show-tags --name myRegistry --repository customer-app
```

---

## 3. Step 2 — Let AKS pull from ACR (no secrets needed)

The clean way is to **attach** the ACR to the AKS cluster. This grants the cluster's managed identity `AcrPull` rights — no `imagePullSecrets`, no stored credentials, nothing to rotate.

```bash
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --attach-acr myRegistry
```

That's a one-time setup step per cluster. After this, any pod in the cluster can reference `myregistry.azurecr.io/...` images directly.

> If you can't attach the registry (e.g. cross-tenant/customer-owned ACR you don't manage), the fallback is a Kubernetes `imagePullSecret` — covered in section 6.

---

## 4. Step 3 — The Helm chart

Helm is just a templating tool: you write Kubernetes YAML once, with placeholders, and `values.yaml` fills in the specifics per environment (dev/staging/prod, or per customer).

**Chart layout:**

```
customer-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

**`Chart.yaml`:**

```yaml
apiVersion: v2
name: customer-app
description: Deployment for customer container
version: 0.1.0
appVersion: "1.0"
```

**`values.yaml`:**

```yaml
image:
  repository: myregistry.azurecr.io/customer-app
  tag: "1.0"
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  host: customer-app.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

**`templates/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**`templates/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

**`templates/ingress.yaml`:**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

---

## 5. Step 4 — Deploy it manually (fine for local testing)

Before wiring up Argo CD, it's worth confirming the chart works with a plain `helm install`:

```bash
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

helm install customer-app ./customer-app --namespace prod --create-namespace \
  --set image.tag=1.0
```

```bash
kubectl get pods -n prod
kubectl get ingress -n prod
```

Once that works, push the chart to Git and let Argo CD take over — see section 6.

---

## 6. Step 5 — Wire up Argo CD (GitOps)

Instead of running `helm install`/`helm upgrade` from a laptop or CI pipeline, Argo CD continuously watches a Git repo and keeps the cluster in sync with it. This is the standard way to combine Helm + AKS in a real, auditable deployment pipeline.

**Repo layout** (what Argo CD watches):

```
customer-app-gitops/
└── customer-app/          # the Helm chart from section 4
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

**6a. Install Argo CD into the AKS cluster (one-time):**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**6b. Let Argo CD pull images from ACR — same trust as the cluster itself.** Since ACR is already attached to AKS (section 3), no extra registry credentials are needed for pulling images; Argo CD only needs read access to the **Git repo**, not the registry.

**6c. Define an Argo CD `Application`** — this is the object that tells Argo CD "watch this Git path, deploy it with Helm, into this namespace":

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: customer-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/customer-app-gitops.git
    targetRevision: main
    path: customer-app
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: image.tag
          value: "1.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true       # remove resources deleted from Git
      selfHeal: true     # revert manual kubectl/helm changes back to match Git
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f customer-app-application.yaml -n argocd
```

**6d. From here on, deployments are Git-driven:**

- To ship a new image version: update `image.tag` in `values.yaml` (or the `Application` spec), commit, push.
- Argo CD detects the change and automatically runs the Helm upgrade for you — `syncPolicy.automated` means no one runs `helm upgrade` manually.
- `selfHeal: true` means if someone changes something directly with `kubectl`, Argo CD reverts it back to match Git — Git is the single source of truth.

**6e. Check sync status:**

```bash
kubectl get application customer-app -n argocd
argocd app get customer-app
```

Or use the Argo CD UI/CLI to see a live diff between what's in Git and what's actually running.

---

## 7. Fallback: if you *can't* attach the ACR

Sometimes the ACR belongs to a different tenant/subscription you don't control (common when it's genuinely "the customer's" registry). In that case, use a pull secret instead:

```bash
kubectl create secret docker-registry acr-secret \
  --namespace prod \
  --docker-server=customerregistry.azurecr.io \
  --docker-username=<sp-app-id> \
  --docker-password=<sp-password>
```

Then reference it in `values.yaml` and the Deployment template:

```yaml
# values.yaml
imagePullSecrets:
  - name: acr-secret
```

```yaml
# deployment.yaml, inside spec.template.spec
imagePullSecrets:
  {{- toYaml .Values.imagePullSecrets | nindent 8 }}
```

`az aks update --attach-acr` (section 3) is preferred whenever the registry is in your own tenant — it's secretless and nothing to rotate. The pull secret is the right tool specifically for a registry you don't own.

---

## 8. Summary

| Step | What it does |
|---|---|
| Push image to ACR | Gets the customer's container into a registry AKS can reach |
| `az aks update --attach-acr` | Grants the cluster identity pull rights — no stored secret |
| Helm chart (`Chart.yaml`, `values.yaml`, `templates/`) | Templates the Deployment/Service/Ingress so config is reusable per environment |
| Chart pushed to Git | Becomes the single source of truth Argo CD watches |
| Argo CD `Application` | Watches Git, runs the Helm deploy automatically, and self-heals drift |
| `imagePullSecrets` (fallback) | Needed only when the registry is outside your tenant/control |

This same chart + Argo CD `Application` pattern can be reused for every customer deployment — only `values.yaml` (image tag, hostname, resource sizing) and the target namespace change per customer. Nobody needs cluster `kubectl`/`helm` access day-to-day — they just open a pull request against the Git repo.
