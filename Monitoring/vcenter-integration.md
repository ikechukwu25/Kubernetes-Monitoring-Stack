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
| Username    | svc_grafana_vcenter       |
| Password    | StrongPassword             |
| Description | Monitoring Service Account |

Resulting login account:

```
svc_grafana_vcenter@vsphere.local
```

---

<img width="903" height="595" alt="image" src="https://github.com/user-attachments/assets/a4e1394e-9a5a-44a1-8fd9-f6b65e0b4d5c" />

```
vCenter Service Account Creation
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
| User                  | [svc_grafana_vcenter@vsphere.local](mailto:svc_grafana_vcenter@vsphere.local) |
| Role                  | Read-Only                                                                     |
| Propagate to Children | Enabled                                                                       |

This allows the exporter to access:

* Datacenters
* Clusters
* Hosts
* Virtual Machines
* Datastores

---

<img width="405" height="224" alt="image" src="https://github.com/user-attachments/assets/0049b5bd-d10a-4e50-9ee0-4b0be21ddb49" />

```
Exporter permissions
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

<img width="469" height="159" alt="image" src="https://github.com/user-attachments/assets/96c65d74-7649-4728-a394-9da87ea45458" />


```
docs/images/kubernetes/vmware-namespace.png
```

---

# 3. Create Kubernetes Secret for vCenter Credentials

Credentials must never be stored directly in Kubernetes deployment files.

A Kubernetes secret is used.

```
kubectl -n vmware-monitoring create secret generic vmware-exporter-secret \
  --from-literal=VSPHERE_USER='svc_grafana_vcenter@vsphere.local' \
  --from-literal=VSPHERE_PASSWORD='YourPassword'
```
  
Verify secret:

```
kubectl get secrets -n vmware-monitoring
```

---

<img width="466" height="55" alt="image" src="https://github.com/user-attachments/assets/da82ba17-ca3e-4d4b-b7b9-5121db64ab18" />

```
VMware Secret
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

<img width="587" height="56" alt="image" src="https://github.com/user-attachments/assets/2d06b2b8-917a-406a-a581-f6776de3b997" />

```
VMware Exporter Pod
```

---

# 5. Create Kubernetes Service for VMware Exporter

A Kubernetes **Service** is required to expose the VMware Exporter pod internally within the cluster.

This allows:
- Prometheus to discover and scrape the exporter endpoint
- The ServiceMonitor to route traffic to the correct pod

> Without this Service, the ServiceMonitor has nothing to select, and Prometheus will never scrape the exporter.

---

## Service Configuration

Create the file:
```
vi vmware-exporter-service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: vmware-exporter
  namespace: vmware-monitoring
  labels:
    app: vmware-exporter
spec:
  type: ClusterIP
  selector:
    app: vmware-exporter
  ports:
    - name: http
      port: 9272
      targetPort: 9272
      protocol: TCP
