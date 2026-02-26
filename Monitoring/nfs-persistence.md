# NFS Persistence (Grafana + Prometheus) — Kubernetes Monitoring Stack

This document explains how I added **NFS-backed persistence** to the monitoring stack in this repository (kube-prometheus-stack: Prometheus, Grafana, Alertmanager) running on a kubeadm-built Kubernetes cluster (RHEL 9). :contentReference[oaicite:0]{index=0}

> [ image ]  
> *(Screenshot idea: Grafana + Prometheus running in the `monitoring` namespace)*

---

## Why NFS persistence?

By default, Grafana and Prometheus can store data on node-local storage (ephemeral). That means:
- **Grafana dashboards/users/plugins** can be lost if the pod is recreated.
- **Prometheus TSDB metrics** can be lost if the pod is recreated.
- Node disks can fill up unexpectedly.

With NFS persistence:
- Grafana uses a persistent directory mounted at `/var/lib/grafana`
- Prometheus uses a persistent directory mounted at `/prometheus`
- Storage is moved off worker-node disks and centralized on the NFS server

---

## My Environment (Reference)

- Kubernetes: kubeadm cluster (1 master, 2 workers)
- Namespace: `monitoring`
- NFS Server: `10.100.25.93`
- Base export: `/nfs/k8s`
- Separated subpaths:
  - Grafana: `/nfs/k8s/grafana`
  - Prometheus: `/nfs/k8s/prometheus`

> [ image ]  
> *(Screenshot idea: cluster nodes + NFS server network diagram)*

---

## Prerequisites

### On Kubernetes nodes (master + workers)
- Ensure NFS client utilities are installed (RHEL):
  - `nfs-utils`
- Ensure the nodes can reach the NFS server:
  - TCP/UDP 2049 (NFS)
  - If using rpcbind/mountd depending on NFS config, ensure related ports are allowed (most NFSv4 setups mainly need 2049)

Quick checks:
- `ping 10.100.25.93`
- `nc -zv 10.100.25.93 2049`

> [ image ]  
> *(Screenshot idea: successful connectivity test from worker to NFS server)*

---

## Step 1 — Configure the NFS server (10.100.25.93)

### 1. Create directories for Grafana and Prometheus
```
sudo mkdir -p /nfs/k8s/grafana
sudo mkdir -p /nfs/k8s/prometheus
