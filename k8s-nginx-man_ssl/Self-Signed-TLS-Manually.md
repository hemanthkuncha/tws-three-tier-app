
# Self-Signed TLS Setup for Kubernetes (Local Dev)

This guide explains how to enable **self-signed TLS certificates** for your Kubernetes-deployed three-tier application using **NGINX Ingress**.

---

## 1. Install NGINX Ingress Controller

If not already installed:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

Check status:

```bash
kubectl get pods -n ingress-nginx
```

---

## 2. Generate Self-Signed TLS Certificate

### Step 1: Generate certs using OpenSSL

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.local/O=myapp"
```

### Step 2: Create Kubernetes TLS secret

```bash
kubectl create secret tls myapp-tls \
  --cert=tls.crt --key=tls.key \
  -n three-tier
```

>  **Warning:** Do **not** use this in production.

---

## 3. Create an Ingress Resource

Save the following as `three-tier-ingress.yaml`:

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

Apply the Ingress:

```bash
kubectl apply -f three-tier-ingress.yaml
```

---

## 4. Add DNS Entry for Local Access

Edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add this line (replace with your VMβs IP):

```
10.0.0.40 myapp.local
```

Then restart networking (optional):

```bash
sudo systemctl restart systemd-networkd
```

---

## 5. Access Your App

### Browser HTTP & HTTPS (shows TLS warning β accept to proceed):

```
http://myapp.local:32475
https://myapp.local:31444
```

### Test using cURL:

```bash
curl -k https://myapp.local:31444           # Frontend ( "-k" ignores the warnings )
curl -k https://myapp.local:31444/api/tasks # Backend
```

---

## Notes

- Self-signed certs are only for dev/testing purposes.
- Donβt check in real certificates.
- You can store the certs under a folder like `kubernetes/dev-tls/` and `.gitignore` it:
  
```bash
echo "kubernetes/dev-tls/" >> .gitignore
```

- For public domains and production, use [cert-manager](https://cert-manager.io) with Let's Encrypt.

---