```

Apply the Service:
```
kubectl apply -f vmware-exporter-service.yaml
```

Verify the Service was created:
```
kubectl get svc -n vmware-monitoring
```

Expected output:
```
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
vmware-exporter    ClusterIP   10.96.x.x       <none>        9272/TCP   10s
```

<img width="601" height="61" alt="image" src="https://github.com/user-attachments/assets/f60cc5c4-aff5-4584-a9db-8c2de85572e7" />
</br>

Verify the Service is routing to the correct pod:
```
kubectl describe svc vmware-exporter -n vmware-monitoring
```

Confirm that the **Endpoints** field shows an IP — if it is empty, the `selector` label does not match the pod labels.


</br>

<img width="686" height="312" alt="image" src="https://github.com/user-attachments/assets/97393303-516a-462c-b1e8-30eadd595f89" />

</br>
---



# 6. Validate Exporter Metrics

### Verify exporter is being scraped successfully by Prometheus

We will validate metrics using the Prometheus API.

```
curl -s http://localhost:9090/api/v1/targets \
| jq '.data.activeTargets[] | select(.labels.job=="vmware-exporter") | {scrapeUrl, health}'
```
<img width="737" height="113" alt="image" src="https://github.com/user-attachments/assets/d6404aab-8bf4-4258-b959-10ebd36fe46c" />
</br>
</br>
Validate that VMware metrics exist in Prometheus

```
curl -s "http://localhost:9090/api/v1/query?query=vmware_vm_power_state" | jq
```

<img width="901" height="549" alt="image" src="https://github.com/user-attachments/assets/a8d6fe3c-049d-445e-b051-bba06e7d218b" />
</br>
</br>
Confirm metric count

```
curl -s "http://localhost:9090/api/v1/query?query=count(vmware_vm_power_state)"
```

<img width="865" height="46" alt="image" src="https://github.com/user-attachments/assets/b18313c8-86f9-480e-9738-8f6703266a35" />


---


### Validate Prometheus Scrape Targets

This step confirms that Prometheus is actively discovering and scraping the VMware exporter via the ServiceMonitor configuration.

Check Active Scrape Targets

- Run the following command against the Prometheus API:

```
curl -s http://localhost:9090/api/v1/targets \
| jq '.data.activeTargets[] | select(.labels.job | test("vmware"))'
```

- Expected Output (Healthy Target)

A successful scrape configuration will show:

<img width="969" height="278" alt="image" src="https://github.com/user-attachments/assets/4129df9b-f71a-415e-8d94-749923b68edb" />


# 7. Configure Prometheus Scraping

Prometheus Operator uses **ServiceMonitor objects** to automatically discover and scrape metrics from Kubernetes workloads.

A ServiceMonitor is a Kubernetes Custom Resource Definition (CRD) that tells Prometheus:

- Which Services to scrape
- Which namespace to look in
- Which endpoint/path to use
- How frequently to scrape metrics

This removes the need for manual Prometheus configuration files and enables fully declarative monitoring.

### Big Picture

This is the core observability flow in Kubernetes:

```
+-------------------+
| ServiceMonitor   |
+-------------------+
         |
         | 
         v
+----------------------+
| Prometheus Operator  |
+----------------------+
         |
         v
+--------------------------------------------+
| Generates Prometheus scrape configuration  |
+--------------------------------------------+
         |
         v
+----------------------+
| Prometheus server   |
+----------------------+
         |
         v
+---------------------------------------+
| Kubernetes Service (vmware-exporter)  |
+---------------------------------------+
         |
         v
+---------------------------------------+
| Exporter Pod (/metrics endpoint)  |
+---------------------------------------+

```

Create:

```
vi vmware-servicemonitor.yaml
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

<img width="554" height="293" alt="image" src="https://github.com/user-attachments/assets/b4b45894-2ed6-4bb5-b3fd-e3d00cb11af6" />

```
Service Monitors running under the monitoring namespace
```

---

# 8. Verify Prometheus Target

Forward Prometheus service:

```
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open:

```
http://localhost:9090/targets
```

Verify that VMware exporter target shows via URL:

```
Status: UP
```

API verification (CLI verification)
```
curl -s http://localhost:9090/api/v1/targets \
| jq '.data.activeTargets[] | select(.labels.job=="vmware-exporter") | {scrapeUrl, health}'
```

Expected Output
</br>

<img width="760" height="117" alt="image" src="https://github.com/user-attachments/assets/29c19aab-815c-4e33-b81d-0705801e44f4" />

</br>
</br>

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

## Total Virtual Machines

```
count(vmware_vm_power_state)
```

<img width="292" height="108" alt="image" src="https://github.com/user-attachments/assets/eab96293-6d1d-4d16-a7a1-5d2d7396b944" />

---

## Powered On VMs

```
count(vmware_vm_power_state == 1)
```

<img width="291" height="106" alt="image" src="https://github.com/user-attachments/assets/7105f077-1cbd-411c-b2ca-781f807136cc" />

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

<img width="587" height="298" alt="image" src="https://github.com/user-attachments/assets/08350bee-2e50-4f3b-b67f-c81b7d02953c" />

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


