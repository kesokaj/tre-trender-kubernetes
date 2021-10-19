````
## Add helm repo
helm repo add grafana https://grafana.github.io/helm-charts

## Install
helm install grafana grafana/grafana -n monitoring-system -f ./grafana/values.yaml

## Upgrade
helm repo update
helm upgrade --install grafana grafana/grafana -n monitoring-system -f ./grafana/values.yaml
