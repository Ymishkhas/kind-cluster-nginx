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
$env:SOPS_AGE_KEY_FILE="path/to/sops-age-key.txt"
```

## Setup Instructions

### 1. Apply everything via Helmfile 

```bash
helmfile -e dev apply
```

This will apply and create:
- Prometheus Stack 
- Istio Ingress Gateway
- Nginx app with exporter
- Istio traffic manifests
- prometheus rules manifests

> **Note (first install only):** If you hit a CRD validation error from helm-diff, temporarily enable disableValidation: true on the monitoring release in helmfile.yaml. After the first successful install, comment it back out.

### 2. Port-forward Istio ingress (dev)

```bash
kubectl -n istio-ingress port-forward svc/istio-ingress 80:80
```

Istio Ingress Gateway will allow access to:
- The NGINX service --> [yousef.localhost](yousef.localhost)
- Grafana service --> [grafana.localhost](grafana.localhost)
- prometheus-operated service --> [prom.localhost](prom.localhost)


> For development on Kind, the Istio ingress Service type is ClusterIP.
> In production, this would typically be a LoadBalancer fronted by DNS.

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
sops infra/monitoring/values.secret.yaml
```

### 3. Check Alerts

Check in Prometheus → Status → Rules / Alerts.

## Clean up

``` bash
helmfile destroy
kubectl delete -f infra/istio/traffic/ --ignore-not-found
kubectl delete -f infra/monitoring/prometheus-rules/ --ignore-not-found
kind delete cluster --name nginx-cluster
```