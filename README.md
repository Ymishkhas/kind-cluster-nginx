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

Must install [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) [helm](https://helm.sh/docs/intro/install/) [helmfile](https://github.com/helmfile/helmfile)

### Create the cluster:

```bash
kind create cluster --name nginx-cluster --config kind/kind-config.yaml
```

Verify:

``` bash
kubectl cluster-info
kubectl get nodes
```

## Setup Instructions

### 1. Install Prometheus Stack & Istio Ingress Gateway via helmfile

```bash
helmfile -e dev apply
```

> [!NOTE] helmfile apply - Error about CRDs
> It's Helmfile and helm-diff's problem where it checks if CRDs installed but errors out since its part of the stack. The solution is in `disableValidation: true` flag in helmfile.yaml. Please uncomment this line for the first instalment, after that you can comment it back. For reference of [Github issue](https://github.com/roboll/helmfile/issues/1353)

### 2. Deploy NGINX with Exporter

```bash
kubectl apply -f nginx/
```

### 3. Expose NGINX via Istio Ingress (Dev)

Istio Ingress Gateway allow to access the NGINX service through a custom domain (yousef.localhost).

Apply the Gateway and VirtualService:
```bash
kubectl apply -f istio/traffic/
```

For local development:
Port-forward the Istio ingress gateway:
```bash
kubectl -n istio-ingress port-forward svc/istio-ingress 80:80
```


Visit http://yousef.localhost in your browser.
You should see the NGINX welcome page.

> For development on Kind, the Istio ingress Service type is ClusterIP.
> In production, this would typically be a LoadBalancer fronted by DNS.

## Monitoring

### 1. Verify Metrics Flow

```bash
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
```

Open http://localhost:9090/targets → ensure nginx-exporter targets are UP.

Test queries:
`up{job="nginx-exporter"}`
`nginx_connections_active`
`rate(nginx_connections_accepted[1m])`

### 2. Visualize in Grafana

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 
```

- URL: http://localhost:3000
- Default login: admin / prom-operator
- Import Grafana dashboard ID 12708 (NGINX exporter(community)).

### 3. Enable Alerts

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