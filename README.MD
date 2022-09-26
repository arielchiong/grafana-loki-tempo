# Getting Started
This was tested in AWS EKS using version 1.21

# Services 
| Service        | Description
| ------------- |:-------------:
| Tempo         | distrubuted tracing
| Loki          | log monitoring
| Promtail      | log agent
| Prometheus    |  metric monitoring
| Grafana       | Web ui

# Prerequisites
```
kubectl
helm
```

## Create Namespace
```
kubectl create ns grafana
```
## Add Helm Repo for Grafana Loki Tempo and Prometheus
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

### Update Repo
```
helm repo update
```

## Tempo
```
helm install tempo grafana/tempo
```
## Loki
```
helm show values grafana/loki-stack > helm-loki-values.yaml #Optional
helm install loki grafana/loki-stack --values loki-values.yaml -n grafana
```
## Grafana Prometheus
```
helm install grafana-prome prometheus-community/kube-prometheus-stack -f grafana-values.yaml -n grafana
```
## Expose without ingress
### Expose to localmachine
```
kubectl -n grafana port-forward service/grafana-prome 3000:80
```
### Expose to anywhere 
```
kubectl -n grafana port-forward service/grafana-prome 3000:80 --address='0.0.0.0'
```

## For Ingress
Currently tested using nginx as ingress-controller
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.1/deploy/static/provider/aws/deploy.yaml

doc: https://kubernetes.github.io/ingress-nginx/deploy/#aws
## Version eks used: 1.21
```
kubectl apply -f grafana-ingress.yaml
```
