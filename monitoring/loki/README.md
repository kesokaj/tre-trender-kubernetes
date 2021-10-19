````
## Add helm repo
helm repo add grafana https://grafana.github.io/helm-charts

## Install
helm install loki grafana/loki -n monitoring-system -f values.yaml

## Upgrade
helm repo update
helm upgrade --install loki grafana/loki -n monitoring-system -f values.yaml
````
