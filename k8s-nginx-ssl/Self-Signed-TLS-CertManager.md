# 🔐 TLS Setup with cert-manager (Self‑Signed Certificates for ArgoCD)

This guide explains how to automate TLS certificate generation using [cert-manager](https://cert-manager.io/) in a Kubernetes cluster — ideal for ArgoCD‑managed projects.

---

## 📁 Recommended Directory Structure

```bash
project-root/
└── kubernetes/
├── certs/
│ ├── selfsigned-issuer.yaml
│ └── tls-certificate.yaml
└── ingress/
└── three-tier-ingress.yaml
```
---

## ✅ Prerequisites

- A working Kubernetes cluster (e.g., MicroK8s, Minikube, etc.)
- An Ingress controller installed (e.g., NGINX)
- `kubectl` access with sufficient permissions
- ArgoCD set up (optional but ideal for GitOps-based deployments)

---

## 🧩 Step 1: Install cert-manager

Apply the cert-manager manifests (only once per cluster):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```
Verify it's running:

```bash
kubectl get pods -n cert-manager
```

🏗️ Step 2: Create a ClusterIssuer (Self‑Signed)

Create kubernetes/certs/selfsigned-issuer.yaml:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

Apply it:

```bash
kubectl apply -f kubernetes/certs/selfsigned-issuer.yaml
```

🎯 Step 3: Create a Certificate Resource

Create kubernetes/certs/tls-certificate.yaml:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: three-tier
spec:
  secretName: myapp-tls
  duration: 8760h       # 365 days
  renewBefore: 720h     # 30 days before expiry
  subject:
    organizations:
      - myapp
  commonName: myapp.local
  dnsNames:
    - myapp.local
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
```

Apply it:

```bash
kubectl apply -f kubernetes/certs/tls-certificate.yaml
```

✅ This will automatically create the secret myapp-tls that your Ingress can use.

🧱 Step 4: Configure the Ingress

Ensure your Ingress YAML (kubernetes/ingress/three-tier-ingress.yaml) contains:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    #nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.local
      secretName: myapp-tls
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
```

Apply it:

```bash
kubectl apply -f kubernetes/ingress/three-tier-ingress.yaml
```
---

🧪 Step 5: Test It

```bash
kubectl get secret -n three-tier
kubectl describe certificate myapp-tls -n three-tier
curl -k https://myapp.local:30101
```

---
