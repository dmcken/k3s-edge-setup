# K3S + Cilium Anycast setup (Single-Node)

A single node k3s


## Install

Base OS: Ubuntu Server 24.04
* Disk: Entire disk
* Network: <DHCP>
* Hostname: <whatever you want, I used k3s1>
* User: <aything with sudo privileges>
* Post install:
  * `sudo apt update && sudo apt upgrade`
  * `git clone https://github.com/dmcken/k3s-edge-setup.git` in home directory.

### Install k3s & helm

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

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Install Cilium


First install the CLI:
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Install cilium itself:
```bash
cilium install --version 1.16.3 --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
    --set bgpControlPlane.enabled=true --set kubeProxyReplacement=true --set ipv6.enabled=true \
    --set k8sServiceHost=127.0.0.1 --set k8sServicePort=6443 --helm-set=operator.replicas=1

cilium status --wait
```

The above will cycle a few times before exiting on its own (hopefully with zero errors). Wait for it to do so before continuing.

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

```bash
kubectl apply -f base/
kubectl apply -f cilium-single/lb-pool.yml
```

```bash
kubectl get deployments
# Output
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
bind-deployment   3/3     3            3           22s
```

```bash
kubectl get service
# Output
NAME                TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
bind-service-lb     LoadBalancer   10.43.68.80    2.2.2.2       53:30534/UDP   2m34s
bind-service-node   NodePort       10.43.65.200   <none>        53:30053/UDP   2m34s
kubernetes          ClusterIP      10.43.0.1      <none>        443/TCP        5m12s
```

#### BGP

```bash
kubectl apply -f cilium-single/bgp-peer-config.yml
kubectl apply -f cilium-single/bgp-cluster-config.yml

# The node needs to be labeled with rack=rack0
kubectl get nodes --show-labels=true
# Use the name from the above to replace k3s1
kubectl label nodes k3s1 rack=rack0

# Confirm the nodes you want to BGP peer are now labeled
kubectl get nodes -l rack=rack0
```

Confirm BGP peers:
```bash
cilium bgp peers
```

Turn on service advertisement:
```bash
kubectl apply -f cilium-single/bgp-advertisement-loadbalancer.yml
```

## Maintainence / Monitoring

### Check BGP routes

```bash
cilium bgp routes
```

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

