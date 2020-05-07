# Setting Up Single Master Kubernetes Cluster using kubeadm

This is an abbreviation of the guide documented [here].

The tool kubeadm and kubernetes in general have specific
networking [requirements].

## On Every Machine (as root)
```bash
#update stuff
apt-get -y update && apt-get -y upgrade

# Install Docker CE
## Set up the repository:
### Install packages to allow apt to use a repository over HTTPS
apt-get install -y \
  apt-transport-https ca-certificates curl software-properties-common gnupg2

### Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

### Add Docker apt repository.
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

## Install Docker CE.
apt-get update && apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker

#IPv4 forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

#let IP tables see bridged traffic
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

#install kubernetes tools
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## On Master

The `kubeadm init` command below will output a command that can be used to join
worker nodes to the cluster.  Save it for later use. Please note that tokens,
and thus the join command, will expire after 24 hours. Note, the master control
plane needs a public IP for certificate generation and 

### As root user
```bash
#initialize
kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=<control plane public IP>
```
then: 
```bash
#setup kubectl config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Or as root user 
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
then:
```bash
#install pod network
kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

## On Worker Nodes
```bash
sudo su -
#output by `kubeadm init` above
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

To add more worker nodes after token expiration, follow these [instructions].

## Kubectl From Client Machines

Install `kubectl` according to your [platform]

```bash
scp root@<control-plane-host>:/etc/kubernetes/admin.conf ~/.kube/config
kubectl get nodes
```

## Install Helm on Client Machine

Install [helm] binary. Then:

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

## Install Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

## Install Image Registry
```bash
helm upgrade register --set service.type=NodePort. --service.nodePort=<port> stable/docker-registry
```
This installs an insecure registry in the cluster available at `<public-node-ip>:<port>`.
To access the registry from your docker daemon add:
```json
{ 
  "insecure-registries":["myregistry.example.com:5000"]
}
```
to /etc/docker/daemon.json and execute `sudo docker service restart`.

[here]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
[requirements]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-the-mac-address-and-product-uuid-are-unique-for-every-node
[instructions]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes
[platform]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[helm]: https://helm.sh/docs/intro/install/
