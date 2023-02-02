# VTT Demo

## Prerequisite

- A clean k8s cluster (tested only with OVHcloud managed k8s)

## Cluster prerequisite

**Note: This example was run from a OVHcloud k8s managed cluster**

2. Install nginx ingress controller on the cluster

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace --version 4.4.2
```

3. Install cert-manager and cluster issuer

```bash
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace -f cert-manager/values.yaml --version v1.11.0
kubectl apply -f cert-manager/cluster-issuer.yml
```

4. Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack/values.yaml --namespace monitoring --create-namespace --version 44.3.0
```

5. Install Keystone application

```bash
cd keystone
helm upgrade --install keystone .

** Please be patient while the chart is being deployed **

Connect to openstack with the following variables:

export OS_USERNAME=admin
export OS_PASSWORD=******
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=https://keystone.57.128.91.118.xip.opensteak.fr/v3/
export OS_IDENTITY_API_VERSION=3
```

5. You can export the variable created from the previous step and create openstack token:

```bash
pip install python-openstackclient
openstack token issue
```

## Demo time

1. Install grafana tempo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install tempo grafana/tempo --set tempo.searchEnabled=true --set tempo.global_overrides.max_search_bytes_per_trace=0
```

2. Add the grafana datasource: http://tempo.default:3100 (1m)

3. Install opentelemetry operator

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
kubectl get pods -n opentelemetry-operator-system
```

4. Install open-telemetry collector

```bash
kubectl apply -f otel/collector.yaml
```

5. Install open-telemetry instrumentation

```bash
kubectl apply -f otel/instrumentation.yaml
```

6. Edit the Keystone deployment with right annotation

```bash
kubectl patch deployment keystone -p '{"spec": {"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"true","instrumentation.opentelemetry.io/container-names": "keystone"}}}}}'
````

7. Create a token a check the trace

```bash
openstack token issue
```

Go to [https://grafana.57.128.91.118.xip.opensteak.fr/explore](https://grafana.57.128.91.118.xip.opensteak.fr/explore)

## Rollback demo

1. Delete grafana datasource

2. Kubectl patch to remove annotation from keystone deployment

```bash
kubectl patch deployment keystone --type=json -p='[{"op":"remove","path":"/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1inject-python"}]'
kubectl patch deployment keystone --type=json -p='[{"op":"remove","path":"/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1container-names"}]'
```

3. Delete k8s object

```bash
helm uninstall tempo
kubectl delete -f otel/collector.yaml
kubectl delete -f otel/instrumentation.yaml
kubectl delete -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```