# Kind + Nginx Demo

This repo shows how to run Nginx on a Kind cluster with a Deployment and Service.

## Ptrrequisit

Must install [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

## Files

- `kind-config.yaml` → Kind cluster config (maps NodePort 30007 to localhost since the cluster runs in a container).  
- `config/nginx-deployment.yaml` → Nginx Deployment (3 replicas).  
- `config/nginx-service.yaml` → NodePort Service exposing Nginx on port 30007.

## Usage

1. Create the cluster:

```bash
kind create cluster --name nginx-cluster --config kind-config.yaml
```

Verify:

``` bash
kubectl cluster-info
kubectl get nodes
```

2. Apply Configs:

```bash
kubectl apply -f configs/
```

3. Access Nginx:

```bash
http://localhost:30007
```

4. Verify

To check Kubernetes objects:

```bash
kubectl get deployments 
kubectl get pods
kubectl get svc
```

Delete a pod:

```bash
kubectl delete pod <pod-name>
```

Kubernetes recreates it automatically (self-healing)

## Clean up

``` bash
kubectl delete service nginx-service
kubectl delete deployment nginx-deployment
kind delete cluster --name nginx-cluster
```