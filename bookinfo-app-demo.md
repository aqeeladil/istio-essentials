# Setting up Istio on a Kubernetes cluster (Minikube) for Istio’s Bookinfo application. 

**It covers installation, configuration, traffic management, observability, and security with real-world explanations and detailed commands.**

**Istio’s Bookinfo application, simulates an e-commerce app:**
- Product Page: Displays book details.
- Details Service: Provides book information.
- Reviews Service: Provides book reviews.
- Ratings Service: Provides star ratings for reviews.

## Architecture 

This diagram illustrates the architecture of the Bookinfo application with Istio:

![Bookinfo with Istio](https://istio.io/latest/docs/examples/bookinfo/withistio.svg)

## 1. Launch an EC2 Instance:
- Launch Ubuntu instance.
- Security Group: Open the following ports:
    - 22: SSH Access
    - 80, 443: Web Traffic
    - 30000-32767: NodePort services (Minikube exposes services on these ports)
- Connect via SSH:
    - `ssh -i your-key.pem ubuntu@your-ec2-public-ip`

## 2. Install Required Dependencies
```bash
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
kubectl version --client

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
minikube version

# Download Istio CLI: 
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.24.2  # Replace with your version
# The installation directory contains: Sample applications in samples/
# The istioctl client binary in the bin/ directory.
# Add the istioctl client to your path (Linux)
export PATH=$PWD/bin:$PATH
istioctl version

# Reboot your EC2 instance to ensure all configurations apply.
sudo reboot
```

## 3. Start Minikube with Docker Driver
```bash
minikube start --driver=docker --memory=4096 --cpus=2

# minikube uses Docker as the virtualization backend.

minikube status
kubectl get nodes
```

## 4. Install Istio on Minikube
```bash
# Istio provides several profiles for installation: Demo, Minimal, Production.
# Install Istio with the Demo Profile
# This will install: Istiod (Control Plane), Ingress Gateway, Egress Gateway
istioctl install --set profile=demo -y

# Verify Installation: 
kubectl get pods -n istio-system
kubectl get svc -n istio-system

# Configure namespace for automatic sidecar injection.
kubectl label namespace default istio-injection=enabled

# Verify the label
kubectl get namespace default --show-labels
```

## 5. Deploy Sample Bookinfo Application
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml

# Verify Deployment
kubectl get pods
kubectl get svc

# Each pod should have 2 containers (application + sidecar proxy).
```

## 6. Expose the Application to External Traffic
```bash
# Apply Gateway Configuration: Istio Ingress Gateway is used to expose the application.
# Use the Kubernetes Gateway API to deploy a gateway called bookinfo-gateway
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/networking/bookinfo-gateway.yaml

# By default, Istio creates a LoadBalancer service for a gateway. As you will access this gateway by a tunnel, you don’t need a load balancer. Change the service type to ClusterIP by annotating the gateway:
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default

# Check Gateway Status
kubectl get gateway
kubectl get svc istio-ingressgateway -n istio-system

# Connect to the Bookinfo productpage service through the gateway you just provisioned
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
# Open your browser and navigate to http://<ec2-public-ip>:8080/productpage to view the Bookinfo application.


# # Get the external IP of the Istio Ingress Gateway:
# export INGRESS_HOST=$(minikube ip)
# export INGRESS_PORT=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
# echo "http://$INGRESS_HOST:$INGRESS_PORT/productpage"

# # Open the URL in your browser to view the Bookinfo Application page.
# http://<Minikube-IP>:<NodePort>/productpage
```

## 7. Enable Mutual TLS (mTLS)
```bash
# Apply mTLS Policy
# By default, Istio runs in Permissive Mode. Enable STRICT mode for better security.
kubectl apply -f mTLS/tls-mode.yaml 

# Ensure traffic between services is encrypted using:
# Test it by accessing services directly via `curl`. You should see a connection error without valid certificates.
istioctl x describe pod <pod-name>
istioctl proxy-config endpoints <pod-name> -n default
kubectl describe peerauthentication default -n default
```

## 8. Enable Observability with Kiali
```bash
# Install Kiali and Add-On tools
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system

# Verify pods:
kubectl get pods -n istio-system

# Access Kiali Dashboard
# Kiali will open in your default browser. 
# Explore traffic graphs, service health, and tracing
istioctl dashboard kiali
istioctl dashboard jaeger
istioctl dashboard prometheus
```

## 9. Traffic Management Example (Canary Deployment)
```bash
# Let's route 90% traffic to v1 and 10% to v2.

# Apply Configurations
kubectl apply -f traffic-management/canary-virtualservice.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/networking/destination-rule-all.yaml

# Refresh the Product Page: You’ll notice 90% of requests go to version 1, and 10% go to version 2 of the reviews service.
```


