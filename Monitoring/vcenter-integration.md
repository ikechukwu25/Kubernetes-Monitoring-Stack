# VMware vCenter Integration with Kubernetes Monitoring Stack

## Overview

This document describes how VMware vCenter was integrated into the Kubernetes monitoring stack deployed in this project.

The monitoring stack consists of:

* **Prometheus (kube-prometheus-stack)**
* **Grafana**
* **VMware Exporter**
* **Prometheus Operator (ServiceMonitor)**

This integration allows the Kubernetes monitoring stack to collect metrics from VMware vCenter, including:

* ESXi host CPU utilization
* ESXi host memory usage
* Virtual machine CPU and memory consumption
* Datastore capacity and free space
* VM power state
* Cluster resource usage

These metrics are scraped by Prometheus and visualized in Grafana dashboards.

---

# Architecture

The monitoring flow is as follows:

```
vCenter API
      │
      ▼
VMware Exporter
      │
      ▼
Kubernetes Service
      │
      ▼
ServiceMonitor
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

**Flow Explanation**

1. VMware Exporter authenticates to vCenter using a service account.
2. Exporter retrieves inventory and performance counters from vCenter.
3. Exporter exposes metrics at `/metrics`.
4. Prometheus scrapes the exporter using a ServiceMonitor.
5. Grafana queries Prometheus and visualizes the data.

---

# 1. Create vCenter Read-Only Service Account

## Why a Service Account is Required

VMware Exporter must authenticate to vCenter in order to read:

* Hosts
* Clusters
* Datastores
* Virtual machines
* Performance counters

For security reasons, a **read-only service account** is used instead of administrator credentials.

---

## 1.1 Create the User

Log in to vCenter.

Navigate to:

```
Menu → Administration → Single Sign On → Users and Groups
```

Select the domain:

```
vsphere.local
```

Create a new user.

Example:

| Field       | Value                      |
| ----------- | -------------------------- |
| Username    | svc_vmware_exporter        |
| Password    | StrongPassword             |
| Description | Monitoring service account |

The resulting user will be:

```
svc_vmware_exporter@vsphere.local
```

---

## 1.2 Assign Read-Only Role

Navigate to:

```
Menu → Hosts and Clusters
```

Right-click the **vCenter root object**.

Select:

```
Add Permission
```

Assign:

| Setting               | Value                                                                         |
| --------------------- | ----------------------------------------------------------------------------- |
| User                  | [svc_vmware_exporter@vsphere.local](mailto:svc_vmware_exporter@vsphere.local) |
| Role                  | Read-Only                                                                     |
| Propagate to children | Enabled                                                                       |

This ensures the exporter can read all infrastructure objects.

---

# 2. Create Kubernetes Namespace

A dedicated namespace is used to isolate VMware monitoring components.

```
kubectl create namespace vmware-monitoring
```

Verify:

```
kubectl get ns
```

---

# 3. Create Kubernetes Secret for vCenter Credentials

Credentials must never be stored in plain text in deployment files.

Create a secret:

```
kubectl -n vmware-monitoring create secret generic vmware-exporter-secret \
  --from-literal=VSPHERE_USER='svc_vmware_exporter@vsphere.local' \
  --from-literal=VSPHERE_PASSWORD='YourPassword'
```

Verify:

```
kubectl get secrets -n vmware-monitoring
```

---

# 4. Deploy VMware Exporter

VMware Exporter is deployed as a Kubernetes Deployment.

Create the file:

```
vmware-exporter.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vmware-exporter
  namespace: vmware-monitoring
  labels:
    app: vmware-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vmware-exporter
  template:
    metadata:
      labels:
        app: vmware-exporter
    spec:
      containers:
        - name: vmware-exporter
          image: pryorda/vmware_exporter:v0.18.4
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 9272
          env:
            - name: VSPHERE_HOST
              value: "vcenter.domain.local"

            - name: VSPHERE_IGNORE_SSL
              value: "true"

            - name: VSPHERE_USER
              valueFrom:
                secretKeyRef:
                  name: vmware-exporter-secret
                  key: VSPHERE_USER

            - name: VSPHERE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: vmware-exporter-secret
                  key: VSPHERE_PASSWORD

            - name: VSPHERE_COLLECT_VMS
              value: "true"

            - name: VSPHERE_COLLECT_HOSTS
              value: "true"

            - name: VSPHERE_COLLECT_DATASTORES
              value: "true"

            - name: VSPHERE_COLLECT_CLUSTERS
              value: "true"

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

