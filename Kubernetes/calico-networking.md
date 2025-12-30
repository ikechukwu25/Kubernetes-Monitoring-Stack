# Calico Networking (CNI) Installation

This document covers the installation and verification of **Calico** as the **Container Network Interface (CNI)** for the Kubernetes cluster.

Calico is responsible for:

* Pod-to-pod networking
* Pod-to-node communication
* Network policy enforcement

Without a CNI plugin, Kubernetes nodes will remain in a `NotReady` state.

---

## 1. Why Calico

Calico was chosen because it:

* Is production-grade and widely adopted
* Works well on on-prem and VM-based clusters
* Supports NetworkPolicies natively
* Does not require overlays by default (BGP-based)

---

## 2. Prerequisites

* Kubernetes control plane initialized
* `kubectl` configured on the control plane
* Pod network CIDR **must match** kubeadm initialization

This guide assumes the following CIDR was used during `kubeadm init`:

```
--pod-network-cidr=192.168.0.0/16
```

---

## 3. Install Calico (Control Plane Node)

A **version-pinned manifest** is used for reproducibility.

### 3.1 Download Calico Manifest

```
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

(Optional) Review the manifest:

```
less calico.yaml
```

---

### 3.2 Apply the Manifest

```
kubectl apply -f calico.yaml
```



```
kubectl get pods -n kube-system | grep calico
```

---

## 4. Verify Calico Deployment

Check Calico DaemonSet status:

```
kubectl get daemonset -n kube-system calico-node
```

Expected output:

* Desired = Current = Ready


```
kubectl get daemonset -n kube-system
```

---

## 5. Verify Node Readiness

Once Calico is running, nodes should transition to `Ready`.

```
kubectl get nodes
```

Expected state:

```
NAME            STATUS   ROLES           VERSION
k8s-master      Ready    control-plane   v1.34.0
k8s-worker-01   Ready    <none>          v1.34.0
k8s-worker-02   Ready    <none>          v1.34.0
```

```
kubectl get nodes -o wide
```

---

## 6. Validate Pod Networking

Deploy a test workload:

```bash
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide
```

Ensure:

* Pod receives an IP from the Calico CIDR
* Pod is scheduled on any node


```
kubectl get pods -o wide
```

---

## 7. Troubleshooting (If Needed)

Check Calico logs:

```
kubectl logs -n kube-system -l k8s-app=calico-node
```

Check events:

```
kubectl get events -A | tail -20
```

---

