# k3s-argocd-pi

GitOps + Argo CD on k3s, MetalLB, Monitoring, and Ingress.

---

## Current Setup

- **k3s** (lightweight Kubernetes)
- **Argo CD** (GitOps control plane)
- **MetalLB** (LoadBalancer IPs in local network)
- **Monitoring** (Prometheus + Grafana)
- **Nginx test app**
- **Ingress-nginx** (Ingress controller via MetalLB)

---

## Steps

### 1. Install k3s
Follow k3s installation steps

### 2. Install Argo CD
Argo CD is installed in the `argocd` namespace.  
Access Argo CD at:

```
http://192.168.178.212
```

Login using the admin password (changed via UI).

### 3. Install MetalLB
MetalLB provides external IPs from the pool `192.168.178.210-220`.  
Each service can get a fixed IP.

### 4. Monitoring (Prometheus + Grafana)
Deployed via Argo CD Helm chart.

- Grafana URL: [http://192.168.178.213](http://192.168.178.213)
- Retrieve admin password if not overridden:
  ```bash
  kubectl -n monitoring get secret monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d
  ```

### 5. Nginx Test App
- Service IP: `http://192.168.178.211`
- Used as a sample workload.

### 6. Ingress-nginx
Deployed via Argo CD.

#### Application Manifest: `argocd-apps/ingress-nginx-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.*
    helm:
      values: |
        controller:
          service:
            type: LoadBalancer
            loadBalancerIP: 192.168.178.214
            externalTrafficPolicy: Local
          publishService:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply and sync:
```bash
kubectl -n argocd apply -f argocd-apps/ingress-nginx-app.yaml
argocd app sync ingress-nginx
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

#### Example Ingress for nginx app
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: nginx.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

Update `/etc/hosts` on your laptop/PC:
```
192.168.178.214  nginx.local
```

Access via: [http://nginx.local](http://nginx.local)

---

## Next Steps
- Add cert-manager for TLS (Let’s Encrypt)
- Deploy real workloads via GitOps
- Add dashboards & alerts to Grafana
