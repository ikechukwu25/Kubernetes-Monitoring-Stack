# NFS Persistence (Grafana + Prometheus) — Kubernetes Monitoring Stack

This document explains how NFS-backed persistent storage was implemented for Grafana and Prometheus in the kubeadm-built Kubernetes cluster (RHEL 9)

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
- Prometheus time-series data survives pod restarts.
- Grafana dashboards, users, and configuration are not lost

---

## My Environment (Reference)

- Kubernetes: kubeadm cluster (1 master, 2 workers)
- Namespace: `monitoring`
- Monitoring stack: kube-prometheus-stack (Helm)
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
    - `dnf install nfs-utils`
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

### 2. — Export the NFS share
- Edit /etc/exports and export the base folder (or export subfolders if preferred).

Example:

```
/nfs/k8s 10.100.24.0/24(rw,sync,no_subtree_check)
```

- Apply exports:
```
sudo exportfs -ra
sudo exportfs -v
```

> [ image ]
> (Screenshot idea: exportfs -v output)


### 3. (Important) Permissions for Prometheus
Prometheus runs as a non-root user in many kube-prometheus-stack deployments. In my case, it ran as:

`uid=1000`

`gid=2000`

Prometheus crashed when it couldn’t write to /prometheus with:
```
permission denied while creating /prometheus/queries.active.
```

Fix on the NFS server:

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

## Step 2 — Decide how storage will be provisioned

There are two common approaches:

Dynamic provisioning using an NFS provisioner + StorageClass

Static provisioning (manual PV + PVC) using nfs volumes

For this project, I used static PVs (manual PV definition pointing to NFS paths).

This is simple, predictable, and perfect for a lab / on-prem environment.


---

## StorageClass Design

For consistency and maintainability, both Grafana and Prometheus use the same static StorageClass:

`nfs-manual`

This StorageClass enables static NFS-backed PersistentVolumes without dynamic provisioning.

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

[ image ]
(Screenshot: kubectl get storageclass showing nfs-manual)

This ensures:

- Controlled static binding

- Predictable storage allocation

- Clean separation of workloads

- Enterprise-friendly architecture

## Step 3 — Grafana persistence (PV + PVC)

Grafana stores:

- `grafana.db` (dashboards, users, orgs)

- plugins directory

We persist these by mounting NFS at:

- Pod mountPath: `/var/lib/grafana`

- NFS path: `/nfs/k8s/grafana`

### 3.1 Create Grafana PV

Create `grafana-nfs-pv.yaml`:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-manual
  nfs:
    server: 10.100.25.93
    path: /nfs/k8s/grafana
```

Apply:
```
kubectl apply -f grafana-nfs-pv.yaml
```

### 3.2 Create Grafana PVC

Create `grafana-nfs-pvc.yaml`:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-nfs-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-manual
  volumeName: grafana-nfs-pv
```
Apply:
```
kubectl apply -f grafana-nfs-pvc.yaml
```

Verify:
```
kubectl get pv | grep grafana
kubectl get pvc -n monitoring | grep grafana
```
[ image ]
(Screenshot idea: kubectl get pv + kubectl get pvc -n monitoring showing Grafana PV/PVC Bound)


3.3 Attach PVC to Grafana (Helm values)

If you deployed `kube-prometheus-stack` via Helm, Grafana persistence is usually configured in `values.yaml`.

Example values snippet (conceptual):
```
grafana:
  enabled: true

  # Persist Grafana database, plugins, and dashboards
  persistence:
    enabled: true
    existingClaim: grafana-nfs-pvc2

  # Optional: keep plugins in PVC (recommended)
  # [ image ]
  # Add screenshot: Grafana pod mount showing /var/lib/grafana on NFS.

prometheus:
  enabled: true

prometheusOperator:
  enabled: true

prometheusSpec:
  # Control TSDB growth (Time-series DB)
  retention: 10d
  retentionSize: 20GB

  # Prometheus Operator creates a PVC from this template
  storageSpec:
    volumeClaimTemplate:
      spec:
        storageClassName: ""
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 200Gi
```
> [ image ]
>  Add screenshot: `kubectl get pvc -n monitoring | grep prometheus showing Bound.

```
Apply your Helm upgrade:
```
```
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values.yaml
```

Verify Grafana is mounted on NFS:
```
kubectl exec -n monitoring deploy/prometheus-grafana -c grafana -- \
  sh -c 'mount | grep -i nfs; df -h | grep /var/lib/grafana'
```
  [ image ]
(Screenshot idea: Grafana pod mount showing NFS on /var/lib/grafana)


## Step 4 — Prometheus persistence (Prometheus Operator VolumeClaimTemplate)

Prometheus in `kube-prometheus-stack` is managed by Prometheus Operator.
That means we don’t edit a normal Deployment/StatefulSet directly — we configure the Prometheus CR.

We want:

- Prometheus TSDB stored on NFS at `/prometheus`

- TSDB growth controlled using:

  - retention: 10d

  - retentionSize: 20GB (so Prometheus does not grow indefinitely)

[ image ]
(Screenshot idea: Prometheus UI targets page + storage verification commands)


### 4.1 Create a Prometheus PV (Static)

Create `prometheus-nfs-pv.yaml`:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-nfs-pv
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-manual
  volumeMode: Filesystem
  nfs:
    server: 10.100.25.93
    path: /nfs/k8s/prometheus
```
Apply:
```
kubectl apply -f prometheus-nfs-pv.yaml
```

### 4.2 Configure the Prometheus CR storage

