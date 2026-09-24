# Cloudflare Tunnel with Nginx Gateway Fabrid
Experiment to integrate NGF with Cloudflare Tunnel.

## Prerequisites
- Active Kubernetes Cluster. You can follow the tutorial below to create your desired Kubernetes Distribution
[RKE2](https://github.com/geraldapm/microos-rke2)
[K3S](https://github.com/geraldapm/microos-k3s)
[Vanilla Kubernetes](https://github.com/geraldapm/microos-kubeadm)

- If you are using cilium, ensure that this configuration is applied to be able to reach nodePort services from floating IP -> https://github.com/cilium/cilium/issues/37691#issuecomment-4253175437
```bash
kubectl -n kube-system edit configmap cilium-config
```
```
...omitted
  nodeport-addresses: 192.168.103.0/24
...omitted
```
```bash
kubectl rollout restart ds -n kube-system cilium
```

- Helm binary. Get it from [helm.sh](https://helm.sh)

## Installation - Nginx Gateway Fabric
- Install the Kubernetes Gateway API CRD. Refer to this [Documentation](https://gateway-api.sigs.k8s.io/guides/getting-started/introduction/#installing-gateway-api).
```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
```

- Install the Nginx Gateway Fabric by using this command. Ensure to enable `nginx.service.type=ClusterIP` because we want to route the cloudflare tunnel to the clusterIP.
```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway --set nginx.service.type=ClusterIP
```

## Installation - Cloudflare Tunnel
- Refer to this [tutorial](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/) for further explanations.
- Create the `cloudflared` namespace:
```bash
kubectl create ns cloudflared
```
- Create the cloudflare tunnel based on the tutorial. Ensure that you have the token from the tunnel creation wizard.
tunnel-token.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-token
  namespace: cloudflared
stringData:
  token: <YOUR_TUNNEL_TOKEN>
```

- Apply the manifest
```bash
kubectl apply -f tunnel-token.yaml
kubectl apply -f cloudflared-tunnel.yaml
```

## Implementation: Create the Gateway
- Apply the sample application
```bash
kubectl create ns httpbin
kubectl apply -f httpbin-deployment.yaml -n httpbin
```

- Apply the following manifests for creating the gateway
```bash
kubectl apply -f nginx-proxy-k8s.yaml
kubectl apply -f k8s-gateway.yaml
kubectl apply -f httpbin-route.yaml
```

## Implementation: Add the Cloudflare tunnel route
- Refer on this [tutorial](https://community.cloudflare.com/t/wildcard-subdomains/501612) for details. In this case, `*.k8s.gpm.my.id` is pointed into `http://k8s-gateway-nginx.nginx-gateway.svc.cluster.local`


## Extra: Add Cert Manager and Private CA Certs
- Ensure that your internet can query other DNS servers. For example in my environment, access to other DNS servers are restricted. So I have to change the nameservers to the provider DNS server. Semarkampret memang!
```bash
--set extraArgs='{--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=10.79.226.107:53:53}
```
- Install Cert Manager:
```bash
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.2 \
  --namespace cert-manager \
  --create-namespace \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true \
  --set extraArgs='{--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=1.1.1.1:53\,8.8.8.8:53}' \
  --set crds.enabled=true
```

- Create Cloudflare DNS API token for editing DNS zones (ACME Challenge)
```bash
kubectl create secret -n cert-manager generic dns-api-token --from-literal=token=<API TOKEN>
```

- Apply Cert Manager Cluster Issuer
```bash
kubectl apply -f letsencrypt-prod.yaml
```

- Create the secure-gateway Gateway Resource:
```bash
kubectl apply -f secure-gateway.yaml
```

- Then finally create the httpbin route for secure gateway:
```bash
kubectl apply -f secure-route.yaml
```