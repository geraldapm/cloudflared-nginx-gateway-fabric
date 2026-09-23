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
