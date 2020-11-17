
# Kubernetes + KubeVirt install

Loosely from [here](https://github.com/christianh814/kubernetes-toolbox#kubernetes-installation), [here](https://thenewstack.io/how-to-install-a-kubernetes-cluster-on-red-hat-centos-8/), [here](https://medium.com/@vinayakpandit9028/https-medium-com-how-to-install-kubernetes-on-centos-8-c33370d612b8), and [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

This will result in a Kubernetes cluster with one control plane and two worker nodes.

To do:

* Use [cri-o](https://github.com/jmutai/k8s-pre-bootstrap/blob/master/roles/kubernetes-bootstrap/tasks/setup_crio.yml), not Docker, in the future

## Environment prep

* Make sure the nodes have DNS entries, either statically or via DHCP

## VM prep

Three VMs, CentOS 8 minimal.
* `kube01` - 2 vCPU, 4GB RAM, 40GB disk
* `kube02` and `kube03` - 4 vCPU, 12GB RAM, 80GB disk
  * Configured for host CPU passthrough

Guest OS prep:

```bash
# apply updates
dnf -y updates

# turn off swap
swapoff -a

# remove swap partition from fstab
vi /etc/fstab

# disable selinux
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# disable firewalld
systemctl disable --now firewalld

# reboot for changes to take effect
reboot
```

## Docker install

Repeat for each node.

```bash
# need to be root
sudo -i

# need wget
dnf -y install wget

# get the docker repo file
cd /etc/yum.repos.d
wget https://download.docker.com/linux/centos/docker-ce.repo

# install docker
dnf -y install docker-ce

# create the daemon.json file
cat << EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# create the systemd dir
mkdir -p /etc/systemd/system/docker.service.d

# reload daemon
systemctl daemon-reload

# start docker
systemctl enable --now docker

# load the br_netfilter kmod
modprobe br_netfilter

# set sysctls
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# load the sysctls
sysctl --system

# add the k8s repo
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# install the k8s tools
dnf -y install kubelet kubeadm kubectl --disableexcludes=kubernetes

# start kubelet
systemctl enable --now kubelet
```

## k8s cluster standup

From the control plane node

```bash
# initialize the control plane
kubeadm init --pod-network-cidr=10.244.0.0/16

# copy the kubeconfig to your directory
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

At the end of the output from the `kubeadmin init` command will be a line with `kubeadm join`.  YOU NEED THIS COMMAND, DON'T LOSE IT!

Next we need to install a CNI, Calico being the default choice.

```bash
# deploy the flannel CNI
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Finally, join the worker nodes.

```bash
# from each of the other nodes, replace the command with the output from your
# kubeadm init command above
kubeadm join 192.168.14.189:6443 --token ckl86q.ksc0menp7231uo65 \
    --discovery-token-ca-cert-hash sha256:f8545e4597089d7cea8f16a1f27b35d5c6731efe0ef5dfea1dde6bd5819105db

# label them as workers
kubectl get nodes --no-headers -l '!node-role.kubernetes.io/master' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
```

## KubeVirt install

From [here](https://kubevirt.io/user-guide/#/installation/installation).

```bash
# create the namespace
kubectl create namespace kubevirt

# choose a version
export VERSION=v0.35.0

# Deploy the KubeVirt operator
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml

# Create the KubeVirt CR (instance deployment request)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml

# wait until all KubeVirt components is up
kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

If desired (and you have infrastructure for it), enable live migration:

```bash
cat << EOF | k create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config
  namespace: kubevirt
  labels:
    kubevirt.io: ""
data:
  feature-gates: "LiveMigration"
EOF
```

## CDI install

From [here](https://github.com/kubevirt/containerized-data-importer#deploy-it).

```bash
export CDI_VERSION=$(curl -s https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -o "v[0-9]\.[0-9]*\.[0-9]*")
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-cr.yaml
```

## NFS provisioner install

From [here](https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client).

```bash
# create the namespace
k apply -f 00-namespace.yaml

# create the service account
k apply -f 01-rbac.yaml

# create the deployment
k apply -f 02-deployment.yaml

# verify the pod comes up after deployment, inspect it if it does not start
k get pod -n nfs-provisioner=

# create the default storage class
k apply -f 03-storageclass.yaml

# create a PVC to test functionality
k apply -f test-claim.yaml
```