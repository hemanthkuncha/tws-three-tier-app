### 1.The Essential Stack (The "Needs")

DNS	- Internal service discovery

MetalLB	- Provides the External IP for your Ingress

Ingress	- The Gateway (NGINX)

Cert-Manager - Automation for SSL certificates

Helm3	- Used to install the DuckDNS Webhook

### 1.2 Create namespace 
```bash
kubectl create ns three-tier
```
### 2.Serial Execution Steps (The "Order")

2.1: Webhook Installation -- Cert-manager cannot talk to DuckDNS without the Webhook.

Add Helm Repository
```bash
microk8s helm3 repo add csp33 https://csp33.github.io/cert-manager-duckdns-webhook
```
```bash
microk8s helm3 repo update
```
Install Webhook (Ensure groupName matches your domain)

```bash
microk8s helm3 install cert-manager-duckdns-webhook csp33/cert-manager-duckdns-webhook \
  --namespace cert-manager \
  --set groupName=acme.mysite06.duckdns.org
```
2.2: Infrastructure Credentials -- Apply your DuckDNS token secret specifically in the `cert-manager` namespace.

```bash
kubectl apply -f duck-dns-tocken
```

OR

```bash
microk8s kubectl create secret generic duckdns-token-secret \
  -n cert-manager \
  --from-literal=token=YOUR_TOKEN_HERE
```
2.3: The "Issuer" (The Authority) -- Apply your `letsencrypt-issuer.yaml`. This establishes the connection to Let's Encrypt.

Check Status: `kubectl describe clusterissuer letsencrypt-prod`

Look for: `Ready: True / ACME account registered ` (Wait for 2 mins-120 sec)

2.4: Application Deployment -- Apply your Three-Tier App (MongoDB, API, Frontend Services).

2.5: The "Trigger" (Ingress + Certificate) -- Apply your three-tier-ingress.yaml. Crucial: The Ingress is what triggers cert-manager to start the SSL challenge

## 🔐 SSL Management Strategy

### ❌ Manual Method (Legacy/Testing)
Previously, SSL was handled via OpenSSL:
1. Generated `.key` and `.cert` manually.
2. Created Kubernetes Secret: `kubectl create secret tls old-tls --key key.pem --cert cert.pem`.
3. Cons: Manual renewal required, browsers show "Not Secure" warnings.

### ✅ Automated Method (Production)
Now handled by **cert-manager + DuckDNS Webhook**:
1. Cert-manager detects the `tls` block in the Ingress.
2. It requests a valid certificate from Let's Encrypt.
3. It automatically stores the cert/key in the `myapp-tls-prod` secret.
4. **Pros:** Fully trusted by browsers, automated 90-day renewals.

## Real TLS Setup with Let's Encrypt, cert-manager, and DuckDNS

This guide covers setting up **DNS-01** challenge validation for a Kubernetes cluster using **DuckDNS**. This method is preferred for local/home environments because it doesn't require opening Port 80 and supports internal IP addresses.

---

## 📋 Prerequisites

* **DuckDNS Account:** A registered domain (e.g., `mysite06.duckdns.org`).
* **API Token:** Your DuckDNS account token.
* **cert-manager:** Installed and running (`v1.14.1+`).
* **DuckDNS Webhook:** Installed in your cluster (required for DNS-01).

---

## 1. Create the API Token Secret

The webhook needs your DuckDNS token to create temporary TXT records for verification. Store it in the `cert-manager` namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: duckdns-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  token: YOUR_DUCKDNS_TOKEN_HERE
```

## 2. Configure the ClusterIssuer
Save as letsencrypt-issuer.yaml. This acts as the "authority" that requests certificates from Let's Encrypt.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: [https://acme-v02.api.letsencrypt.org/directory](https://acme-v02.api.letsencrypt.org/directory)
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.mysite06.duckdns.org # Must match your installed webhook group
          solverName: duckdns
          config:
            apiTokenSecretRef:
              name: duckdns-token-secret
              key: token
```

## 3. Configure the Ingress Resource
Update your three-tier-ingress.yaml to include the TLS block and the cert-manager annotation.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - mysite06.duckdns.org
    secretName: myapp-tls-prod
  rules:
  - host: mysite06.duckdns.org
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3500
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```
## 4. Troubleshooting & "Zombie" Loop Recovery
If challenges get stuck in Pending or recreate themselves after deletion, follow this Circuit Breaker sequence:

#### Step 1: Delete the Trigger
```Bash
kubectl delete ingress three-tier-ingress -n three-tier
```
#### Step 2: Clear the State
```Bash
kubectl delete certificate myapp-tls-prod -n three-tier
kubectl delete orders --all -n three-tier
kubectl delete challenges --all -n three-tier --force --grace-period=0
```
#### Step 3: Strip Finalizers (If objects are stuck)
```Bash
kubectl patch challenge <CHALLENGE_NAME> -n three-tier --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```
#### Step 4: Re-Apply
Fix the ClusterIssuer if needed, then re-apply your YAML files in order:
```Bash
kubectl apply -f letsencrypt-issuer.yaml

kubectl apply -f three-tier-ingress.yaml
```
## 5. Verification Commands

Check Cert Status: `kubectl get certificate -n three-tier`

Check Challenges: `kubectl get challenges -n three-tier`

Check Issuer: `kubectl describe clusterissuer letsencrypt-prod`
