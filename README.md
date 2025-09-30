# Kubernetes Monitoring with Prometheus & Grafana on Kind cluster

This project demonstrates how to deploy the Prometheus monitoring stack on a Kind cluster using Helm, and how to monitor an NGINX Deployment with custom metrics, Grafana dashboards, and alerting rules.

## Features

- Prometheus + Grafana via Helm (kube-prometheus-stack)
- Custom values for prometheus configured via values.yaml
- NGINX Deployment with Prometheus exporter sidecar
- ServiceMonitor for scraping NGINX metrics
- Grafana dashboard for live visualization
- PrometheusRules for alerts (CPU, restarts, availability, NGINX 5xx errors)

## Prequisit

Must install [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/), [helm](https://helm.sh/docs/intro/install/)

## Setup Instructions

### 1. Create the cluster:

```bash
kind create cluster --name nginx-cluster --config kind/kind-config.yaml
```

Verify:

``` bash
kubectl cluster-info
kubectl get nodes
```

### 2. Install Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f helm/prometheus/values.yaml --version 77.12.0
```

### 3. Deploy NGINX with Exporter

```bash
kubectl apply -f nginx/
kubectl port-forward svc/nginx-service 8080:80
```

Can access from http://localhost:8080

### 4. Verify Metrics Flow

```bash
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
```

Open http://localhost:9090/targets → ensure nginx-exporter targets are UP.

Test queries:
`up{job="nginx-exporter"}`
`nginx_connections_active`
`rate(nginx_connections_accepted[1m])`

### 4. Visualize in Grafana

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 
```

- URL: http://localhost:3000
- Default login: admin / prom-operator
- Import Grafana dashboard ID 12708 (NGINX exporter(community)).

### 5. Enable Alerts

``` bash
kubectl apply -f prometheus-rules/basic.rules.yaml
```
Check in Prometheus → Status → Rules / Alerts.


## Clean up

``` bash
kubectl delete -f nginx/
kubectl delete -f prometheus-rules/basic.rules.yaml
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
kind delete cluster --name nginx-cluster
```