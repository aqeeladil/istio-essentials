# Intro to Service Mesh | Istio

## 1. Service Mesh Overview

- A Service Mesh is a dedicated infrastructure layer built to manage and control communication between services in a microservices architecture. 
- It focuses on east-west traffic (service-to-service communication within a Kubernetes cluster) rather than north-south traffic (traffic entering or exiting the cluster).
- It provides features like traffic management, observability, security, and resilience without requiring changes to the application code.

**Popular Service Mesh Tools**
- Istio: 
    - Open-source, feature-rich service mesh built on Envoy.
- Linkerd: 
    - Lightweight and developer-friendly service mesh.
- Consul by HashiCorp: 
    - Integrates with both service mesh and service discovery.
- Kuma: 
    - Universal control plane, built on Envoy.
- AWS App Mesh: 
    - Managed service mesh offering by AWS.

**Service Mesh vs Ingress**
- A service mesh operates at the layer of service-to-service communication within a cluster, providing features like traffic management, security, and observability. 
- In contrast, an ingress controller manages external access to services within the cluster, typically handling tasks like load balancing, SSL termination, and routing based on HTTP/HTTPS requests.

## 2. Why and When to Use Service Mesh?

Without a service mesh, developers have to manually implement: service discovery, load balancing, security, advanced deployment strategies and observability.

**Why Use a Service Mesh?**
- *Traffic Management:*
    - Control complex traffic flow between services.
    - Implement advanced strategies like Canary Deployments and Blue-Green Deployments.
    - Virtual Services & Destination Rules enable fine-grained traffic control.
- *Security:*
    - Enable Mutual TLS (mTLS) for encrypted and authenticated communication between services.
    - Services validate each other's identities using certificates.
- *Observability:*
    - Monitor traffic, errors, and latency between services.
    - Integrated tools like Kiali provide dashboards for visual insights.
- *Fault Tolerance:*
    - Implement Circuit Breaking to prevent cascading failures.
    - Set up retry and timeout policies.

**When to Use a Service Mesh?**
- Microservices architecture with many services.
- When communication patterns are complex.
- When security and observability are priorities.

## 3. How Istio Works?

Istio is an open-source service mesh that provides a unified way to secure, connect, and observe microservices architectures. Built on Envoy Proxy, it simplifies managing microservices traffic, ensures security, and offers in-depth observability.

1. **Sidecar Containers (Envoy Proxy) (Data Plane)**
    - In Kubernetes, a sidecar container is an additional container that runs alongside the main application container in the same pod. 
    - Sidecar proxies (Envoy) are the backbone of Istio's capabilities.
    - They enhance or extend the functionality of the main container without modifying its code.
        - Intercept traffic coming in/out of the pod.
        - Enforce policies (e.g., Mutual TLS).
        - Enable observability and traffic management.
    - Envoy Proxy acts as a data plane proxy to handle all communication between services.
    - Istio injects a sidecar container (Envoy Proxy) into every Kubernetes Pod.
    - Examples of Sidecar Use Cases:
        - Service Proxy: Envoy Proxy in Istio.
        - Logging & Monitoring: Fluentd.
        - Security: mTLS (Mutual TLS).

2. **Traffic Management (Virtual Services & Destination Rules)**
    - Traffic management in a service mesh involves controlling the flow of requests between services. This includes features such as load balancing, routing, traffic shaping, and fault tolerance mechanisms like retries and circuit breaking.
    - *Virtual Services:* 
        - Define traffic routing rules (how traffic should flow between services).
        - e.g., split traffic 50-50 between two versions. 
        - They enable sophisticated traffic management strategies like A/B testing and canary deployments.
    - *Destination Rules:* 
        - Define policies for traffic routing (where traffic should go) such as subsets and load-balancing strategies.
        - e.g., send 50% to version 1 and 50% to version 2. 
        - They define things like load balancing algorithms, circuit breaking settings, and TLS settings for communication between services.
    - Capabilities: Canary deployments, Traffic mirroring, Fault injection, Traffic splitting (A/B testing).
    - Example Use Case: Canary Deployment
        - Initially, route 100% traffic to v1.
        - Gradually route 10%, 20%, 50% of traffic to v2.
        - Once stable, route 100% traffic to v2.

