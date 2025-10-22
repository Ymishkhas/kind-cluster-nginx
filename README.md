# Kubernetes Monitoring with Prometheus & Grafana on Kind cluster

This project demonstrates how to deploy the Prometheus monitoring stack on a Kind cluster using Helm, and how to monitor an NGINX Deployment with custom metrics, Grafana dashboards, and alerting rules.

## Features

- Prometheus + Grafana via Helm (kube-prometheus-stack)
- Custom values for prometheus configured via values.yaml
- NGINX Deployment with Prometheus exporter sidecar
- ServiceMonitor for scraping NGINX metrics
- Grafana dashboard for live visualization
- PrometheusRules for alerts (CPU, restarts, availability, NGINX 5xx errors)
- SOPS encryption of secret values
- Argo CD GitOps workflow

## Prequisit

Must install 
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) 
- [helm](https://helm.sh/docs/intro/install/) 
- [helmfile](https://github.com/helmfile/helmfile)
- [SOPS](https://github.com/getsops/sops) **and** [age](https://github.com/FiloSottile/age) (for SOPS keys)

### Create the cluster:

```bash
kind create cluster --name nginx-cluster --config kind-config.yaml
```

Verify:

``` bash
kubectl cluster-info
kubectl get nodes
```

### Secrets (SOPS)
This repo keeps secret values **encrypted** with SOPS. You’ll need access to the Age private key.

Set the env variable to point to the private key:
```bash
$env:SOPS_AGE_KEY_FILE="path/to/key.txt"
```

Create the kubernetes secret holding the exported private key for ArgoCD to access
```bash
kubectl create namespace argocd
kubectl -n argocd create secret generic helm-secrets-private-keys --from-file=key.txt=path/to/key.txt
```

## Setup Instructions

### 1. Apply argocd via Helmfile 

```bash
helmfile apply
```

This will apply and create:
- ArgoCD
- Configure helm-secrets with age for ArgoCD

Apply argocd root app
```bash
kubectl apply -f gitops/root-app.yaml
```

After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. Port-forward Istio ingress (dev)

```bash
kubectl -n istio-system port-forward svc/istio-gateway 80:80
```

Istio Ingress Gateway will allow access to:
- ArgoCD UI --> [argocd.localhost](argocd.localhost)
- The NGINX service --> [yousef.localhost](yousef.localhost)
- Grafana service --> [grafana.localhost](grafana.localhost)
- prometheus-operated service --> [prom.localhost](prom.localhost)


## Monitoring

### 1. Verify Metrics Flow

Open http://prom.localhost/targets → ensure nginx-exporter targets are UP.

Test queries:
`up{job="nginx-exporter"}`
`nginx_connections_active`
`rate(nginx_connections_accepted[1m])`

### 2. Visualize in Grafana

- URL: http://grafana.localhost
- Credentials: from the secret (admin-user / admin-password).

Run to see Decrypted values:
```bash
sops infra/monitoring/secrets.yaml
```

### 3. Check Alerts

Check in Prometheus → Status → Rules / Alerts.

## Clean up

``` bash
helmfile destroy
kind delete cluster --name nginx-cluster
```
