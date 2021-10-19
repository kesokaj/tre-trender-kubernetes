````
kubectl create namespace istio-system
helm install istio-base manifests/charts/base -n istio-system
helm install istiod manifests/charts/istio-control/istio-discovery -n istio-system
kubectl apply -f samples/addons/prometheus.yaml -n istio-system
kubectl apply -f samples/addons/grafana.yaml -n istio-system
kubectl apply -f samples/addons/jaeger.yaml -n istio-system
kubectl apply -f samples/addons/kiali.yaml -n istio-system

````
