# Kubernetes Networking Made Simple: LoadBalancer, Ingress, and Gateway API

## The Problem We're Solving

You've deployed an app inside Kubernetes. It's running happily, but it's **invisible to the outside world**. Pods get IP addresses that change constantly and aren't reachable from the internet. So how do real users hit your app with a browser?

That's what these three things are for. Think of them as three generations of "front door" for your cluster, each solving the same core problem in a different way.

---

## 1. LoadBalancer (Service type)

**What it is:** The simplest way to expose one app to the outside world. When you create a `LoadBalancer` Service, Kubernetes asks your cloud provider (AWS, GCP, Azure, etc.) to spin up a **real external load balancer** with its own public IP address, pointed at your pods.

**Analogy:** Imagine renting a single dedicated phone line just for one shop. It works great, but if you have 10 shops, you need 10 phone lines — that gets expensive and hard to manage.

**Simple example:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

Apply it, and your cloud provider gives you an external IP:

```bash
kubectl get svc my-app-service
# EXTERNAL-IP: 34.123.45.67
```

Now `http://34.123.45.67` reaches your app directly.

**Limitation:** One LoadBalancer = one external IP = one cloud bill. If you have 20 microservices, you'd need 20 load balancers unless you use something smarter — which is exactly where Ingress comes in.

---

## 2. Ingress

**What it is:** A single entry point that sits in front of *many* services, routing traffic based on the URL path or hostname. Instead of one load balancer per app, you get **one load balancer for the whole cluster**, with rules deciding where each request goes.

**Analogy:** Instead of 10 phone lines for 10 shops, you have one receptionist (the Ingress Controller) who answers every call and transfers it to the right shop based on what the caller asks for.

**How it works:**
- You deploy an **Ingress Controller** (e.g., NGINX, Traefik) — this is the actual traffic router.
- You create **Ingress resources** — simple rulebooks saying "path X goes to service Y."

**Simple example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /shop
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
          - path: /blog
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```

Now:
- `myapp.example.com/shop` → `shop-service`
- `myapp.example.com/blog` → `blog-service`

One IP, many apps, routed by path.

**Limitation:** Ingress was designed for basic HTTP routing. It struggles with more advanced needs (traffic splitting, gRPC, multiple teams sharing rules safely), and every controller vendor ends up bolting on custom annotations to fill the gaps — leading to inconsistent, non-portable configs.

---

## 3. Gateway API

**What it is:** The modern, more powerful successor to Ingress. It's a set of Kubernetes APIs designed to fix Ingress's limitations with a cleaner, **role-oriented** model and native support for advanced routing.

**Analogy:** Instead of one receptionist with a single messy rulebook, you now have:
- An **infrastructure team** that sets up the building's main switchboard (`Gateway`)
- Individual **teams** who each write their own routing rules for their shop (`HTTPRoute`) without touching anyone else's

This separation of concerns is the big shift: infra teams manage the entry point, app teams manage their own routes independently.

**Key pieces:**
| Resource | Purpose |
|---|---|
| `GatewayClass` | Defines *which* controller implements the gateway (like a template) |
| `Gateway` | The actual listener — IP, port, protocol (managed by infra/platform team) |
| `HTTPRoute` | Routing rules for a specific app (managed by app teams) |

**Simple example:**

```yaml
# 1. The Gateway (set up once by the platform team)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
# 2. The Route (owned by the app team)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /shop
      backendRefs:
        - name: shop-service
          port: 80
```

**Why it's better than Ingress:**
- Native support for traffic splitting (e.g., 90% to v1, 10% to v2 for canary releases)
- Works cleanly across HTTP, gRPC, TCP, and TLS
- Multiple teams can safely manage their own routes without stepping on each other
- No vendor-specific annotation soup — it's a standard, portable API

---

## Quick Comparison

| | LoadBalancer | Ingress | Gateway API |
|---|---|---|---|
| Exposes | One service | Many services (HTTP only) | Many services (HTTP, gRPC, TCP, TLS) |
| External IPs needed | One per service | One for the whole cluster | One for the whole cluster |
| Routing rules | None (direct) | Path/host based | Path/host + traffic splitting, weighting |
| Who manages it | Cloud provider | One controller, one config style | Split between infra team and app teams |
| Best for | Quick single-app exposure | Simple multi-app HTTP routing | Complex, multi-team, production-grade routing |

## Which Should You Use?

- **Testing one app quickly, or a single microservice?** → `LoadBalancer`
- **Several apps, simple path/host routing, small team?** → `Ingress`
- **Multiple teams, advanced routing, or building for the long term?** → `Gateway API` (this is where Kubernetes networking is heading — Ingress is now considered a legacy API in maintenance mode)
