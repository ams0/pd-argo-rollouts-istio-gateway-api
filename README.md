# Progressive Delivery with Istio and Gateway API plugin for Argo Rollouts

As presented at Cloud Native Rejekts in Paris in March 2024


## Prerequisite

- A kubernetes cluster (we're using the great [Civo](https://www.civo.com) Kubernetes service)


## Setup

- [ ] (Optional) Install Argo Rollouts kubectl plugin
- [ ] (Optional) Create an IP in Civo and setup DNS
- [ ] (Optional) Create a cluster with Civo
- [ ] Install Gateway API CRDs
- [ ] (Optional) Enable Hubble UI
- [ ] Install Istio
- [ ] Install monitoring stack
- [ ] (Optional) Install ArgoCD
- [ ] Install Argo Rollouts
- [ ] Install Argo Rollouts Gateway API Plugin

### (Optional) Install Argo Rollouts kubectl plugin

```bash
brew install argoproj/tap/kubectl-argo-rollouts
```

### (Optional) Create an IP in Civo and setup DNS

```bash
civo ip reserve -n kube
```

Now go and set up a DNS record (I use `*.rejekts.kubespaces.io`) with your favorite DNS provider (I use Azure DNS). Something along the lines:

```bash
az network dns record-set a add-record \
    --resource-group dns \
    --zone-name kubespaces.io \
    --record-set-name "*.rejekts" \
    --ipv4-address 74.220.28.53
```


### (Optional) Cluster creation

```bash
civo network create cilium
civo kubernetes create -p cilium -r traefik2-nodeport -v 1.29.2-k3s1 --merge --save --switch --wait rejekts
```
### (Optional) Enable Hubble UI

```bash
# needs the cilum cli
cilium hubble enable --ui
```

### Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

### Install Istio

```bash
istioctl install --set meshConfig.accessLogFile=/dev/stdout -y \
 --set "values.gateways.istio-ingressgateway.serviceAnnotations.kubernetes\.civo\.com/ipv4-address=74.220.28.53"

```

### Install monitoring stack

```bash
helm upgrade -i prom kube-prometheus-stack \
  --repo https://prometheus-community.github.io/helm-charts \
  --set grafana.defaultDashboardsEnabled=false \
  --set 'grafana.grafana\.ini.auth\.anonymous.enabled'=true \
  --set 'grafana.grafana\.ini.auth\.anonymous.org_role'=Admin \
  --set 'grafana.grafana\.ini.auth\.anonymous.org_name'="Main Org." --create-namespace   -n prometheus
```

### (Optional) Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```



### Install Argo Rollouts

```bash
helm upgrade -i argo-rollouts argo/argo-rollouts \
  --create-namespace -n argo-rollouts \
  --set dashboard.enabled=true
```

### Install Argo Rollouts Gateway API Plugin

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config # must be so name
  namespace: argo-rollouts # must be in this namespace
data:
  trafficRouterPlugins: |-
    - name: "argoproj-labs/gatewayAPI"
      location: "https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.2.0/gateway-api-plugin-amd64"
EOF
kubectl rollout restart deployment -n argo-rollouts argo-rollouts
```

### Apply the ingresses

```bash
kubectl apply -f ingress
```


### Clean up

```bash
civo kubernetes delete rejekts -y
```


### Notes

- We're using the Civo controller manager feature to use a pre-created IP address ([ref](https://github.com/civo/civo-cloud-controller-manager))
- We use Gateway API in [manual deployment mode](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment) to attach the `Gateway` resource to an existing `ingressGateway`
- We set Grafana to no auth access for simplicity 