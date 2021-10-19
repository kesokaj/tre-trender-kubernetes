````
## Add helm repo
helm repo add grafana https://grafana.github.io/helm-charts

## Install
helm install promtail grafana/promtail -n monitoring-system -f values.yaml

## Upgrade
helm repo update
helm upgrade --install promtail grafana/promtail -n monitoring-system -f values.yaml
````
