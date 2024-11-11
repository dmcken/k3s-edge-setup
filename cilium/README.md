# K3S + Cilium

Install on Ubuntu

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy --disable-kube-proxy' sudo sh -
echo -e 'K3S_KUBECONFIG_MODE="644"' | sudo tee -a /etc/systemd/system/k3s.service.env
sudo service k3s restart
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Install cilium cli
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

sudo systemctl status k3s


```bash
cilium install --version 1.16.3 --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" --set bgpControlPlane.enabled=true



--set kubeProxyReplacement=true
--set ipv6.enabled=true --set ipv4.enabled=true
--set k8sServiceHost=172.16.103.110 --set k8sServicePort=6443

```



### Notes
* https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-adverts-service

