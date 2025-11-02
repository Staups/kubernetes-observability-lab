# Setup Guide - Kubernetes Observability Lab

This guide will walk you through setting up the complete observability lab environment. Choose your preferred deployment method based on your infrastructure requirements.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Options](#environment-options)
- [Local Setup with KinD](#local-setup-with-kind)
- [Local Setup with Minikube](#local-setup-with-minikube)
- [Cloud Setup](#cloud-setup)
- [Deploy Observability Stack](#deploy-observability-stack)
- [Verification](#verification)
- [Next Steps](#next-steps)

## Prerequisites

Before starting, ensure you have the following tools installed:

### Required Tools

```bash
# Docker (version 20.10+)
docker --version

# kubectl (version 1.25+)
kubectl version --client

# Helm (version 3.8+)
helm version

# Git
git --version
```

### Optional Tools

```bash
# For cloud deployments
terraform --version

# For advanced debugging
k9s version
```

### System Requirements

- **CPU**: 4+ cores recommended
- **Memory**: 8GB+ RAM
- **Storage**: 20GB+ free space
- **Network**: Internet access for pulling images

## Environment Options

Choose one of the following deployment options:

| Option | Pros | Cons | Best For |
|--------|------|------|----------|
| **KinD** | Fast, lightweight, Docker-based | Limited to single node | Development, CI/CD |
| **Minikube** | Feature-rich, supports addons | Higher resource usage | Local development |
| **Cloud** | Production-like, scalable | Costs money, complex setup | Production testing |

## Local Setup with KinD

### 1. Install KinD

```bash
# Linux/macOS
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Windows (using chocolatey)
choco install kind

# Verify installation
kind version
```

### 2. Create Cluster Configuration

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: observability-lab
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 3000
    hostPort: 3000
    protocol: TCP
  - containerPort: 9090
    hostPort: 9090
    protocol: TCP
- role: worker
- role: worker
```

### 3. Create the Cluster

```bash
# Create cluster
kind create cluster --config=kind-config.yaml

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

## Local Setup with Minikube

### 1. Install Minikube

```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS
brew install minikube

# Windows
choco install minikube

# Verify installation
minikube version
```

### 2. Start Minikube

```bash
# Start with specific resources
minikube start \
  --cpus=4 \
  --memory=8192 \
  --disk-size=20g \
  --driver=docker \
  --kubernetes-version=v1.28.0

# Enable required addons
minikube addons enable ingress
minikube addons enable metrics-server

# Verify cluster
kubectl cluster-info
```

## Cloud Setup

### Azure Kubernetes Service (AKS)

```bash
# Login to Azure
az login

# Create resource group
az group create --name observability-lab-rg --location eastus

# Create AKS cluster
az aks create \
  --resource-group observability-lab-rg \
  --name observability-lab-aks \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group observability-lab-rg --name observability-lab-aks
```

### Google Kubernetes Engine (GKE)

```bash
# Set project and zone
gcloud config set project YOUR_PROJECT_ID
gcloud config set compute/zone us-central1-a

# Create cluster
gcloud container clusters create observability-lab \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autorepair \
  --enable-autoupgrade

# Get credentials
gcloud container clusters get-credentials observability-lab
```

## Deploy Observability Stack

### 1. Add Helm Repositories

```bash
# Add required Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

# Update repositories
helm repo update
```

### 2. Create Namespace

```bash
# Create observability namespace
kubectl create namespace observability

# Set as default namespace (optional)
kubectl config set-context --current --namespace=observability
```

### 3. Deploy Prometheus Stack

```bash
# Deploy kube-prometheus-stack (includes Prometheus, Grafana, AlertManager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values ./charts/prometheus/values.yaml \
  --wait
```

### 4. Deploy Loki

```bash
# Deploy Loki for log aggregation
helm install loki grafana/loki-stack \
  --namespace observability \
  --values ./charts/loki/values.yaml \
  --wait
```

### 5. Deploy Jaeger

```bash
# Deploy Jaeger for distributed tracing
helm install jaeger jaegertracing/jaeger \
  --namespace observability \
  --values ./charts/jaeger/values.yaml \
  --wait
```

### 6. Deploy OpenTelemetry Collector

```bash
# Deploy OpenTelemetry Collector
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values ./charts/opentelemetry/values.yaml \
  --wait
```

## Verification

### 1. Check Pod Status

```bash
# Verify all pods are running
kubectl get pods -n observability

# Check services
kubectl get svc -n observability
```

### 2. Access Web Interfaces

#### Grafana Dashboard

```bash
# Get Grafana password
kubectl get secret --namespace observability prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# Port forward (if not using ingress)
kubectl port-forward --namespace observability svc/prometheus-grafana 3000:80

# Access: http://localhost:3000
# Username: admin
# Password: (from above command)
```

#### Prometheus UI

```bash
# Port forward Prometheus
kubectl port-forward --namespace observability svc/prometheus-kube-prometheus-prometheus 9090:9090

# Access: http://localhost:9090
```

#### Jaeger UI

```bash
# Port forward Jaeger
kubectl port-forward --namespace observability svc/jaeger-query 16686:16686

# Access: http://localhost:16686
```

### 3. Deploy Sample Application

```bash
# Deploy sample instrumented application
kubectl apply -f ./examples/sample-app/

# Verify sample app
kubectl get pods -n default
```

## Next Steps

After successful setup, you can:

1. **Explore Dashboards**: Navigate to Grafana and explore pre-configured dashboards
2. **Generate Traffic**: Use the sample applications to generate metrics, logs, and traces
3. **Configure Alerts**: Set up alerting rules in Prometheus AlertManager
4. **Custom Instrumentation**: Add OpenTelemetry instrumentation to your applications
5. **Advanced Configuration**: Customize retention policies, storage, and integrations

## Troubleshooting

### Common Issues

#### Pods Not Starting

```bash
# Check pod logs
kubectl logs -n observability <pod-name>

# Describe pod for events
kubectl describe pod -n observability <pod-name>
```

#### Resource Constraints

```bash
# Check node resources
kubectl top nodes
kubectl top pods -n observability

# Scale down if needed
kubectl scale deployment -n observability <deployment-name> --replicas=1
```

#### Network Issues

```bash
# Check services
kubectl get svc -n observability

# Test service connectivity
kubectl run debug --image=busybox -it --rm --restart=Never -- sh
```

### Getting Help

If you encounter issues:

1. Check the [troubleshooting section](troubleshooting.md)
2. Review pod logs and events
3. Consult the official documentation for each tool
4. Open an issue in this repository

## Cleanup

### Remove Lab Environment

```bash
# Delete KinD cluster
kind delete cluster --name observability-lab

# Delete Minikube cluster
minikube delete

# Delete cloud resources (be careful!)
# Azure: az group delete --name observability-lab-rg
# GCP: gcloud container clusters delete observability-lab
```

---

**Next**: Continue with [Configuration Guide](configuration.md) to customize your observability stack.
