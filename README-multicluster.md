# Progressive Delivery with Istio and Gateway API plugin for Argo Rollouts

As presented at Cloud Native Rejekts in Paris in March 2024. With multicluster support!

## Prerequisite

- **Two** kubernetes cluster (we're using the great [Civo](https://www.civo.com) Kubernetes service)

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
civo ip reserve -n kube1
civo ip reserve -n kube2

export IP1=$(civo ip list -o custom -f address,name | grep kube1 | cut -f1 -d",")
export IP2=$(civo ip list -o custom -f address,name | grep kube2 | cut -f1 -d",")
```

Now go and set up a DNS record (I use `*.rejekts.kubespaces.io`) with your favorite DNS provider (I use Azure DNS). Something along the lines:

```bash
az network dns record-set a add-record \
    --resource-group dns \
    --zone-name kubespaces.io \
    --record-set-name "*.rejekts" \
    --ipv4-address $IP1
```

### (Optional) Cluster creation

```bash
civo network create cilium
civo kubernetes create -p cilium -r traefik2-nodeport -v 1.29.2-k3s1 --merge --save --switch --wait rejekts1
civo kubernetes create -p cilium -r traefik2-nodeport -v 1.29.2-k3s1 --merge --save --switch --wait rejekts2
export CTX_CLUSTER1=rejekts1
export CTX_CLUSTER2=rejekts2
```

### (Optional) Enable Hubble UI

```bash
# needs the cilum cli
cilium --context="${CTX_CLUSTER1}"  hubble enable --ui
cilium --context="${CTX_CLUSTER2}"  hubble enable --ui
```

### Install Gateway API CRDs

```bash
kubectl --context="${CTX_CLUSTER1}" apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
kubectl --context="${CTX_CLUSTER2}" apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

### Install Istio

Multicluster, [multi-primary istio](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/) setup is done as follows:

For cluster 1:

```bash
envsubst < istio-operator/cluster1.yaml| istioctl install --context="${CTX_CLUSTER1}" -y -f -
```

For cluster 2:

```bash
envsubst < istio-operator/cluster2.yaml| istioctl install --context="${CTX_CLUSTER2}" -y -f -
```

You can determine the private IPs attached to the east/west gateays with the Civo APIs:

```bash
civo loadbalancer show --fields private_ip -o custom ${CTX_CLUSTER1}-istio-system-istio-ewgw
civo loadbalancer show --fields private_ip -o custom ${CTX_CLUSTER2}-istio-system-istio-ewgw
```

### Install monitoring stack

```bash
helm --kube-context="${CTX_CLUSTER1}" upgrade -i prom kube-prometheus-stack \
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
  --values rollout/helm/values.yaml
```

Unfortunately some extra permissions are needed for the plugin to work on HTTProutes objects:

```bash
kubectl apply --context ${CTX_CLUSTER1} rollout/0-clusterrole.yaml
```

### Install Argo Rollouts Gateway API Plugin

```bash
kubectl --context ${CTX_CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config # must be so name
  namespace: argo-rollouts # must be in this namespace
data:
  trafficRouterPlugins: |-
    - name: "argoproj-labs/gatewayAPI"
      location: "https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.2.0/gateway-api-plugin-linux-amd64"
EOF
kubectl rollout restart deployment -n argo-rollouts argo-rollouts
```

### Apply the ingresses (Gateway and HTTPRoutes)

```bash
kubectl --context ${CTX_CLUSTER1} apply -f ingress
```

### Rollout progressive delivery

You can keep an eye on the HTTPRoute weights:

```bash
watch -n 0.5 "kubectl get httproutes.gateway.networking.k8s.io argo-rollouts-http-route -o json | jq '.spec.rules[].backendRefs'"
```

### Clean up

```bash
civo kubernetes delete rejekts -y
```

### Notes

- We're using the Civo controller manager feature to use a pre-created IP address ([ref](https://github.com/civo/civo-cloud-controller-manager))
- We use Gateway API in [manual deployment mode](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment) to attach the `Gateway` resource to an existing `ingressGateway`
- We set Grafana to no auth access for simplicity
