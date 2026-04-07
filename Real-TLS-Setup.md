
# Real TLS Setup with Let's Encrypt and cert-manager (Recommended for Production)

To avoid TLS warnings and ensure secure HTTPS access, you should use **Let's Encrypt** certificates managed by **cert-manager** in your Kubernetes cluster.

---

## Prerequisites

- A **real domain** (e.g., `todo.example.com`)
- DNS record (`A` or `CNAME`) pointing to your cluster's **external IP**
- Ingress Controller with **public access**
- cert-manager installed

---

## 1. Install cert-manager

Apply the cert-manager manifest (v1.14.1):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.1/cert-manager.yaml
```

Verify it's running:

```bash
kubectl get pods -n cert-manager
```

---

## 2. Create a ClusterIssuer for Let's Encrypt (HTTP-01)

Save the following as `cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

---

## 3. Update Your Ingress Resource

Update your Ingress to include TLS and cert-manager annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
        - todo.example.com
      secretName: letsencrypt-todo-cert
  ingressClassName: nginx
  rules:
    - host: todo.example.com
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

Apply the Ingress:

```bash
kubectl apply -f three-tier-ingress.yaml
```

---

## What cert-manager Does Automatically

 Provisions TLS certificates from Let's Encrypt  
 Automatically renews them before they expire  
 Stores them in the specified Kubernetes Secret

---

## Tips

- Replace `todo.example.com` with your **real domain**
- Make sure **DNS is configured correctly** before applying Ingress
- Ensure your **Ingress controller has a reachable external IP**
- For private/internal clusters, use DNS-01 validation instead

 More info: [https://cert-manager.io/docs/](https://cert-manager.io/docs/)

---
