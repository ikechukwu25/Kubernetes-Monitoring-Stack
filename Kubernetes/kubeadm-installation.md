# Kubernetes Cluster Installation (kubeadm on RHEL 9)

This document describes the **standard, production-aligned installation of a Kubernetes cluster** using **kubeadm** on **RHEL 9**.

The setup uses:

* **containerd** as the container runtime (systemd cgroups)
* **kubeadm** for cluster bootstrapping
* A **1 control plane + 2 worker nodes** architecture

---

## 1. Prerequisites

* RHEL 9 servers (minimum):

  * 1 × Control Plane node
  * 2 × Worker nodes

* Minimum specs per node:

  * 2 CPU
  * 4 GB RAM

* Internet access (for pulling images)

* User with `sudo` privileges

All steps in this document are executed on **ALL nodes unless stated otherwise**.

---

## 2. Base System Preparation (All Nodes)

### 2.1 Update System

```
sudo dnf update -y
sudo reboot
```

`cat /etc/redhat-release`


---

### 2.2 Disable Swap (Required by Kubernetes)

```
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

📸


`free -h`

---

### 2.3 Load Required Kernel Modules

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Persist module loading:


```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

📸


`lsmod | egrep 'overlay|br_netfilter`


---

### 2.4 Configure Kernel Networking Parameters

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

`sudo sysctl --system`


📸

`sysctl net.ipv4.ip_forward`


---

## 3. Install Container Runtime (containerd)

This guide uses the **RHEL-supported containerd package**.

```
sudo dnf module enable container-tools:4.0 -y
sudo dnf install -y containerd
sudo systemctl enable --now containerd
```

### 3.1 Configure containerd to use systemd cgroups

```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```


Edit `/etc/containerd/config.toml` and ensure the following is set:

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
```

Restart containerd:

`sudo systemctl restart containerd`

📸

`systemctl status containerd`

---

## 4. Install Kubernetes Components (All Nodes)

### 4.1 Add Kubernetes Repository

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF
```

---

### 4.2 Install kubeadm, kubelet, kubectl

```
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

📸

```
kubeadm version
kubectl version --client
```


---

### 4.3 Lock Kubernetes Package Versions


```
sudo dnf install -y python3-dnf-plugin-versionlock
sudo dnf versionlock add kubelet kubeadm kubectl
```


---

## 5. Initialize the Control Plane (Control Plane Node Only)

Pull images ahead of initialization:


`sudo kubeadm config images pull --kubernetes-version 1.34.0`


Initialize the cluster:

```
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=1.34.0
```

---

### 5.1 Configure kubectl Access

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


📸


`kubectl get nodes`


---

## 6. Join Worker Nodes

On the control plane, generate the join command:


`kubeadm token create --print-join-command`


Run the output command on **each worker node**:

```
sudo kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

📸


`kubectl get nodes -o wide`

---

## 7. Verify Cluster Status


`kubectl get nodes`

Expected state:

```
NAME            STATUS   ROLES           VERSION
k8s-master      Ready    control-plane   v1.34.0
k8s-worker-01   Ready    <none>          v1.34.0
k8s-worker-02   Ready    <none>          v1.34.0
```

If nodes are `NotReady`, proceed to install the CNI plugin (Calico) in the next chapter.

---

## 8. Test Basic Workload (Optional)

```
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide
```

Expose service:

```
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
```

📸

`kubectl get pods -o wide`


---

