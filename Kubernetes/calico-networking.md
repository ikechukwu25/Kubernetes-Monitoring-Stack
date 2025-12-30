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


<img width="514" height="89" alt="image" src="https://github.com/user-attachments/assets/12d1f2ee-43fc-4c9e-9c09-6baea9323707" />


---

## 4. Verify Calico Deployment

Check Calico DaemonSet status:

```
kubectl get daemonset -n kube-system calico-node
```



<img width="676" height="54" alt="image" src="https://github.com/user-attachments/assets/5a2303a8-f587-4872-b58d-2b6d7d28c879" />


Expected output:

* Desired = Current = Ready


```
kubectl get daemonset -n kube-system
```



<img width="683" height="73" alt="image" src="https://github.com/user-attachments/assets/3540dc49-45f0-4644-aa5b-8d2a454fe0f2" />


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

```
[root@K8SMaster ~]# kubectl get pods -o wide
NAME                     READY   STATUS   RESTARTS     AGE      IP                NODE         NOMINATED NODE   READINESS GATES
nginx-66686b6766-tlhql   1/1     Running    0          2m6s   192.168.230.197   k8sworker1        <none>           <none>
```

---

## 7. Troubleshooting (If Needed)

Check Calico logs:

```
kubectl logs -n kube-system -l k8s-app=calico-node
```

<img width="1274" height="465" alt="image" src="https://github.com/user-attachments/assets/bbc94d14-e3f5-40b3-bfc5-57a83b03c341" />





Check events:

```
kubectl get events -A | tail -20
```



<img width="1276" height="359" alt="image" src="https://github.com/user-attachments/assets/221f62e7-735c-45ca-9661-6f2351910861" />



---