---

apiVersion: v1
kind: Service
metadata:
  name: vmware-exporter
  namespace: vmware-monitoring
  labels:
    app: vmware-exporter
spec:
  selector:
    app: vmware-exporter
  ports:
    - name: http
      port: 9272
      targetPort: 9272
  type: ClusterIP
```

Deploy the exporter:

```
kubectl apply -f vmware-exporter.yaml
```

Verify:

```
kubectl get pods -n vmware-monitoring
```

---

# 5. Verify Exporter Metrics

Forward the exporter service:

```
kubectl port-forward svc/vmware-exporter -n vmware-monitoring 9272:9272
```

Test metrics endpoint:

```
curl http://localhost:9272/metrics
```

Expected output:

Prometheus formatted metrics similar to:

```
vmware_host_cpu_usage_average
vmware_vm_power_state
vmware_datastore_capacity_size
```

---

# 6. Configure Prometheus Scraping (ServiceMonitor)

Because this project uses **Prometheus Operator**, scraping is configured using a ServiceMonitor.

Create:

```
vmware-servicemonitor.yaml
```

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vmware-exporter
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - vmware-monitoring
  selector:
    matchLabels:
      app: vmware-exporter
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

Apply:

```
kubectl apply -f vmware-servicemonitor.yaml
```

---

# 7. Verify Prometheus Target

Forward Prometheus:

```
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open:

```
http://localhost:9090/targets
```

The VMware exporter target should show:

```
Status: UP
```

---

# 8. VMware Metrics Collected

## Host Metrics

```
vmware_host_cpu_usage_average
vmware_host_memory_usage
vmware_host_memory_max
vmware_host_num_cpu
```

---

## VM Metrics

```
vmware_vm_cpu_usage_average
vmware_vm_memory_usage_average
vmware_vm_power_state
vmware_vm_num_cpu
```

---

## Datastore Metrics

```
vmware_datastore_capacity_size
vmware_datastore_freespace_size
vmware_datastore_uncommited_size
vmware_datastore_accessible
```

---

# 9. Grafana Dashboard Queries

## Cluster CPU Utilization

```
avg(vmware_host_cpu_usage_average) / 100
```

---

## Cluster Memory Utilization

```
avg(vmware_host_memory_usage / vmware_host_memory_max) * 100
```

---

## Datastore Usage %

```
(
sum(vmware_datastore_capacity_size)
-
sum(vmware_datastore_freespace_size)
)
/ sum(vmware_datastore_capacity_size) * 100
```

---

## Total Virtual Machines

```
count(vmware_vm_power_state)
```

---

## Powered On VMs

```
count(vmware_vm_power_state == 1)
```

---

# 10. Troubleshooting

## Exporter Pod Fails

Check logs:

```
kubectl logs -n vmware-monitoring deploy/vmware-exporter
```

Common causes:

* Invalid credentials
* DNS resolution issues
* SSL verification problems

---

## Prometheus Target Down

Check ServiceMonitor:

```
kubectl describe servicemonitor vmware-exporter -n monitoring
```

Ensure label matches Helm release:

```
release: prometheus
```

---

# Result

The Kubernetes monitoring stack now provides full observability into VMware infrastructure, including:

* Host CPU and memory utilization
* VM performance metrics
* Datastore capacity monitoring
* Virtual machine power state visibility

These metrics are collected automatically and visualized through Grafana dashboards.

This completes the integration of VMware vCenter into the Kubernetes Monitoring Stack.
