# Project 7: Kubernetes Monitoring

A local Kubernetes monitoring lab using Prometheus and Grafana on kind.
Helm values are stored in this repository, and Grafana loads its dashboard
from the version-controlled configuration.

## Architecture

- Prometheus collects Kubernetes object metrics from kube-state-metrics.
- Grafana visualizes the Project 6 Deployment's desired and available
  replicas, running Pods, and container restarts.
- A Prometheus alert detects when a kind node is cordoned.
- Helm installs and configures the monitoring components.

## Requirements

- Ubuntu VM with Docker, kubectl, kind, and Helm
- Local kind cluster using the `kind-devops-lab` context
- Project 6's `project-06-web` Deployment in the `project-06` namespace
- About 2 vCPUs and 4 GiB RAM allocated to the VM

This lab disables node-exporter and Alertmanager to keep resource use lower.
The alert appears in Prometheus; it does not send email or chat notifications.

## Install or update

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana-community https://grafana-community.github.io/helm-charts
    helm repo update

    helm upgrade --install prometheus prometheus-community/prometheus \
      --namespace monitoring --create-namespace \
      --values values/prometheus.yaml

    helm upgrade --install grafana grafana-community/grafana \
      --namespace monitoring \
      --values values/grafana.yaml

Check the Pods:

    kubectl get pods -n monitoring

Grafana and Prometheus should be ready before opening their dashboards.

## Access Grafana

On the VM, start port forwarding:

    kubectl port-forward -n monitoring svc/grafana 3000:80

In another Git Bash window on the host, create an SSH tunnel, replacing
`<VM-IP>` with the VM's bridged IP address:

    ssh -L 3000:127.0.0.1:3000 zaid@<VM-IP>

Open http://localhost:3000 and sign in as `admin`. Retrieve the generated
password on the VM:

    kubectl get secret --namespace monitoring grafana \
      -o jsonpath="{.data.admin-password}" | base64 --decode
    echo

The dashboard is provisioned from `values/grafana.yaml`. Grafana's database
and Prometheus metrics use temporary storage in this lab, so saved UI changes
and metric history are lost when their Pods are replaced.

## Access Prometheus and inspect targets

On the VM:

    kubectl port-forward -n monitoring svc/prometheus-server 9090:80

In another host Git Bash window:

    ssh -L 9090:127.0.0.1:9090 zaid@<VM-IP>

Open http://localhost:9090/targets. The alert is available at
http://localhost:9090/alerts.

## Trigger the node alert

Cordoning a node marks it unschedulable without stopping it:

    kubectl cordon devops-lab-control-plane

After the alert appears in Prometheus, restore the node:

    kubectl uncordon devops-lab-control-plane

Use the exact node name shown by `kubectl get nodes` if it differs.

## Cleanup

    helm uninstall grafana -n monitoring
    helm uninstall prometheus -n monitoring
    kubectl delete namespace monitoring
