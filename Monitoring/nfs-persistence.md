# NFS Persistence (Grafana + Prometheus) — Kubernetes Monitoring Stack

This document explains how I added **NFS-backed persistence** to the monitoring stack in this repository (kube-prometheus-stack: Prometheus, Grafana, Alertmanager) running on a kubeadm-built Kubernetes cluster (RHEL 9).

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
```

> [ image ]
> *(Screenshot idea: ls -lah /nfs/k8s showing grafana/ and prometheus/)*

## Step 2 — Export the NFS share

### 1. Edit /etc/exports and export the base folder (or export subfolders if preferred).

Example:

```
/nfs/k8s 10.100.24.0/24(rw,sync,no_subtree_check)
```

### 2. Apply exports:
```
sudo exportfs -ra
sudo exportfs -v
```

> [ image ]
> (Screenshot idea: exportfs -v output)


### 3. (Important) Permissions for Prometheus

Prometheus runs as a non-root user in many kube-prometheus-stack deployments. In my case, it ran as:

uid=1000

gid=2000

Prometheus crashed when it couldn’t write to /prometheus with:
```
permission denied while creating /prometheus/queries.active.
```

Fix on NFS server:

```
sudo chown -R 1000:2000 /nfs/k8s/prometheus
sudo chmod -R 775 /nfs/k8s/prometheus
```

Optional (recommended): enforce group permissions on folders:

```
sudo find /nfs/k8s/prometheus -type d -exec chmod 2775 {} \;
sudo find /nfs/k8s/prometheus -type f -exec chmod 0664 {} \;
```

>[ image ]
>(Screenshot idea: ls -ld /nfs/k8s/prometheus showing owner/group and perms)
