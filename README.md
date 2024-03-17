# Progressive Delivery with Istio and Gateway API plugin for Argo Rollouts - Single cluster

As presented at Cloud Native Rejekts in Paris in March 2024. With multicluster support!

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
civo ip reserve -n kube1
export IP1=$(civo ip list -o custom -f address,name | grep kube1 | cut -f1 -d",")
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
export CTX_CLUSTER1=rejekts1
```

### (Optional) Enable Hubble UI

```bash
# needs the cilum cli
cilium --context="${CTX_CLUSTER1}"  hubble enable --ui
```

### Install Gateway API CRDs

```bash
kubectl --context="${CTX_CLUSTER1}" apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

### Install Istio

Single cluster setup:

```bash
envsubst < istio-operator/cluster1.yaml| istioctl install --context="${CTX_CLUSTER1}" -y -f -
```

### Install monitoring stack

```bash
helm --kube-context="${CTX_CLUSTER1}" upgrade -i prom kube-prometheus-stack \
  --create-namespace -n prometheus \
  --repo https://prometheus-community.github.io/helm-charts \
  --values monitoring/prom-grafana-values.yaml
```

Note in the `prom-grafana-values.yaml` how the Argo Rollout dashboard is added from a url. Now add some Istio dashboards and scraping targets:

```bash
kubectl apply -f monitoring/istio-grafana-dashboards.yaml
kubectl apply -f monitoring/service-pod-monitor.yaml
```

### (Optional) Install ArgoCD

This will be useful in the future if you want to integrate ArgoCD with Argo Rollouts with the [rollout extension](https://github.com/argoproj-labs/rollout-extension?tab=readme-ov-file).

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade -i argocd argo/argo-cd \
  --namespace argocd --create-namespace
```

### Install Argo Rollouts

```bash
helm upgrade -i argo-rollouts argo/argo-rollouts \
  --create-namespace -n argo-rollouts \
  --values rollout/helm/values.yaml
```

> [!WARNING]
> Unfortunately some extra permissions are needed for the plugin to work on HTTProutes objects:

```bash
kubectl apply --context ${CTX_CLUSTER1} rollout/rbac/0-clusterrole.yaml
```

### Install Argo Rollouts Gateway API Plugin

```bash
kubectl apply -f rollout/plugins/gatewayapi-rollout-plugin-cm.yaml
kubectl rollout restart deployment -n argo-rollouts argo-rollouts
```

### Apply the ingresses (Gateway and HTTPRoutes)

```bash
kubectl --context ${CTX_CLUSTER1} apply -f ingress
```

> [!WARNING]
> Argo Rollouts dashboard doesn't work thru the Istio ingress gateway (the infamous `You need to enable JavaScript to run this app.` error). It's anyway better to instead use it only via port-forwarding, as you can actually can trigger (and we will show it) releases thru the UI (it can be disabled).



### Rollout progressive delivery: Canary



Check the result:

```bash
while :; do curl http://app.rejekts.kubespaces.io/callme; sleep 1; done
```

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