Edit the Prometheus custom resource:
```
kubectl edit prometheus prometheus-kube-prometheus-prometheus -n monitoring
```
Ensure the following block exists (indentation matters):
```
spec:
  retention: 10d
  retentionSize: 20GB

  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: nfs-manual
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 200Gi
```
[ image ]
(Screenshot idea: snippet of the Prometheus CR showing retention + storage block)

### 4.3 Important: StatefulSet Immutability

The Prometheus Operator creates a StatefulSet to manage the Prometheus pod.

`volumeClaimTemplates` inside a StatefulSet are immutable after creation.  
If storage configuration is added or modified after initial deployment, the existing StatefulSet must be recreated.

To trigger reconciliation with the updated storage configuration:
```
kubectl delete sts -n monitoring prometheus-prometheus-kube-prometheus-prometheus
```

The Prometheus Operator will automatically recreate it using the updated `volumeClaimTemplate`.

> [ image ]  
> *(Screenshot idea: deleting the StatefulSet and watching it reappear)*

---

### 4.4 Validate Prometheus PVC and PV Binding

After reconciliation, Prometheus automatically creates its PVC using the defined `volumeClaimTemplate`.

The PVC name will follow this pattern:

prometheus-prometheus-kube-prometheus-prometheus-db-prometheus-prometheus-kube-prometheus-prometheus-0


Validate binding:
```
kubectl get pvc -n monitoring | grep -i prometheus
kubectl get pv prometheus-nfs-pv
```

Expected result:
- PVC status: `Bound`
- PV status: `Bound`
- StorageClass: `nfs-manual`

> [ image ]  
> *(Screenshot idea: Prometheus PVC Bound to prometheus-nfs-pv with nfs-manual)*

---

### 4.5 Validate NFS Mount Inside the Pod

Confirm Prometheus is using the NFS-backed volume:
```
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus --
sh -c 'mount | grep -i " on /prometheus "; df -h | grep /prometheus'
```

Expected output should show:
```
10.100.25.93:/nfs/k8s/prometheus on /prometheus type nfs4
```

> [ image ]  
> *(Screenshot idea: mount output showing NFS path)*

---



## Step 5 — Verify Prometheus is using NFS

### 5.1 Confirm the pod is writing to /prometheus

Once Prometheus is running, verify the mount and write access:

```
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
  sh -c 'id; touch /prometheus/_write_test && ls -l /prometheus/_write_test; df -h | grep /prometheus; mount | grep -i " on /prometheus "'
```

Expected output should show:

- `uid=1000 gid=2000`
- NFS mount on `/prometheus`
- file created successfully

[ image ]
(Screenshot idea: successful write test + NFS mount line)

### 5.2 Resolve “permission denied” crash (real issue encountered)

Prometheus crashed with:
`open /prometheus/queries.active: permission denied`

Fix was applied on the NFS server:
```
sudo chown -R 1000:2000 /nfs/k8s/prometheus
sudo chmod -R 775 /nfs/k8s/prometheus
```
[ image ]
(Screenshot idea: Prometheus logs showing permission denied before fix and healthy after)

### Notes: “Why does Prometheus show the full NFS size?”

NFS exposes the underlying filesystem size, so inside the pod you may see ~200GB available even if your PV is 200Gi.

The real protection is Prometheus retention policy:

retention: 10d

retentionSize: 20GB

That’s what keeps the TSDB from growing indefinitely.

## Final Storage Architecture

| Component   | NFS Path                | StorageClass | Access Mode |
|------------|--------------------------|--------------|-------------|
| Grafana    | /nfs/k8s/grafana         | nfs-manual  | RWX |
| Prometheus | /nfs/k8s/prometheus      | nfs-manual  | RWO |

> [ image ]  
> *(Screenshot: `kubectl get pv` + `kubectl get pvc -n monitoring` showing both Bound with nfs-manual)*

 

### Troubleshooting (What I hit and how I fixed it)
1) storage class "nfs-manual" does not exist

- Cause: Prometheus Operator validates storageClassName in volumeClaimTemplate.
- Fix: Use `storageClassName: ""` for static PV binding OR create an actual StorageClass object named nfs-manual.

[ image ]
(Screenshot idea: operator logs showing the error)

2) PVC Pending: `no persistent volumes available for this claim and no storage class is set`

- Cause: PV and PVC StorageClass mismatch (empty vs named).
- Fix: Make PV storageClassName: "" match the PVC.

3) `spec.persistentVolumeSource is immutable after creation`

- Cause: You can’t edit PV NFS path in-place.
- Fix: Create a new PV pointing to the new NFS path, bind it to a new PVC.

4) Prometheus CrashLoop: `permission denied`

- Cause: NFS permissions don’t match Prometheus UID/GID.
- Fix: `chown -R 1000:2000 + chmod -R 775` on the NFS server.


## Final Verification Checklist

Run these and confirm everything is healthy:

Grafana
```
kubectl get pvc -n monitoring | grep grafana
kubectl exec -n monitoring deploy/prometheus-grafana -c grafana -- \
  sh -c 'mount | grep -i " on /var/lib/grafana "; df -h | grep /var/lib/grafana'
```
[ image ]
(Screenshot idea: Grafana NFS mount + df output)

Prometheus
```
kubectl get pvc -n monitoring | grep -i prometheus
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
  sh -c 'mount | grep -i " on /prometheus "; df -h | grep /prometheus'
```

[ image ]
(Screenshot idea: Prometheus NFS mount + df output)
