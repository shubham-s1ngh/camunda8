
# Camunda Platform 8 Local Development Setup with KinD


# Camunda Platform 8 Local Development Setup with KinD

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
   - [Create KinD Cluster](#1-create-kind-cluster-with-storage-configuration)
   - [Install Storage Provisioner](#2-install-storage-provisioner)
   - [Configure Storage Class](#3-configure-storage-class)
   - [Install NGINX Ingress](#4-install-nginx-ingress-controller)
3. [Important Notes About Kind Cluster](#important-notes-about-kind-cluster)
4. [Camunda Configuration](#camunda-configuration)
5. [Deployment Commands](#deployment-commands)
   - [Create Namespace](#create-namespace)
   - [Install Camunda](#install-camunda)
   - [Upgrade Installation](#upgrade-existing-installation)
6. [Management Commands](#management-commands)
   - [Shutdown Options](#shutdown-options)
   - [Startup After Scale Down](#startup-after-scale-down)
   - [Cleanup Commands](#cleanup-commands)
7. [Accessing Elasticsearch](#accessing-elasticsearch)
8. [Troubleshooting](#troubleshooting)
9. [Important Notes](#important-notes)
10. [Common Issues](#common-issues)
11. [Accessing the Applications](#accessing-the-applications)

## Setup Checklist
- [ ] Install Prerequisites
  - [ ] Docker Desktop
  - [ ] kubectl
  - [ ] Helm
  - [ ] KinD
- [ ] Create and configure KinD cluster
- [ ] Install Storage Provisioner
- [ ] Configure Storage Class
- [ ] Install NGINX Ingress Controller
- [ ] Create Camunda namespace
- [ ] Deploy Camunda Platform
- [ ] Configure hosts file
- [ ] Verify access to applications
  - [ ] Operate
  - [ ] Tasklist
  - [ ] Zeebe Gateway
  - [ ] Elasticsearch

## Quick Links
- [Docker Desktop Download](https://www.docker.com/products/docker-desktop)
- [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/)
- [Helm Installation Guide](https://helm.sh/docs/intro/install/)
- [KinD Installation Guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [Camunda Documentation](https://docs.camunda.io/)

## Prerequisites
- Docker Desktop
- kubectl
- Helm
- KinD (Kubernetes in Docker)

## Initial Setup

### 1. Create KinD Cluster with Storage Configuration

Create a file named `kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: C:\your\local\storage\path    # Windows path
    containerPath: /data
```

Create the cluster:
```bash
kind create cluster --config kind-config.yaml
```

### 2. Install Storage Provisioner
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### 3. Configure Storage Class

For Windows CMD:
```bash
kubectl patch storageclass local-path -p "{\"metadata\": {\"annotations\":{\"storageclass.kubernetes.io/is-default-class\":\"true\"}}}"
```

For PowerShell:
```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 4. Install NGINX Ingress Controller
```bash
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update helm repositories
helm repo update

# Install the ingress-nginx controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.hostPort.enabled=true \
  --set controller.service.ports.http=80 \
  --set controller.service.ports.https=443
```

Verify the installation:
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## Important Notes About Kind Cluster

When you delete a Kind cluster using `kind delete cluster`, everything installed in it will be deleted, including:
1. The NGINX ingress controller
2. All its configurations
3. All namespaces and their contents
4. All PersistentVolumeClaims (PVCs) and their data

This means you'll need to reinstall components like NGINX ingress controller after recreating the cluster.

## Camunda Configuration

Create a file named `values.yaml`:
```yaml
global:
  ingress:
    enabled: true
    className: nginx
    host: "camunda.local"
  identity:
    auth:
      enabled: false

identity:
  enabled: false

identityKeycloak:
  enabled: false

operate:
  contextPath: "/operate"

tasklist:
  contextPath: "/tasklist"

optimize:
  enabled: false

zeebe:
  clusterSize: 1
  partitionCount: 1
  replicationFactor: 1
  persistence:
    enabled: true
    storageClass: "standard"
    size: 10Gi

zeebeGateway:
  replicas: 1
  ingress:
    enabled: true
    className: nginx
    host: "zeebe.camunda.local"

connectors:
  enabled: true
  inbound:
    mode: disabled

elasticsearch:
  master:
    replicaCount: 1
    persistence:
      enabled: true
      storageClass: "standard"
      size: 15Gi
```

## Deployment Commands

### Create Namespace
```bash
kubectl create namespace camunda-app
```

### Install Camunda
```bash
helm install camunda camunda/camunda-platform -f values.yaml -n camunda-app
```

### Upgrade Existing Installation
```bash
helm upgrade camunda camunda/camunda-platform -f values.yaml -n camunda-app
```

## Management Commands

### Shutdown Options

Complete Uninstall:
```bash
helm uninstall camunda -n camunda-app
```

Scale Down (temporary):
```bash
kubectl scale statefulset,deployment -l app.kubernetes.io/instance=camunda --replicas=0 -n camunda-app
```

### Startup After Scale Down
```bash
kubectl scale statefulset,deployment -l app.kubernetes.io/instance=camunda --replicas=1 -n camunda-app
```

### Cleanup Commands

Delete PVCs (removes persistent data):
```bash
kubectl delete pvc --all -n camunda-app
```

Delete all related resources:
```bash
kubectl delete all -l app.kubernetes.io/instance=camunda -n camunda-app
```

## Accessing Elasticsearch

To access the Elasticsearch instance running in your cluster:

1. Use port-forward to access Elasticsearch:
```bash
kubectl port-forward -n camunda-app svc/camunda-elasticsearch 9200:9200
```

2. Test the connection (in a new terminal):
```bash
curl http://localhost:9200
```

3. View Elasticsearch indices:
```bash
curl http://localhost:9200/_cat/indices
```

4. Check cluster health:
```bash
curl http://localhost:9200/_cluster/health?pretty
```

If authentication is enabled, you can get the elastic password:
```bash
kubectl get secret -n camunda-app camunda-elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n camunda-app
kubectl describe pod <pod-name> -n camunda-app
```

### Check Storage
```bash
kubectl get pvc -n camunda-app
kubectl get storageclass
```

### View Logs
```bash
kubectl logs <pod-name> -n camunda-app
```

## Important Notes

- Data persists in the configured local storage path
- Default credentials: demo/demo
- Ensure enough resources are allocated to Docker Desktop
- Configure hosts file to map camunda.local and zeebe.camunda.local to localhost

## Common Issues

1. **Pending Pods**: Usually related to insufficient resources or storage class issues
2. **Storage Class Issues**: Verify the storage class name matches your cluster configuration
3. **Resource Constraints**: Adjust Docker Desktop resource limits if pods fail to start

## Accessing the Applications

After successful deployment, access the following URLs:
- Operate: http://camunda.local/operate
- Tasklist: http://camunda.local/tasklist
- Zeebe: zeebe.camunda.local

Remember to add these hostnames to your local hosts file (/etc/hosts or C:\Windows\System32\drivers\etc\hosts):
```
127.0.0.1 camunda.local
127.0.0.1 zeebe.camunda.local
```
```