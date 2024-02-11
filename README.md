# k3s-edge-setup

Mostly pulled from the following:
* Installing k3s and calico:
  * https://docs.tigera.io/calico/latest/getting-started/kubernetes/k3s/quickstart
  * https://docs.tigera.io/calico/latest/getting-started/kubernetes/k3s/multi-node-install
* Setting up BGP:
  * https://docs.tigera.io/calico/latest/reference/resources/bgpconfig - general config
  * https://docs.tigera.io/calico/latest/networking/configuring/bgp - Peering

## Single Node:
### Install K3s (change the cluster-cidr to a free network on your network)
```
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--flannel-backend=none --cluster-cidr=172.30.0.0/18 --disable-network-policy --disable=traefik" sh -
```

### Install Calico via Operator (sed is only required if you changed the cluster-cidr)
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
sed 's/cidr: 192.168.0.0\/16/cidr: 172.30.0.0\/18/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

Verify the node is operational (confirm the status is Ready):
```
dmcken@k3s001:~$  kubectl get nodes
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
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: 'kubernetes'
  kubeconfig: '/etc/rancher/k3s/k3s.yaml'
```

### Setup BGP:

#### Global BGP Config: 
Create a file calico-default-bgp-config.yaml with the following contents:
```
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  nodeMeshMaxRestartTime: 120s
  asNumber: 63400
  serviceClusterIPs:
    - cidr: 10.96.0.0/12
  serviceExternalIPs:
    - cidr: 104.244.42.129/32
    - cidr: 172.217.3.0/24
  listenPort: 178
  bindMode: NodeIP
```

#### BGP Peers:
Create a file global-bgp-peer.yaml with the following contents:
```
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peer
spec:
  peerIP: 192.168.1.1
  asNumber: 64567
```

## Multi-node:
TODO
