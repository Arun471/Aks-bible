# Using Azure Application Gateway as Ingress in AKS

A plain-language explanation of how to get traffic into your AKS cluster using Azure's Application Gateway instead of a normal in-cluster ingress controller like NGINX.

---

## 1. The basic idea

Normally, when you set up Ingress in Kubernetes, you install something like NGINX as a pod *inside* your cluster. That pod does the routing.

With Azure, you have another option: let **Azure Application Gateway** (a real Azure resource, not a pod) do that job instead. Application Gateway is Azure's own Layer 7 load balancer — it already knows how to do SSL, routing by path/hostname, and it has a Web Application Firewall (WAF) built in.

So instead of:

```
Internet → Load Balancer → NGINX pod (inside cluster) → your app
```

You get:

```
Internet → Application Gateway (an Azure resource, outside the cluster) → your app
```

No ingress controller pod eating cluster resources. Azure runs and scales the Gateway for you.

---

## 2. Two ways to do this: AGIC vs. AGC

There are actually two products. It's worth knowing both because Microsoft is moving from the first to the second.

### AGIC (Application Gateway Ingress Controller) — the original

- A small pod runs inside your AKS cluster.
- It watches your normal Kubernetes `Ingress` objects.
- Whenever something changes, it updates the real Application Gateway to match.
- You still write standard Kubernetes `Ingress` YAML — AGIC just translates it.

### AGC (Application Gateway for Containers) — the newer, recommended one

- A newer Azure product, purpose-built for Kubernetes.
- Updates happen much faster (near real-time, instead of AGIC's slower reconcile loop).
- Supports the modern Kubernetes **Gateway API**, not just the older `Ingress` API.
- Microsoft now recommends this over AGIC for new projects, partly because the standalone NGINX Ingress Controller is being retired and teams need a new default.

**Plain rule of thumb:** if you're setting this up brand new today, prefer AGC. If you're maintaining something older, you'll likely see AGIC.

---

## 3. How AGIC actually works, step by step

1. You create an Application Gateway in Azure (a normal Azure resource, like a VM or a database).
2. You turn on the AGIC add-on on your AKS cluster and point it at that Application Gateway.
3. AGIC installs a pod in your cluster.
4. You write a totally normal `Ingress` YAML file, just like you would for NGINX.
5. AGIC notices it, and configures the real Application Gateway to match — listeners, routing rules, backend pools, health probes.
6. Traffic then flows straight from the internet into Application Gateway, and Application Gateway sends it to your pods — the AGIC pod itself isn't in the traffic path, it just configures things.

---

## 4. Simple example

**Step 1 — Create the Application Gateway (once, outside Kubernetes):**

```bash
az network application-gateway create \
  --name myAppGateway \
  --resource-group myResourceGroup \
  --sku Standard_v2 \
  --public-ip-address myAppGatewayIP \
  --vnet-name myVnet \
  --subnet myAppGatewaySubnet
```

**Step 2 — Turn on AGIC on your AKS cluster:**

```bash
APPGW_ID=$(az network application-gateway show \
  --resource-group myResourceGroup \
  --name myAppGateway \
  --query id -o tsv)

az aks enable-addons \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --addon ingress-appgw \
  --appgw-id "$APPGW_ID"
```

**Step 3 — Deploy your app like normal:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: myacr.azurecr.io/myapp:v1
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 8080 }]
```

**Step 4 — The Ingress, pointing at Application Gateway instead of NGINX:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
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

That one annotation — `kubernetes.io/ingress.class: azure/application-gateway` — is the only real difference from an NGINX Ingress file. AGIC sees this Ingress, reaches out to Azure, and configures the Application Gateway to route `myapp.example.com` to your pods.

---

## 5. Why choose this over NGINX

| | In-cluster NGINX | Application Gateway (AGIC/AGC) |
|---|---|---|
| Where it runs | Pod inside your cluster | Managed Azure resource, outside the cluster |
| Uses cluster CPU/memory | Yes | No |
| WAF (firewall) | Needs separate setup | Built in |
| TLS certs from Key Vault | Needs extra tooling | Native integration |
| Who patches/scales it | You | Azure |
| Best for | Simple, portable, cloud-agnostic setups | Azure-native, enterprise setups needing WAF/compliance |

---

## 6. Summary

- Application Gateway can act as your Ingress instead of NGINX — Azure runs the load balancer itself, not a pod in your cluster.
- **AGIC** is the original way: a small in-cluster pod translates your normal `Ingress` YAML into Application Gateway config.
- **AGC** is the newer, faster, Gateway-API-native replacement — the one Microsoft now recommends for new clusters.
- Either way, you keep writing familiar Kubernetes YAML; only the `ingress.class` (or Gateway API config, for AGC) changes.
