# Setting up Istio with Kubernetes (Minikube)
A comprehensive guide for deploying and managing Istio service mesh with the Bookinfo sample application on a Minikube cluster.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Infrastructure Setup](#infrastructure-setup)
- [Istio Installation](#istio-installation)
- [Bookinfo Application Deployment](#bookinfo-application-deployment)
- [Security Configuration](#security-configuration)
- [Observability Setup](#observability-setup)
- [Traffic Management](#traffic-management)
- [Troubleshooting](#troubleshooting)

## Overview

### About Bookinfo Application
The Bookinfo sample application demonstrates Istio's core functionalities through a simulated e-commerce platform consisting of four microservices:
- **Product Page Service**: Frontend service providing the user interface
- **Details Service**: Book information microservice
- **Reviews Service**: Book reviews microservice
- **Ratings Service**: Book ratings microservice

### Architecture
The application architecture implements a microservices pattern with Istio service mesh:

```ascii
[External Client] → [Istio Ingress Gateway] → [Product Page] → [Details]
                                                            ↘ [Reviews] → [Ratings]
```
![Bookinfo with Istio](https://istio.io/latest/docs/examples/bookinfo/withistio.svg)

## Prerequisites

### System Requirements
- Minimum 4GB RAM
- 2 CPU cores
- 20GB free disk space
- Ubuntu 20.04 or later

### Required Software Versions
- Docker 20.10.x or later
- Kubernetes 1.24.x or later
- Minikube v1.24.x or later
- Istio 1.24.x or later

## Infrastructure Setup

### 1. EC2 Instance Configuration
```bash
# Security Group Configuration
- Inbound Rules:
  - SSH (22)
  - HTTP (80)
  - HTTPS (443)
  - NodePort Services (30000-32767)
```

### 2. Dependencies Installation

```bash
# Update System
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube

# Install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.24.2
# The installation directory contains: 
# Sample applications in samples/
# istioctl client binary in the bin/ directory.
# Add the istioctl client to your path (Linux)
export PATH=$PWD/bin:$PATH
istioctl version

# Reboot your EC2 instance to ensure all configurations apply.
sudo reboot
```

### 3. Minikube Cluster Setup
```bash
# Start Minikube with sufficient resources
minikube start --driver=docker --memory=4096 --cpus=2

# minikube uses Docker as the virtualization backend.

# Verify cluster status
minikube status
kubectl get nodes
```

## Istio Installation

### 1. Install Istio Control Plane
```bash
# Istio provides several profiles for installation: Demo, Minimal, Production.
# Install Istio with the Demo Profile
# This will install: Istiod (Control Plane), Ingress Gateway, Egress Gateway

# Install demo profile
istioctl install --set profile=demo -y

# Verify installation
kubectl get pods,svc -n istio-system
```

### 2. Configure Namespace
```bash
# Configure namespace for automatic sidecar injection.
kubectl label namespace default istio-injection=enabled

# Verify the label
kubectl get namespace default --show-labels
```

## Bookinfo Application Deployment

### 1. Deploy Application
```bash
# Deploy core services
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml

# Verify deployment
kubectl get pods,svc
# Each pod should have 2 containers (application + sidecar proxy).
```

### 2. Configure External Access
```bash
# Deploy Gateway
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/networking/bookinfo-gateway.yaml

# By default, Istio creates a LoadBalancer service for a gateway. As you will access this gateway by a tunnel, you don’t need a load balancer. Change the service type to ClusterIP by annotating the gateway:
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default

# Check Gateway Status
kubectl get gateway
kubectl get svc istio-ingressgateway -n istio-system

# Connect to the Bookinfo productpage service through the gateway you just provisioned
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
# Open your browser and navigate to http://<ec2-public-ip>:8080/productpage to view the Bookinfo application.
```

## Security Configuration

### Enable mTLS
```bash
# Apply mTLS Policy
# By default, Istio runs in Permissive Mode. Enable STRICT mode for better security.
kubectl apply -f mTLS/tls-mode.yaml

# Verify configuration
# Ensure traffic between services is encrypted:
# Test it by accessing services directly via `curl`. You should see a connection error without valid certificates.
istioctl x describe pod <pod-name>
istioctl proxy-config endpoints <pod-name> -n default
kubectl describe peerauthentication default -n default
```

## Observability Setup

### 1. Install Monitoring Tools
```bash
# Deploy monitoring stack
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system

# Verify deployment
kubectl get pods -n istio-system
```

### 2. Access Dashboards
```bash
# Open monitoring dashboards
istioctl dashboard kiali     # Service mesh visualization
istioctl dashboard jaeger    # Distributed tracing
istioctl dashboard prometheus # Metrics and monitoring
```

## Traffic Management

### Configure Canary Deployment
```bash
# Let's route 90% traffic to v1 and 10% to v2.

# Apply Configurations
kubectl apply -f traffic-management/canary-virtualservice.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/networking/destination-rule-all.yaml

# Refresh the Product Page: You’ll notice 90% of requests go to version 1, and 10% go to version 2 of the reviews service.
```

## Troubleshooting

### Common Issues and Solutions
1. **Pod Creation Failures**
   ```bash
   kubectl describe pod <pod-name>
   kubectl logs <pod-name> -c istio-proxy
   ```

2. **Gateway Connectivity Issues**
   ```bash
   istioctl analyze
   kubectl get events -n istio-system
   ```

3. **mTLS Problems**
   ```bash
   istioctl x describe pod <pod-name>
   kubectl get peerauthentication -A
   ```

### Health Checks
```bash
# Verify Istio components
istioctl verify-install

# Check proxy status
istioctl proxy-status
```
