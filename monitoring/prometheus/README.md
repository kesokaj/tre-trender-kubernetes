````
## Add helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

## Install
helm install prometheus prometheus-community/prometheus -n monitoring -f values.yaml

## Upgrade
helm repo update
helm upgrade --install prometheus prometheus-community/prometheus -n monitoring -f values.yaml
````