3. **Mutual TLS (mTLS)**
    - Mutual TLS in Istio ensures end-to-end encryption and secure communication between services by requiring both the client and the server to present a valid TLS certificate. 
    - Communication fails if certificates don't match.
    - It encrypts traffic and provides authentication, preventing unauthorized access or eavesdropping.
    - Permissive Mode: Services can communicate with or without TLS.
    - Strict Mode: Services must use mTLS for communication.

4. **Admission Controllers**
    - Kubernetes uses Admission Controllers to validate or modify Pod configurations before they are created.
    - Admission controllers in Kubernetes enforce policies on objects during their creation or modification. 
    - Based on predefined rules, they validate(approve/reject) or mutate(modify) resource requests (coming to the Kubernetes API server) before they are saved in etcd.
    - Examples of Admission Controllers:
        - Default Storage Class Controller: Adds default storage class if none is specified in Persistent Volume Claim (PVC).
        - Resource Quota Controller: Ensures namespace resource limits are respected.
    - Istio uses Dynamic Admission Webhooks to automatically inject sidecar proxies into Pods during creation.
    - Dynamic Admission Control:
        - Mutating Admission Webhook: Allows external systems (like Istio) to modify resources dynamically (e.g., injecting sidecars).
        - Validating Admission Webhook: Ensures requests meet certain rules before being processed.
    - Practical Example:
        - A PersistentVolumeClaim without a storage class is automatically assigned one via an admission controller.
        - Kubernetes validates resource quotas using the validation webhook before creating a pod.

5. **Circuit Breaking**
    - Circuit breaking is a pattern used to prevent cascading failures in distributed systems.
    - In Istio, it involves setting thresholds for error rates or latency, and if these thresholds are exceeded, requests to a service are automatically stopped, preventing further damage.

6. **Gateways:**
    - Gateways in Istio allow external traffic to enter the service mesh. 
    - They act as ingress(incoming) and egress(outgoing) points for traffic, enabling communication between services inside the mesh and external clients or services outside the mesh.

## 4. Istio Key Components:

- Istiod: The control plane managing traffic and configurations.
- Ingress Gateway: Manages external (North-South) traffic.
- Egress Gateway: Manages outgoing traffic.

## 5. Istio Workflow

1. Deploy Services with Sidecar Proxy (Envoy):
    - Each microservice is paired with an Envoy proxy via sidecar injection.
2. Traffic Interception:
    - All incoming and outgoing traffic flows through Envoy.
3. Control Plane Directives:
    - Pilot pushes routing, security, and policy rules to Envoy proxies.
4. Observability & Security:
    Envoy proxies collect telemetry data, enforce mTLS, and report logs.

## 6. How Istio Uses Dynamic Admission Controllers for Sidecar Injection

**How Istio Uses Admission Webhooks:**
- Pod Creation Request → Kubernetes API Server.
- Server → Mutating Admission Webhook.
- Webhook → Istio Sidecar Injection Logic.
- Istio Injects Sidecar Container into Pod.
- Request Persisted in etcd.

**Custom Resource Example:**
- Kubernetes uses a MutatingWebhookConfiguration to identify which webhook to call.
  ```yaml
  apiVersion: admissionregistration.k8s.io/v1
  kind: MutatingWebhookConfiguration
  webhooks:
  - name: istio-sidecar-injector.istio.io
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
  ```

## 7. Observability with Kiali

- Observability in a service mesh refers to the ability to monitor, trace, and analyze the behavior and performance of services. 
- It includes features like distributed tracing, metrics collection, and logging to provide insights into the health and operation of the system.
- Kiali provides a visual dashboard for monitoring service-to-service traffic.
- Integrates with Prometheus for metrics collection.

