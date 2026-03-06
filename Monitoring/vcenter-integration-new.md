# VMware vCenter Integration with Kubernetes Monitoring Stack

## Overview

This document describes the **complete integration of VMware vCenter into the Kubernetes Monitoring Stack** deployed in this project.

The goal of this integration is to enable Prometheus and Grafana to collect and visualize metrics from VMware infrastructure, such as:

* ESXi Host CPU utilization
* ESXi Host memory consumption
* Datastore capacity and usage
* Virtual machine CPU and memory metrics
* VM power states
* Cluster resource capacity

The monitoring stack consists of the following components:

| Component           | Purpose                                   |
| ------------------- | ----------------------------------------- |
| Prometheus          | Scrapes and stores infrastructure metrics |
| Grafana             | Visualizes infrastructure metrics         |
| VMware Exporter     | Collects metrics from vCenter             |
| Prometheus Operator | Automates Prometheus configuration        |
| ServiceMonitor      | Defines scrape targets                    |

---

# Architecture

## Monitoring Flow

```
vCenter → VMware Exporter → Kubernetes Service → ServiceMonitor → Prometheus → Grafana
```

### Explanation

1. VMware Exporter connects to the **vCenter API**.
2. Exporter retrieves inventory and performance metrics.
3. Exporter exposes metrics through `/metrics`.
4. Prometheus scrapes the exporter endpoint.
5. Grafana queries Prometheus and displays the data.

---

## Infrastructure Layout

```
+-------------------+
| VMware vCenter    |
+-------------------+
         |
         | API
         v
+----------------------+
| VMware Exporter Pod  |
| Kubernetes Cluster   |
+----------------------+
         |
         v
+----------------------+
| Prometheus Operator  |
+----------------------+
         |
         v
+----------------------+
| Prometheus Server    |
+----------------------+
         |
         v
+----------------------+
| Grafana Dashboards   |
+----------------------+
```

---

# 1. Create vCenter Read-Only Service Account

## Purpose

VMware Exporter authenticates with vCenter using a service account.

This account must have permissions to **read infrastructure inventory and performance metrics** but **must not have administrative privileges**.

Using a dedicated service account improves:

* security
* auditing
* credential management

---

## 1.1 Create Service Account

Login to the **vCenter Web Client**.

Navigate to:

```
Menu → Administration → Single Sign On → Users and Groups
```

Select the domain:

```
vsphere.local
```

Create a new user.

Example configuration:

| Field       | Value                      |
| ----------- | -------------------------- |
| Username    | svc_vmware_exporter        |
| Password    | StrongPassword             |
| Description | Monitoring Service Account |

Resulting login account:

```
svc_vmware_exporter@vsphere.local
```

---

### 📸 Screenshot Evidence (NEW)

Insert screenshot of user creation.

```
docs/images/vmware/vcenter-user-created.png
```

Example placeholder:

```
![vCenter Service Account Creation](../images/vmware/vcenter-user-created.png)
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

Assign the following:

| Setting               | Value                                                                         |
| --------------------- | ----------------------------------------------------------------------------- |
| User                  | [svc_vmware_exporter@vsphere.local](mailto:svc_vmware_exporter@vsphere.local) |
| Role                  | Read-Only                                                                     |
| Propagate to Children | Enabled                                                                       |

This allows the exporter to access:

* Datacenters
* Clusters
* Hosts
* Virtual Machines
* Datastores

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/vmware/vcenter-permissions.png
```

---

# 2. Create Kubernetes Namespace

To isolate VMware monitoring resources, a dedicated namespace is created.

```
kubectl create namespace vmware-monitoring
```

Verify:

```
kubectl get namespaces
```

Expected output:

```
vmware-monitoring   Active
```

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/kubernetes/vmware-namespace.png
```

---

# 3. Create Kubernetes Secret for vCenter Credentials

Credentials must never be stored directly in Kubernetes deployment files.

A Kubernetes secret is used.

```
kubectl -n vmware-monitoring create secret generic vmware-exporter-secret \
  --from-literal=VSPHERE_USER='svc_vmware_exporter@vsphere.local' \
  --from-literal=VSPHERE_PASSWORD='YourPassword'
```

Verify secret:

```
kubectl get secrets -n vmware-monitoring
```

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/kubernetes/vmware-secret.png
```

---

# 4. Deploy VMware Exporter

VMware Exporter is deployed as a Kubernetes Deployment.

File created:

```
vmware-exporter.yaml
```

### Deployment Configuration

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
```

Apply deployment:

```
kubectl apply -f vmware-exporter.yaml
```

Verify:

```
kubectl get pods -n vmware-monitoring
```

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/kubernetes/vmware-exporter-pod.png
```

---

# 5. Validate Exporter Metrics

Test that exporter is serving metrics.

```
kubectl port-forward svc/vmware-exporter -n vmware-monitoring 9272:9272
```

Test endpoint:

```
curl http://localhost:9272/metrics
```

Expected metrics example:

```
vmware_host_cpu_usage_average
vmware_vm_power_state
vmware_datastore_capacity_size
```

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/prometheus/exporter-metrics.png
```

---

# 6. Configure Prometheus Scraping

Prometheus Operator uses **ServiceMonitor objects**.

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

### 📸 Screenshot Evidence (NEW)

```
docs/images/prometheus/servicemonitor-created.png
```

---

# 7. Verify Prometheus Target

Forward Prometheus service:

```
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open:

```
http://localhost:9090/targets
```

Verify VMware exporter target shows:

```
Status: UP
```

---

### 📸 Screenshot Evidence (NEW)

```
docs/images/prometheus/prometheus-target-up.png
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

## VM Metrics

```
vmware_vm_cpu_usage_average
vmware_vm_memory_usage_average
vmware_vm_power_state
vmware_vm_num_cpu
```

## Datastore Metrics

```
vmware_datastore_capacity_size
vmware_datastore_freespace_size
vmware_datastore_uncommited_size
vmware_datastore_accessible
```

---

# 9. Grafana Dashboards

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

### 📸 Screenshot Evidence (NEW)

```
docs/images/grafana/vcenter-dashboard.png
```

---

# 10. Troubleshooting

## Exporter Pod Crash

```
kubectl logs -n vmware-monitoring deploy/vmware-exporter
```

Common causes:

* Incorrect credentials
* DNS resolution failure
* SSL verification errors

---

## Prometheus Target Down

Verify ServiceMonitor:

```
kubectl describe servicemonitor vmware-exporter -n monitoring
```

Ensure label matches Helm release:

```
release: prometheus
```

---

# Final Result

After completing this integration, the Kubernetes monitoring stack provides full observability of VMware infrastructure, including:

* Host resource utilization
* Datastore capacity monitoring
* Virtual machine performance metrics
* Real-time virtualization visibility

This enables proactive monitoring, capacity planning, and performance troubleshooting for VMware environments.

---


