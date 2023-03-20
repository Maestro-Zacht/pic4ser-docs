# INSTALLATION <!-- omit in toc -->

## Contents <!-- omit in toc -->

- [Installation](#installation)
  - [Setup nodes](#setup-nodes)
  - [Install Kubernetes](#install-kubernetes)
  - [Network plugin](#network-plugin)
- [References](#references)

## Installation

This section will show how kubernetes has been deployed at the PIC4SeR centre.

### Setup nodes

The following steps are done on every node.

- Login to the nodes with superuser privileges: `sudo su`
- Turn off swap

  ```bash
  swapoff -a
  sed -i '/ swap / s/^/#/' /etc/fstab
  ```

- Check that required ports are avaiable

  ```bash
  nc -v 127.0.0.1 6443
  ```

- Activate required components for container runtime and networking

  ```bash
  cat <<EOF | tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF

  modprobe overlay
  modprobe br_netfilter

  cat <<EOF | tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF

  sysctl --system
  ```

- Install dependencies

  ```bash
  apt update && apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates lsb-release
  ```

- Setup Docker DEB repo

  ```bash
  mkdir -m 0755 -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
  ```

- Install containerd

  ```bash
  apt update -y && apt install -y containerd.io
  systemctl daemon-reload
  systemctl restart containerd
  systemctl enable --now containerd
  ```

- Setup Kubernetes repo

  ```bash
  curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

  echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
  ```

- Install Kubernetes packages

  ```bash
  apt update
  apt install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl
  ```

### Install Kubernetes

Now the cluster needs to be created. A first master endpoint needs to be chosen, in our case the machine called Bonnie.
On this master node run the init command:

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint="bonnie.polito.it" --upload-certs
```

At the end of the output there are 2 other commands that need to be copy-pasted: one is for making the `kubectl` command avaiable for the user and one (actually two) for joining other nodes. The first is run on the master to make the command available for root user. It looks like this:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

About the join command, there is one for joining master nodes and one for joining worker nodes. The machine Kitt has been chosen for being the second master node, thus the first one has been used.

These join commands use a certificate that expires after 2 hours. If you need to generate another join command, use the following:

- Worker join command:

  ```bash
  kubeadm token create --print-join-command
  ```

- Master join command:

  ```bash
  kubeadm init phase upload-certs --upload-certs
  kubeadm token create --print-join-command
  ```

  There is a part which needs to be added to the command printed out by the token create: `--control-plane --certificate-key <certificate>` where `<certificate>` is the hash provided by the first kubeadm command.

### Network plugin

Now the cluster is running but it's not fully working. With the command `kubectl get nodes` the nodes are marked as NotReady and with `kubectl get pods -n kube-system` is clear that some pods are not running. The reason is that there is no network plugin installed.
In order to clearly see that this is the case, another command can be run: `kubectl describe node <node>` where `<node>` is the name of any node. The output is quite long but there is a `Conditions` section which has a `NetworkUnavailable` type and its status is `True`.

To install Flannel network plugin run the following command:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Now the cluster is set up and it can verified with the commands listed above.

## References

- [NERC Project](https://nerc-project.github.io/nerc-docs/other-tools/kubernetes/kubeadm/single-master-clusters-with-kubeadm/)
- [Kubernetes docs](https://kubernetes.io/docs/home/)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [Helm docs](https://helm.sh/docs/)
- [NVIDIA integration](https://github.com/NVIDIA/k8s-device-plugin)
- [NFS Ubuntu](https://ubuntu.com/server/docs/service-nfs)
- [NFS Kubernetes integration](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/charts/nfs-subdir-external-provisioner/README.md)
- [Tensorflow training operator](https://github.com/kubeflow/training-operator)
- [Multi-worker training](https://www.tensorflow.org/tutorials/distribute/multi_worker_with_keras)
