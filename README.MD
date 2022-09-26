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
Modify loki-values.yaml
```
loki:
  enabled: true
  isDefault: true
  url: http://{{(include "loki.serviceName" .)}}:{{ .Values.loki.service.port }}
  config:
    schema_config:
      configs:
        - from: 2022-01-11
          store: boltdb-shipper
          object_store: aws
          schema: v12
          index:
            period: 24h
            prefix: index_
    storage_config:
      aws:
        s3: s3://region/bucket-name #Change Region and Bucket Name
        region: region # Change Region
      boltdb_shipper:
        active_index_directory: /data/loki/boltdb-shipper-active
        cache_location: /data/loki/boltdb-shipper-cache
        cache_ttl: 24h
        shared_store: s3
      filesystem:
        directory: /data/loki/chunks
          
  readinessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  livenessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  datasource:
    jsonData: {}
    uid: ""


promtail:
  enabled: true
  config:
    logLevel: info
    serverPort: 3101
    clients:
      - url: http://{{ .Release.Name }}:3100/loki/api/v1/push

```
```
helm show values grafana/loki-stack > helm-loki-values.yaml #Optional
helm install loki grafana/loki-stack --values loki-values.yaml -n grafana
```
## Grafana Prometheus
Modify grafana-values.yaml
Append the below config in line 831
```
helm show values prometheus-community/kube-prometheus-stack > grafana-values.yaml
```
```
  additionalDataSources: 
    - name: Loki
      type: loki
      url: http://loki.grafana.svc.cluster.local:3100 # Change here
      access: proxy
      jsonData:
          derivedFields:
            - datasourceUid: tempo
              matcherRegex: ((\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+)(\d+|[a-z]+))
              name: TraceID
              url: $${__value.raw}
          maxLines: 1000
      version: 1
    - name: Tempo
      type: tempo
      uid: tempo
      url: http://tempo.grafana.svc.cluster.local:3100 # Change here
      access: proxy
      version: 1
    - name: prometheus-sample
      access: proxy
      basicAuth: true
      basicAuthPassword: pass
      basicAuthUser: daco
      editable: false
      jsonData:
          tlsSkipVerify: true
      orgId: 1
      type: prometheus
      url: https://{{ printf "%s-prometheus.svc" .Release.Name }}:9090
      version: 1
```

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
