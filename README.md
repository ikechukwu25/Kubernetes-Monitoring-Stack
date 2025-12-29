# Kubernetes Monitoring Stack (Kubernetes + Prometheus + Grafana)

This repository documents the **end-to-end setup of a Kubernetes cluster on RHEL 9 using kubeadm**, followed by deployment of a **production-grade monitoring stack (Prometheus, Grafana, Alertmanager)** using **Helm**.

This project is designed as:

* A **hands-on infrastructure lab**
* A **repeatable reference** for on‑prem / VM‑based Kubernetes
* A **portfolio-ready DevOps / SysAdmin project**

---

## Technologies Used

* **OS**: Red Hat Enterprise Linux 9
* **Kubernetes**: kubeadm (v1.34)
* **Container Runtime**: containerd (systemd cgroups)
* **CNI**: Calico
* **Monitoring**: kube‑prometheus‑stack
* **Package Manager**: Helm

---

## Cluster Architecture

* 1 × Control Plane node
* 2 × Worker nodes

Monitoring stack deployed into a dedicated namespace:

* Prometheus
* Grafana
* Alertmanager
* Node Exporter
* kube‑state‑metrics

---

## Repository Structure

```text
.
├── README.md
├── kubernetes/
│   ├── kubeadm-installation.md
│   └── calico-networking.md
├── monitoring/
│   ├── prometheus-grafana.md
│   └── dashboards.md
└── screenshots/
    └── README.md
```

---

## Setup Flow

1. Prepare RHEL 9 nodes and install containerd
2. Install Kubernetes components (kubeadm, kubelet, kubectl)
3. Initialize the Kubernetes control plane
4. Deploy Calico CNI networking
5. Join worker nodes
6. Verify cluster health
7. Deploy Prometheus & Grafana using Helm
8. Visualize cluster metrics in Grafana

Each step is documented with **verification commands** and **screenshots**.

---

## Verification Screenshots

Screenshots are stored in the `screenshots/` directory and include:

* Kubernetes nodes ready
* Calico networking running
* Monitoring stack pods
* Grafana UI access
* Kubernetes dashboards

---

## Why This Project

This project intentionally avoids managed Kubernetes services to demonstrate:

* Deep understanding of Kubernetes internals
* Manual cluster bootstrapping and troubleshooting
* Enterprise‑aligned monitoring practices
* Reproducible infrastructure documentation

---

## Next Improvements

* Ingress for Grafana
* Persistent storage for Prometheus
* Alertmanager notification routing
* Offline / air‑gapped image registry support

---

## Author

Built as a hands‑on Kubernetes and observability project for learning, reference, and professional portfolio use.
