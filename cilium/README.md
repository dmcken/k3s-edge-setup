# K3S + Cilium Anycast setup


## Install

Base OS: Ubuntu Server 24.04

### Install k3s

```bash
curl -sfL https://get.k3s.io | sudo sh -s - \
  --flannel-backend=none \
  --disable-kube-proxy \
  --disable-network-policy \
  --disable servicelb \
  --disable traefik \
  --cluster-init

echo -e 'K3S_KUBECONFIG_MODE="644"' | sudo tee -a /etc/systemd/system/k3s.service.env
sudo service k3s restart
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> $HOME/.bashrc
source $HOME/.bashrc
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt update
sudo apt install kubeadm
```
### Install cilium cli

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Install cilium itself

```bash
cilium install --version 1.16.3 --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
    --set bgpControlPlane.enabled=true --set kubeProxyReplacement=true --set ipv6.enabled=true \
    --set k8sServiceHost=127.0.0.1 --set k8sServicePort=6443 --helm-set=operator.replicas=1


cilium status --wait
```

### Confirm all is working

Test that k3s is working (ensure the status is Ready):
```bash
kubectl get nodes -o wide

kubectl get pods --all-namespaces

kubectl api-resources | grep -i cilium
kubectl get pods -n kube-system
```

Turn on BGP control plane:
```bash
cilium config view | grep -i bgp

# Ensure the following
enable-bgp-control-plane                       true

# if not use the following:
cilium config set enable-bgp-control-plane true
```

## Configure

### Common setup

#### BGP

```bash
kubectl apply -f cilium/bgp-peer-config.yml
kubectl apply -f cilium/bgp-cluster-config.yml

kubectl get nodes
# Use the name from the above to replace k3s1
kubectl label nodes k3s1 rack=rack0

# Confirm the nodes you want to BGP peer are now labeled
kubectl get nodes -l rack=rack0
```

Confirm BGP peers:
```bash
cilium bgp peers
```


#### IP Pool

```bash
kubectl apply -f cilium/lb-pool.yml
kubectl get ippools
kubectl describe ippool lb-pool
```


### Deployment + Service

Start deployment

```bash
kubectl apply -f base/bind-deployment.yml

# Confirm running
kubectl get deployment
```

Start service

```bash
kubectl apply -f base/bind-service-loadbalancer.yml

# Confirm running
kubectl get service
```



















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

How to check status

## Notes

* Troubleshooting:
  * Commands:
    * cilium config view
    * kubectl -n kube-system get pods -l k8s-app=cilium
    * kubectl -n kube-system exec cilium-2hq5z -- cilium-dbg status
    * kubectl exec -n kube-system ds/cilium -- cilium-dbg status
    * kubectl exec -n kube-system ds/cilium -- cilium-dbg status --verbose
    *
  * General links:
    * https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-troubleshooting/
    * https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-operation/
    * https://docs.cilium.io/en/stable/network/kubernetes/troubleshooting/
    * https://docs.cilium.io/en/stable/operations/troubleshooting/
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

```bash
dmcken@k3s1:~/k3s-edge-setup/base$ kubectl describe service bind-service-lb
Name:                     bind-service-lb
Namespace:                default
Labels:                   <none>
Annotations:              lbipam.cilium.io/ips: 2.2.2.2
Selector:                 app=bind
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.227.184
IPs:                      10.43.227.184
LoadBalancer Ingress:     2.2.2.2
Port:                     <unset>  53/UDP
TargetPort:               53/UDP
NodePort:                 <unset>  31850/UDP
Endpoints:                10.42.0.174:53
Session Affinity:         None
External Traffic Policy:  Local
HealthCheck NodePort:     30501
Events:
  Type     Reason                   Age   From                   Message
  ----     ------                   ----  ----                   -------
  Normal   EnsuringLoadBalancer     117s  service-controller     Ensuring load balancer
  Warning  UnAvailableLoadBalancer  117s  service-lb-controller  There are no available nodes for LoadBalancer
  Normal   AppliedDaemonSet         117s  service-lb-controller  Applied LoadBalancer DaemonSet kube-system/svclb-bind-service-lb-b5f07482
  Normal   UpdatedLoadBalancer      117s  service-lb-controller  Updated LoadBalancer with new IPs: [2.2.2.2] -> [192.168.1.219]
```


References:
* https://blog.stonegarden.dev/articles/2024/02/bootstrapping-k3s-with-cilium/
  * Used for install without kube-proxy
* https://rx-m.com/kubernetes-loadbalance-service-using-cilium-bgp-control-plane/
  * Used for BGP setup
* Done

