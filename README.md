# kubernetes-install
# https://www.weave.works/blog/kubernetes-faq-how-can-i-route-traffic-for-kubernetes-on-bare-metal

#Letting iptables see bridged traffic
#__________________________________________
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# ___________________________________________


# (Install Docker CE)
# Set up the repository:
# Install packages to allow apt to use a repository over HTTPS
apt-get update && sudo apt-get install -y \
  apt-transport-https ca-certificates curl software-properties-common gnupg2

# Add Docker's official GPG key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add the Docker apt repository:
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

# Install Docker CE
apt-get update && sudo apt-get install -y \
  containerd.io=1.2.13-2 \
  docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)

# Set up the Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
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

# Restart Docker
systemctl daemon-reload
systemctl restart docker
systemctl enable docker

# __________________________________________________
# Install kubeadm, kubelet and kubectl [Ref] 

# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Restart Kubelet 
systemctl daemon-reload
systemctl restart kubelet

# To install a specific version of kubeadm, kubelet, kubectl:

# apt-get install -y kubelet=1.19.0 kubeadm=1.19.0 kubectl=1.19.0


# Init Cluster 

# Only on master : 
kubeadm init

# At the end of this execution we'll have a join command line 

________________________________________________________________________________________________________________________________________________________________
Above command will provide 2 important information.

    Details of cluster api-server to be added to .kubeconfig as regular (non-root) user
    Command to be execute in worker nodes (as root user) to allow worker nodes to join the cluster.

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:

Execute as regular non-root user : 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config



Apply the CNI plugin of your choice: Follow these instructions to install the CNI provider. Make sure the configuration corresponds to the Pod CIDR specified in the kubeadm configuration file if applicable.

In this example we are using Weave Net:


Temps: 
sudo kubeadm join 172.31.17.34:6443 --token sm58qu.iayvbtlhm0wl7coa     --discovery-token-ca-cert-hash sha256:41d0986ce379c131cc77c3164d05a71ff43027a6eaee1312a1629a90dc524263



Install LoadBalancer:
https://metallb.universe.tf/installation/

Preparation

If you’re using kube-proxy in IPVS mode, since Kubernetes v1.14.2 you have to enable strict ARP mode.

Note, you don’t need this if you’re using kube-router as service-proxy because it is enabling strict arp by default.


# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system

Setup : 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

Configuration : Please note that for LinuxAcademy Playgrounds you should have a nodeport within the range authorized otherwise you can configure it manually.

https://metallb.universe.tf/configuration/





________________________________________________________________________________________________________

Installing Helm, Prometheus and graphane
https://www.magalix.com/blog/monitoring-of-kubernetes-cluster-through-prometheus-and-grafana


