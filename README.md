# k3s-edge-setup

Problem definition: 



## Single Node:
### Prerequisites:
* Ubuntu 22.04
* Static IP is not required (DHCP with a static lease to anchor the BGP peer).
* Local user with sudo privileges.

### Install K3s (change the cluster-cidr to a free network on your network)
```
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--flannel-backend=none --cluster-cidr=172.30.0.0/18 --disable-network-policy --disable=traefik" sh -
```

### Install Calico via Operator (sed is only required if you changed the cluster-cidr)
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
sed -i 's/cidr: 192.168.0.0\/16/cidr: 172.30.64.0\/18/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

Wait and Verify the node is operational (confirm the status is Ready):
```
dmcken@k3s001:~$  watch kubectl get nodes
NAME     STATUS   ROLES                  AGE    VERSION
k3s001   Ready    control-plane,master   130m   v1.28.6+k3s2
```

### Install calicoctl :

Use `whereis kubectl` to find where that command is and move it to the same folder if it is not /usr/local/bin

```
curl -L https://github.com/projectcalico/calico/releases/download/v3.27.0/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
sudo mv calicoctl /usr/local/bin/
```

Create `/etc/calico/calicoctl.cfg` with mode 644 and the following contents:
```
cat <<EOF >calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: 'kubernetes'
  kubeconfig: '/etc/rancher/k3s/k3s.yaml'
EOF

chmod 644 calicoctl.cfg
sudo mkdir -p /etc/calico/
sudo mv calicoctl.cfg /etc/calico/
```

Verify calicoctl is working:

```
dmcken@k3s001:~$ calicoctl get nodes 
NAME     
k3s001   
```
If you get any errors these need to be resolved first.

### Setup BGP:

#### Global BGP Config: 
Create a file calico-default-bgp-config.yaml with the following contents:

```
kubectl get nodes -o wide
NAME     STATUS   ROLES                  AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k3s001   Ready    control-plane,master   12h   v1.28.6+k3s2   192.168.1.120   <none>        Ubuntu 22.04.3 LTS   5.15.0-94-generic   containerd://1.7.11-k3s2

calicoctl get nodes -o wide
NAME     ASN       IPV4               IPV6   
k3s001   (64512)   192.168.1.120/24          
```

Setup the global config:
```
cat <<EOF >calico-global-bgp.cfg
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  nodeMeshMaxRestartTime: 120s
  asNumber: 65521
  bindMode: NodeIP
  listenPort: 179
  #serviceClusterIPs:
  #  - cidr: 10.96.0.0/12
  #serviceExternalIPs:
  #  - cidr: 104.244.42.129/32
  #  - cidr: 172.217.3.0/24  
EOF

calicoctl apply -f calico-global-bgp.cfg
```

Notes:
* The NodeIP is the IP of the host itself (Node in Kurbernetes parlance):
  * `kubectl get nodes -o yaml`
  * `calicoctl get nodes -o yaml`

#### BGP Peers:
Create a file global-bgp-peer.yaml with the following contents:
```
cat <<EOF >calico-bgp-upstream-peer.cfg
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peer
spec:
  peerIP: 192.168.1.1
  asNumber: 65530
EOF

calicoctl apply -f calico-bgp-upstream-peer.cfg
```

## Multi-node:
TODO

## Useful commands

* kubectl
  * `kubectl get installations -o yaml`
* calicoctl
  * `calicoctl version`
* Done

## References:
* Installing k3s and calico:
  * https://docs.tigera.io/calico/latest/getting-started/kubernetes/k3s/quickstart
  * https://docs.tigera.io/calico/latest/getting-started/kubernetes/k3s/multi-node-install
* Setting up BGP:
  * https://docs.tigera.io/calico/latest/reference/resources/bgpconfig - general config
  * https://docs.tigera.io/calico/latest/networking/configuring/bgp - Peering
* IP Pools
  * https://devpress.csdn.net/k8s/62fce6a87e66823466190eac.html - Calico VxLAN pool
