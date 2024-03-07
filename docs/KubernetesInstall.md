## Kubernetes installation scripts

## Setup loadbalancer
#### HaProxy
```bash

###################################################################
frontend Kubernetes
        bind *:6443
        mode tcp
        default_backend Kubernetes

backend Kubernetes
       description "Kubernetes Cluster"
       balance roundrobin
       mode tcp
       server K8s1 master1_ip:6443 check inter 10s fall 3 rise 2
       server K8s2 master2_ip:6443 check inter 10s fall 3 rise 2
       server K8s3 master3_ip:6443 check inter 10s fall 3 rise 2

```

# **Thực hiện trên tất cả các VM**
## Install containerd

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
apt-get update
sudo apt install containerd -y
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
```

## Install kubeadm, kubelet, kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
# sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
# echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Init cluster
### Thực hiện trên 1 VM Master
```bash
kubeadm init --control-plane-endpoint=lb_ip:6443 --upload-certs --pod-network-cidr=10.0.0.0/8
```

## Install Helm: [OS Khác](https://helm.sh/docs/intro/install/)
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Install cni
```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.15.1 --namespace kube-system
```

## Add master node
```bash
kubeadm join lb_ip:6443 --token oylvmu.12pwimke0blaaeji --discovery-token-ca-cert-hash sha256:303a791ef0bdaeb3a3b54ca80f8f4831dff6d0bb1c43c664d9102c9ec569ef61 --control-plane --certificate-key 3b4da12cd25d1c1e7a47abcb908c73405c4abd5e542f99692d8f1b9d368d307a
```

## Add worker node
```bash
kubeadm join lb_ip:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

## Get token list
```bash
kubeadm token list
```

## Create token
```bash
kubeadm token create
```

## Discovery token ca cert hash
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Kube Reset

### kubeadm reset
The "reset" command executes the following phases:
```bash
# preflight           Run reset pre-flight checks
# remove-etcd-member  Remove a local etcd member.
# cleanup-node        Run cleanup node.

kubeadm reset [flags]

```

### External etcd clean up
`kubeadm` reset will not delete any etcd data if external etcd is used. This means that if you run `kubeadm init` again using the same etcd endpoints, you will see state from previous clusters.

```bash
etcdctl del "" --prefix
```