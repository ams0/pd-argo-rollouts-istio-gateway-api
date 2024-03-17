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
- [ ] (Optional) Deploy a sample httpbin app
- [ ] Apply the ingresses (Gateway and HTTPRoutes)


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
kubectl apply --context ${CTX_CLUSTER1} -f rollout/rbac/0-clusterrole.yaml
```

### Install Argo Rollouts Gateway API Plugin

```bash
kubectl apply -f rollout/plugins/gatewayapi-rollout-plugin-cm.yaml
kubectl rollout restart deployment -n argo-rollouts argo-rollouts
```

### (Optional) Deploy a sample httpbin app

```bash
kubectl apply -f httpbin/
```

### Apply the ingresses (Gateway and HTTPRoutes)

```bash
kubectl --context ${CTX_CLUSTER1} apply -f ingress
```

> [!WARNING]
> Argo Rollouts dashboard doesn't work thru the Istio ingress gateway (the infamous `You need to enable JavaScript to run this app.` error). It's anyway better to instead use it only via port-forwarding, as you can actually can trigger (and we will show it) releases thru the UI (it can be disabled).


### Access the apps thru the Gateway API-enabled istio ingress

You can reach the monitoring stack and the httpbin app via:

```bash
open http://grafana.rejekts.kubespaces.io
open http://hubble.rejekts.kubespaces.io
open http://prometheus.rejekts.kubespaces.io
open http://rollouts.rejekts.kubespaces.io #won't work
open http://httpbin.rejekts.kubespaces.io/get
```

### Rollout progressive delivery: Canary

We'll use Rollout as a drop-in replacement for Deployments, so we will just need to deploy the services and HTTProutes:

```bash
kubectl apply -f rollout/canary/1-services.yaml
kubectl apply -f rollout/canary/2-httproute.yaml
```

Lastly, create the Rollout resource:

```bash
kubectl apply -f rollout/canary/3-rollout.yaml
```

Check the pods being created:

```bash
kubectl get po
NAME                             READY   STATUS    RESTARTS   AGE
rollouts-demo-668468789b-957zb   1/1     Running   0          22s
rollouts-demo-668468789b-6crk6   1/1     Running   0          22s
rollouts-demo-668468789b-7pb24   1/1     Running   0          22s
```

If you the Argo Rollout plugin, you can check the status here:

```bash
kubectl argo rollouts get rollout rollouts-demo | tail -7
NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy  4m12s
└──# revision:1
   └──⧉ rollouts-demo-668468789b           ReplicaSet  ✔ Healthy  4m12s  stable
      ├──□ rollouts-demo-668468789b-6crk6  Pod         ✔ Running  4m12s  ready:1/1
      ├──□ rollouts-demo-668468789b-7pb24  Pod         ✔ Running  4m12s  ready:1/1
      ├──□ rollouts-demo-668468789b-957zb  Pod         ✔ Running  4m12s  ready:1/1
      ├──□ rollouts-demo-668468789b-h7nsc  Pod         ✔ Running  5s     ready:1/1
      └──□ rollouts-demo-668468789b-wmtv4  Pod         ✔ Running  5s     ready:1/1
```

Check that we're serving the first revisions (1.0) of our app

```bash
while :; do curl "http://app.rejekts.kubespaces.io/callme"; sleep 1; done
```

Go ahead and change the image version on the dashboard, and observe the version rolling out both as new pods come up and as the HTTProute weights are shifted (and the stable/canary versions are swapped at the end of the process).

You can keep an eye on the HTTPRoute weights:

```bash
watch -n 0.5 "kubectl get httproutes.gateway.networking.k8s.io argo-rollouts-http-route -o json | jq '.spec.rules[].backendRefs'"
```

### More demos (to do)

- [ ] Demonstrate the `workloadRef` feature.
- [ ] Demonstrate the header based routing feature

### Clean up

```bash
civo kubernetes delete rejekts1 -y
```

### Notes

- We're using the Civo controller manager feature to use a pre-created IP address ([ref](https://github.com/civo/civo-cloud-controller-manager))
- We use Gateway API in [manual deployment mode](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment) to attach the `Gateway` resource to an existing `ingressGateway`
- We set Grafana to no auth access for simplicity
