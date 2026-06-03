# GitOps Base — Kubernetes Manifests

This directory contains the Kustomize base manifests for the microservices platform.
External traffic is served over **HTTPS** via NGINX Ingress + cert-manager (Let's Encrypt).

---

## Prerequisites

- A Kubernetes cluster with `kubectl` access
- Domain `quntamvector.in` DNS A record pointing to your cluster's external IP
- NGINX Ingress Controller installed (Step 1 below)

---

## Step 1: Install NGINX Ingress Controller

**For a managed cloud cluster (GKE / EKS / AKS):**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

Wait for the controller to be ready:
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Get the external IP assigned to the NGINX controller — you will need this for your DNS A record:
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

> Point your domain `quntamvector.in` A record to the `EXTERNAL-IP` shown above before continuing.

---

## Step 2: Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

Wait for cert-manager to be ready:
```bash
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

Verify all cert-manager pods are running:
```bash
kubectl get pods -n cert-manager
```

---

## Step 3: Apply the Manifests

This will deploy all services, the ClusterIssuer, and the Ingress:
```bash
kubectl apply -k base/
```

> **Note:** The ClusterIssuer (`letsencrypt-prod`) is included in the base manifests.
> cert-manager will automatically provision a TLS certificate for `quntamvector.in`.

---

## Step 4: Verify the Certificate is Issued

Check the certificate status (may take 1-2 minutes):
```bash
kubectl get certificate
```

You should see `READY = True`. If not, describe it for details:
```bash
kubectl describe certificate frontend-tls
```

Check the underlying cert-manager challenge:
```bash
kubectl get challenges
kubectl describe challenge
```

---

## Step 5: Verify the Ingress

```bash
kubectl get ingress frontend-ingress
```

Once `ADDRESS` is populated, open `https://quntamvector.in` in your browser.

---

## Troubleshooting

**Certificate stuck in pending:**
```bash
# Check ClusterIssuer is ready
kubectl get clusterissuer letsencrypt-prod

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check events on the certificate
kubectl describe certificate frontend-tls
```

**Ingress not routing traffic:**
```bash
# Describe the ingress
kubectl describe ingress frontend-ingress

# Check NGINX controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

**Check all resources at once:**
```bash
kubectl get pods,svc,ingress,certificate,clusterissuer
```

---

## Remove Old LoadBalancer (if previously applied)

If `frontend-external` LoadBalancer was previously deployed, delete it to stop cloud charges:
```bash
kubectl delete service frontend-external
```

---

## Networking Overview

| Resource | Type | Port | Notes |
|---|---|---|---|
| `frontend` | ClusterIP | 80 | Internal only, backed by Ingress |
| All other services | ClusterIP | varies | Internal pod-to-pod only |
| NGINX Ingress Controller | LoadBalancer | 80, 443 | Single cluster entry point |
| `frontend-tls` | Certificate | — | Auto-issued by cert-manager (Let's Encrypt) |
| `letsencrypt-prod` | ClusterIssuer | — | ACME HTTP-01 challenge via NGINX |
