# Prometheus & Grafana Monitoring Setup (kube-prometheus-stack)

This section documents the deployment of a full Kubernetes monitoring stack using **Prometheus**, **Alertmanager**, and **Grafana** via the **kube-prometheus-stack Helm chart**.

The goal is to achieve:

* Cluster-wide visibility
* Node and pod metrics collection
* A production-grade monitoring foundation

---

## Prerequisites

* A running Kubernetes cluster (installed via kubeadm)
* Calico CNI installed and all nodes in `Ready` state
* `kubectl` configured for the cluster
* Helm installed

### Verify cluster health

```
kubectl get nodes
kubectl get pods -A
```

All nodes should be `Ready` and core system pods running.

---

## Install Helm (if not already installed)

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

<img width="1016" height="39" alt="image" src="https://github.com/user-attachments/assets/21e8550e-bf19-40d9-82c8-658b2b408069" />

---

## Add Prometheus Community Helm Repository

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Verify repository:

```
helm search repo prometheus-community
```

<img width="1048" height="284" alt="image" src="https://github.com/user-attachments/assets/f48b3e60-f660-49fa-977f-09dd3569b8fc" />


---

## Deploy kube-prometheus-stack

This chart installs:

* Prometheus
* Alertmanager
* Grafana
* Node Exporter
* kube-state-metrics

### Create monitoring namespace

```
kubectl create namespace monitoring
```

### Install the stack

```
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

---

## Verify Monitoring Components

### Check pods

```
kubectl get pods -n monitoring
```

Expected pods include:

* `prometheus-monitoring-kube-prometheus-prometheus`
* `alertmanager-monitoring-kube-prometheus-alertmanager`
* `monitoring-grafana`
* `node-exporter`

<img width="711" height="174" alt="image" src="https://github.com/user-attachments/assets/d94c9a80-9655-4016-9489-9517998a5319" /> </br>

---

## Understanding Key Components

### Node Exporter

* Runs as a DaemonSet
* Collects node-level metrics (CPU, memory, disk, network)
* Used by Prometheus automatically
* Not VMware-specific

### Alertmanager

* Handles alert routing and grouping
* Supports email, Slack, PagerDuty, etc.
* Deployed even if alerts are not yet configured

---


## Access Grafana


### Ingress Controller Prerequisite

Ensure an Ingress controller (for example, NGINX Ingress Controller) is already deployed in the cluster.

Verify:

```
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
Create Grafana Ingress Resource
```

Create an Ingress manifest for Grafana:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitoring-grafana
            port:
              number: 80
```

Apply:

```
kubectl apply -f grafana-ingress.yaml
Verify Ingress
kubectl get ingress -n monitoring
```

Retrieve the ingress IP:

```
kubectl get ingress grafana-ingress -n monitoring
```

Access Grafana via browser:
```
http://<INGRESS-IP>
```

### Retrieve Grafana admin password

```
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```



## Access Prometheus (Optional – Port Forward)

Port-forwarding may be used for troubleshooting or local access if the Ingress is unavailable.

```
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Access Grafana at:

```
http://localhost:3000
```

Login:

* **Username:** admin
* **Password:** Retrieved from secret

<img width="1003" height="671" alt="image" src="https://github.com/user-attachments/assets/c45b4719-a604-4c46-9983-49fd3acc58e7" />



<img width="1363" height="571" alt="image" src="https://github.com/user-attachments/assets/88c2b490-841a-4850-b353-d91f3d1d335d" />
---

## Validation Commands (Crucial)

```
kubectl get svc -n monitoring
kubectl get daemonsets -n monitoring
kubectl get deployments -n monitoring
```

Check Node Exporter:

```
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter
```

---

## Outcome

At this stage:

* Kubernetes metrics are being collected
* Nodes and pods are monitored
* Grafana dashboards are available

The next section focuses on **dashboards and visualization**.
