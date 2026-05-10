## Ingress Controllers in Kubernetes

### Key Concept
- **Ingress resource**: defines routing rules (what traffic goes where)
- **Ingress controller**: the actual software that enforces those rules
- Without a controller, Ingress resources do nothing

### Why Vanilla K8s Has No Built-in Controller
- CNCF avoids favoring specific ecosystem projects
- So no Ingress controller ships with upstream Kubernetes

### How You Get One
| Source | Example |
|---|---|
| K8s distribution (GKE, EKS, etc.) | Usually pre-installed or offered |
| Manual install | NGINX, Traefik, HAProxy Ingress |

### Bottom Line
> If you're on vanilla Kubernetes, you **must** install an Ingress controller
> yourself before any Ingress resources will work.

## Installing NGINX Ingress Controller (Vanilla K8s Only)

> Skip this on GKE/EKS/managed clusters — they handle it for you.
> On the CKAD exam cluster, it's likely pre-installed too.

### Step 1 — Install via Helm
```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

**Flag breakdown:**

| Flag | Meaning |
|---|---|
| `--install` | Install if not exists, upgrade if it does |
| `ingress-nginx` (1st) | Helm release name |
| `ingress-nginx` (2nd) | Chart name |
| `--repo` | Chart repository URL |
| `--namespace ingress-nginx` | Deploy into this namespace |
| `--create-namespace` | Create the namespace if it doesn't exist |


### Step 2 — Verify It's Running
```bash
kubectl get pods -n ingress-nginx
```
Expect to see an `ingress-nginx-controller-*` pod in `Running` state.


## Ingress on Minikube

### Enable Built-in NGINX Ingress Addon
```bash
minikube addons enable ingress

# Verify
kubectl get pods -n ingress-nginx
```
> No Helm needed — minikube packages the same NGINX controller as an addon.

### IngressClass
```bash
kubectl get ingressclass
# NAME    CONTROLLER             DEFAULT
# nginx   k8s.io/ingress-nginx   false
```
Same `ingressClassName: nginx` in your YAML — no difference.

### Accessing Ingress Locally (Minikube Gotcha)
Minikube has no real LoadBalancer IP by default. Two options:

| Option | Command | Notes |
|---|---|---|
| Tunnel | `minikube tunnel` | Runs in separate terminal, assigns real IP |
| Manual | `minikube ip` | Get cluster IP → add to `/etc/hosts` |

### Bottom Line
- Use the addon, not Helm
- Same YAML as any other cluster
- Just need `minikube tunnel` to hit it from browser


## Example
```shell
# 1. Enable ingress addon (minikube only)
minikube addons enable ingress

# 2. Create a deployment
kubectl create deployment myapp --image=nginx --port=80

# 3. Expose it as a ClusterIP service
kubectl expose deployment myapp --port=80 --target-port=80

# 4. Verify both are up
kubectl get deployment myapp
kubectl get service myapp

# 5. Create the Ingress
kubectl create ingress myapp-ingress \
  --class=nginx \
  --rule="myapp.local/=myapp:80"

# 6. Verify ingress
kubectl get ingress myapp-ingress
kubectl describe ingress myapp-ingress   # check ADDRESS field gets populated

# 7. Start tunnel (separate terminal)
minikube tunnel

# 8. Add to /etc/hosts (one-time)
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts

# 9. Test
curl http://myapp.local
```

## Kubernetes Ingress — Core Concept

### What Ingress Solves
Without Ingress: every app needs its own LoadBalancer → separate IP, separate cost.
With Ingress: one controller, one IP, routes to many apps.

### How It Works — Virtual Hosting
Both domains resolve to the same IP. NGINX reads the HTTP `Host` header to decide where to forward.

### Two Routing Strategies

| Strategy | Example | Use Case |
|---|---|---|
| By hostname | `myapp.local` vs `myapp2.local` | Separate apps |
| By path | `myapp.local/app1` vs `myapp.local/app2` | Same domain, different services |

### Imperative Commands

```shell
# Hostname-based
kubectl create ingress myapp2-ingress \
  --class=nginx \
  --rule="myapp2.local/=myapp2:80"

# Path-based (multiple rules)
kubectl create ingress combined-ingress \
  --class=nginx \
  --rule="myapp.local/app1=myapp:80" \
  --rule="myapp.local/app2=myapp2:80"

### /etc/hosts (local dev only)
127.0.0.1  myapp.local
127.0.0.1  myapp2.local   # same IP — Host header does the routing
```

### Big Picture
Browser
  │
  ▼
NGINX Ingress Controller  (one IP)
  ├── myapp.local   →  myapp svc   →  myapp pods
  └── myapp2.local  →  myapp2 svc  →  myapp2 pods

## Production Step Flow
```
1. Buy domains          myapp.com / myapp2.com        (Route53, Namecheap, etc.)
        │
2. Buy static IP        from GCP/AWS                  (one IP for both)
        │
3. DNS A Records        myapp.com  → 34.x.x.x
                        myapp2.com → 34.x.x.x          (same IP!)
        │
4. Ingress Controller   gets that static IP assigned
        │
5. Ingress Rules        route by Host header           (same as local)
```
