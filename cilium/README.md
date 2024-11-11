# K3S + Cilium Anycast setup


## Install

Base OS: Ubuntu Server

### Install k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy --disable-kube-proxy' sudo sh -
echo -e 'K3S_KUBECONFIG_MODE="644"' | sudo tee -a /etc/systemd/system/k3s.service.env
sudo service k3s restart
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```


Test that k3s is working:
```bash
kubectl get pods
```

### Install cilium cli

```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```



```bash
cilium install --version 1.16.3 --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
    --set bgpControlPlane.enabled=true

# --set kubeProxyReplacement=true
# --set ipv6.enabled=true --set ipv4.enabled=true
# --set k8sServiceHost=172.16.103.110 --set k8sServicePort=6443
```

## Configure

The bulk of this is pulled from:
https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/

1. Edit [BGP Cluster Config](bgp-cluster-config.yml) changing the following.
   1. localASN - Whatever ASN you want your cluster to use.
   2. For each upstream peer:
      1. peerASN: <ASN of upstream router>
      2. peerAddress: <IP Address (v4 or v6) of upstream router>
2. Edit
3. Apply
   1. `kubectl apply -f bgp-cluster-config.yml`
   2. `kubectl apply -f bgp-peer-config.yml`
4. Done


## Maintainence / Monitoring


#### Notes

* Troubleshooting:
  * https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-troubleshooting/
  * https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-operation/
* Enhancements / Changes:
  * --allocate-node-cidrs - https://docs.cilium.io/en/stable/network/kubernetes/requirements/#enable-automatic-node-cidr-allocation-recommended
* Done



```
Failing connectivity tests:
{
  "labels": {
    "direction": "INGRESS",
    "reason": "VLAN traffic disallowed by VLAN filter"
  },
  "name": "cilium_drop_count_total",
  "value": 9
}
```



