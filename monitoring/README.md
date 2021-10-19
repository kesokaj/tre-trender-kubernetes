````
## Setup repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

## Create namespace
kubectl create ns monitoring-system

## Install
helm install loki grafana/loki -n monitoring-system -f ./loki/values.yaml
helm install promtail grafana/promtail -n monitoring-system -f ./promtail/values.yaml
helm install prometheus prometheus-community/prometheus -n monitoring-system -f ./prometheus/values.yaml
helm install grafana grafana/grafana -n monitoring-system -f ./grafana/values.yaml

````
