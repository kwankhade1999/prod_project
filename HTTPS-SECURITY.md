# HTTPS Security Architecture

This document explains how HTTPS is implemented in this GitOps project using
NGINX Ingress Controller, cert-manager, and Let's Encrypt for the domain `quntamvector.in`.

---

## Table of Contents

1. [Overview](#overview)
2. [Full Traffic Flow](#full-traffic-flow)
3. [Components Involved](#components-involved)
4. [ClusterIssuer — How the Certificate is Obtained](#clusterissuer--how-the-certificate-is-obtained)
5. [Ingress — How Traffic is Routed and Secured](#ingress--how-traffic-is-routed-and-secured)
6. [Where Security Comes In](#where-security-comes-in)
7. [Certificate Lifecycle](#certificate-lifecycle)
8. [Why Internal Services Use HTTP](#why-internal-services-use-http)
9. [Troubleshooting](#troubleshooting)

---

## Overview

When a user visits `https://quntamvector.in/login`, the request is:

1. **Encrypted** end-to-end between the browser and the NGINX Ingress Controller
2. **Routed** by NGINX to the correct internal service
3. **Decrypted** at the NGINX layer before forwarding to the frontend pod

The TLS certificate that enables HTTPS is automatically issued and renewed by
**cert-manager** using **Let's Encrypt** — a free, trusted Certificate Authority.

---

## Full Traffic Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          OUTSIDE CLUSTER                                │
│                                                                         │
│   User Browser                                                          │
│   https://quntamvector.in/login                                        │
│          │                                                              │
│          │  HTTPS (port 443) — encrypted with TLS certificate           │
│          ▼                                                              │
│   AWS Elastic Load Balancer                                             │
│   (created by NGINX Ingress Controller on EKS)                         │
│          │                                                              │
│          │  Forwards raw TCP traffic to NGINX pods                      │
└──────────┼──────────────────────────────────────────────────────────────┘
           │
┌──────────┼──────────────────────────────────────────────────────────────┐
│          │              INSIDE CLUSTER                                  │
│          ▼                                                              │
│   NGINX Ingress Controller Pod                                          │
│   ┌─────────────────────────────────────────────────┐                  │
│   │  1. Matches host: quntamvector.in               │                  │
│   │  2. Loads certificate from secret: frontend-tls  │                  │
│   │  3. Terminates TLS (decrypts HTTPS → HTTP)       │                  │
│   │  4. Checks ssl-redirect: if HTTP → redirect 443  │                  │
│   │  5. Forwards to backend service on port 80       │                  │
│   └─────────────────────────────────────────────────┘                  │
│          │                                                              │
│          │  Plain HTTP (port 80) — internal only, never leaves cluster  │
│          ▼                                                              │
│   frontend Service (ClusterIP: 80)                                      │
│          │                                                              │
│          ▼                                                              │
│   frontend Pod (port 8080)                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Components Involved

| Component | Role |
|---|---|
| **AWS ELB** | Single public entry point — accepts traffic on ports 80 and 443 |
| **NGINX Ingress Controller** | Routes traffic by host/path, terminates TLS |
| **cert-manager** | Automates certificate issuance and renewal |
| **ClusterIssuer** | Defines how cert-manager talks to Let's Encrypt |
| **Let's Encrypt** | Free Certificate Authority — issues trusted TLS certificates |
| **frontend-tls Secret** | Kubernetes secret that stores the issued certificate and private key |
| **frontend Service** | ClusterIP service — internal only, not exposed to internet |

---

## ClusterIssuer — How the Certificate is Obtained

The `ClusterIssuer` is a cert-manager resource that defines the certificate authority
and the method used to prove domain ownership.

```yaml
# base/frontend/clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: manojdevopstest@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

### Field by Field Explanation

| Field | Value | What it Does |
|---|---|---|
| `server` | Let's Encrypt production URL | Points to the real Let's Encrypt CA (not staging) |
| `email` | manojdevopstest@gmail.com | Let's Encrypt sends expiry warnings to this address |
| `privateKeySecretRef.name` | `letsencrypt-prod-account-key` | Stores your Let's Encrypt account private key securely in a Kubernetes secret |
| `solvers.http01` | via NGINX ingress | Uses HTTP-01 challenge to prove domain ownership (explained below) |

### How Let's Encrypt HTTP-01 Challenge Works

This is how Let's Encrypt verifies you own `quntamvector.in` before issuing a certificate:

```
Step 1: cert-manager contacts Let's Encrypt
        "I want a certificate for quntamvector.in"

Step 2: Let's Encrypt responds with a challenge
        "Place this token at: http://quntamvector.in/.well-known/acme-challenge/<random-token>"

Step 3: cert-manager creates a temporary Ingress rule and pod
        that serves the token at that URL path

Step 4: Let's Encrypt makes an HTTP request to that URL from the internet
        If the token matches → domain ownership PROVED

Step 5: Let's Encrypt issues the signed TLS certificate

Step 6: cert-manager stores the certificate in Kubernetes secret: frontend-tls

Step 7: NGINX Ingress loads the certificate from the secret automatically

Step 8: cert-manager cleans up the temporary challenge resources
```

> **Why HTTP-01?** Because it only requires port 80 to be publicly accessible,
> which the NGINX Ingress LoadBalancer already provides.

---

## Ingress — How Traffic is Routed and Secured

The Ingress resource is the rule book for NGINX — it defines what traffic comes in,
how it is secured, and where it goes.

```yaml
# base/frontend/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  labels:
    app: frontend
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - quntamvector.in
    secretName: frontend-tls
  rules:
  - host: quntamvector.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

### Annotation Explanation

| Annotation | What it Does |
|---|---|
| `ssl-redirect: "true"` | Any request to `http://quntamvector.in` is automatically redirected to `https://quntamvector.in` with HTTP 308 |
| `cert-manager.io/cluster-issuer: letsencrypt-prod` | Tells cert-manager to watch this Ingress and issue a certificate using the `letsencrypt-prod` ClusterIssuer |

### Spec Explanation

| Field | What it Does |
|---|---|
| `ingressClassName: nginx` | Targets the NGINX Ingress Controller specifically |
| `tls.hosts` | Declares that `quntamvector.in` must be served over HTTPS |
| `tls.secretName: frontend-tls` | NGINX loads the certificate from this Kubernetes secret to encrypt traffic |
| `rules.host` | NGINX only processes requests where the HTTP Host header matches `quntamvector.in` — all other domains are rejected |
| `path: /` with `pathType: Prefix` | All URL paths (`/login`, `/cart`, `/checkout` etc.) are routed to the frontend service |
| `backend.service.port: 80` | After TLS termination, NGINX forwards plain HTTP to the frontend ClusterIP service on port 80 |

---

## Where Security Comes In

### Layer 1 — Encryption in Transit (TLS)
- All traffic between the user's browser and NGINX is encrypted using **TLS 1.2/1.3**
- Passwords, session tokens, cookies, and personal data cannot be intercepted
- Enforced by the `frontend-tls` certificate issued by Let's Encrypt

### Layer 2 — Forced HTTPS
- `ssl-redirect: "true"` ensures no user ever accidentally uses plain HTTP
- Even if someone types `http://quntamvector.in`, they are immediately redirected to HTTPS
- Prevents accidental exposure of sensitive data over unencrypted connections

### Layer 3 — Trusted Certificate Authority
- The certificate is signed by **Let's Encrypt**, which is trusted by all major browsers
- The browser shows the padlock icon — users can verify the site is genuine
- Prevents man-in-the-middle attacks and phishing via fake sites

### Layer 4 — Single Entry Point
- **Only NGINX Ingress** has a public IP (via AWS ELB)
- All other services (`authservice`, `cartservice`, `paymentservice`, etc.) use `ClusterIP` — they have no external IP and are completely unreachable from the internet
- Attack surface is minimised to a single controlled entry point

### Layer 5 — Domain Validation
- The `host: quntamvector.in` rule means NGINX rejects requests for any other domain
- Prevents host header injection attacks

### Layer 6 — Automatic Certificate Renewal
- Let's Encrypt certificates are valid for **90 days**
- cert-manager automatically renews them **30 days before expiry**
- No manual intervention needed — the certificate never silently expires

---

## Certificate Lifecycle

```
Day 0:   ClusterIssuer + Ingress applied
              │
              ▼
         cert-manager detects Ingress annotation
              │
              ▼
         HTTP-01 Challenge created → Let's Encrypt verifies domain
              │
              ▼
         Certificate issued → stored in secret: frontend-tls
              │
              ▼
         NGINX loads certificate → HTTPS is live
              │
Day 60:       ▼
         cert-manager detects certificate expires in 30 days
              │
              ▼
         Automatic renewal triggered → new certificate issued
              │
              ▼
         NGINX hot-reloads new certificate — zero downtime
```

---

## Why Internal Services Use HTTP

A common question is: *"If HTTPS is important, why do internal services use plain HTTP?"*

The answer is **defence in depth with pragmatism**:

| Aspect | External (Internet → NGINX) | Internal (NGINX → Services) |
|---|---|---|
| Network | Public internet — untrusted | Kubernetes cluster network — private VPC |
| Encryption needed | Yes — data crosses untrusted networks | Optional — traffic never leaves AWS VPC |
| Certificate management | Handled by cert-manager | Would require internal CA for each service |
| Complexity | Low (one certificate) | High (certificates for every service) |
| Risk | High without HTTPS | Low — VPC network isolation provides security |

For production environments requiring stricter security (e.g. PCI-DSS, HIPAA), you would
add **mTLS between services** using a service mesh like **Istio** — the Istio sidecar
annotations already present in the frontend and loadgenerator deployments indicate
this is a planned future enhancement for this project.

---

## Troubleshooting

### Certificate not being issued
```bash
# Check ClusterIssuer is Ready
kubectl get clusterissuer letsencrypt-prod

# Check certificate status
kubectl get certificate
kubectl describe certificate frontend-tls

# Check the ACME challenge
kubectl get challenges
kubectl describe challenge
```

### HTTPS not working after certificate is issued
```bash
# Check NGINX has loaded the certificate
kubectl describe ingress frontend-ingress

# Check NGINX controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### Certificate stuck in Pending
Most common cause: **DNS not pointing to the NGINX LoadBalancer IP**.
Verify your domain A/CNAME record:
```bash
# Get NGINX external address
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Check DNS resolution
nslookup quntamvector.in
```

The IP/CNAME from the first command must match what `nslookup` returns.
If they don't match, update your domain DNS records and wait for propagation (up to 48 hours).
