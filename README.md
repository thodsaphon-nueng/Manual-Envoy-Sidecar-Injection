# Manual Envoy Sidecar Injection

## Overview

This repo teaches you how to manually inject an Envoy sidecar proxy to understand how Istio works under the hood. You'll learn about iptables redirection, traffic flow, and Envoy configuration by implementing the sidecar pattern manually.

### What You'll Learn

- Deploy Envoy proxy as a sidecar container manually
- Configure Envoy proxy directly
- Setup iptables rules for traffic interception
- Understand detailed traffic flow in a service mesh
- Debug and modify Envoy configurations

## Prerequisites

- Kubernetes cluster (minikube recommended)
- kubectl CLI tool
- Basic understanding of containers and networking

## Architecture

```
┌─────────────────────────────────────┐
│             Pod                     │
│  ┌─────────────┐  ┌──────────────┐  │
│  │   Nginx     │  │    Envoy     │  │
│  │   App       │  │   Sidecar    │  │
│  │   (port 80) │  │  (port 15001)│  │
│  └─────────────┘  └──────────────┘  │
│         │                 │         │
│         └─────────────────┘         │
│              iptables               │
└─────────────────────────────────────┘
```

Traffic Flow:
1. External request comes to Pod
2. iptables redirects traffic to Envoy (port 15001)
3. Envoy processes the request (logging, routing, etc.)
4. Envoy forwards to the actual application (port 80/8080)

## Lab Setup

### Step 1: Start Minikube

```bash
minikube start --driver=docker --container-runtime=containerd
```

### Step 2: Apply Configuration Files

Deploy the necessary configuration files in order:

```bash
# Apply iptables initialization configuration
kubectl apply -f iptables-init.yaml

# Apply Envoy proxy configuration
kubectl apply -f envoy-config.yaml

# Apply Nginx application configuration
kubectl apply -f nginx-config.yaml

# Deploy the application with manual sidecar
kubectl apply -f manual-sidecar-proxy-app.yaml
```

### Step 3: Verify Deployment

Check if all pods are running:

```bash
kubectl get pods
```

Get detailed information about the sidecar-enabled pod:

```bash
kubectl describe pod -l version=with-sidecar
```
<img width="1902" height="967" alt="Image" src="https://github.com/user-attachments/assets/c82cb464-e99a-4452-83c4-06d3485fea7b" />

You should see:
- An init container for iptables setup
- The main nginx application container
- The Envoy sidecar proxy container

## Testing the Setup

### Test 1: Initial Traffic Routing

Create a test client and verify traffic flows through Envoy:

```bash
kubectl run test-client --image=curlimages/curl --rm -it -- sh
```

Inside the test client, make a request:

```bash
curl simple-app-sidecar-service.default.svc.cluster.local
```
<img width="1897" height="240" alt="Image" src="https://github.com/user-attachments/assets/93bda31d-63db-482c-a128-505dd709b906" />

This should work, but we can test if iptables and Envoy are actually working. We will change the port configuration to verify the traffic routing.

### Test 2: Fix Port Configuration and Reload

The initial configuration has the old port mapping. Let's change it:

1. Edit the `envoy-config.yaml` file and change:

```yaml
- lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 127.0.0.1
            port_value: 8080  # Changed from 80 to 8080
```

2. Apply the updated configuration:

```bash
kubectl apply -f envoy-config.yaml
```

3. Restart the application to pick up new config:

```bash
kubectl delete -f manual-sidecar-proxy-app.yaml
kubectl apply -f manual-sidecar-proxy-app.yaml
```

### Test 3: Verify Fixed Configuration

Test again with the corrected configuration:

```bash
kubectl run test-client --image=curlimages/curl --rm -it -- sh
curl simple-app-sidecar-service.default.svc.cluster.local
```
<img width="1898" height="416" alt="Image" src="https://github.com/user-attachments/assets/c74cc347-0854-4cce-b7a6-e890cee81a1f" />

Now you should see traffic properly routed to port 8080 (app2 container).

## Understanding the Components

### 1. iptables-init.yaml
- Contains init container that sets up iptables rules
- Redirects incoming traffic to Envoy proxy port (15001)
- Redirects outgoing traffic through Envoy for egress control

### 2. envoy-config.yaml
- ConfigMap containing Envoy proxy configuration
- Defines listeners, clusters, and routing rules
- Specifies upstream services and load balancing

### 3. nginx-config.yaml
- ConfigMap for nginx configuration
- Defines the application behavior
- Sets up the actual service endpoints

### 4. manual-sidecar-proxy-app.yaml
- Main deployment with sidecar pattern
- Contains multiple containers in a single pod:
  - Init container for iptables setup
  - Application container (nginx)
  - Sidecar proxy container (Envoy)

## Key Concepts

### Sidecar Pattern
- Each application pod gets its own proxy instance
- Proxy intercepts all network traffic
- Provides observability, security, and traffic management

### iptables Redirection
- Init container modifies iptables rules
- Transparently redirects traffic without app changes
- Enables proxy to intercept all communication

### Envoy Configuration
- Listeners: Define ports where Envoy accepts connections
- Clusters: Define upstream services to route to
- Routes: Define routing rules and policies

## Troubleshooting

### Common Issues

1. **Pod not starting**: Check if all ConfigMaps are applied first
2. **Connection refused**: Verify port configurations in Envoy config
3. **Traffic not intercepted**: Check iptables rules in init container
4. **Wrong upstream**: Verify cluster configuration in Envoy

### Debug Commands

```bash
# Check pod logs
kubectl logs <pod-name> -c envoy-sidecar
kubectl logs <pod-name> -c nginx-app

# Check iptables rules
kubectl exec <pod-name> -- iptables -t nat -L

# Check Envoy admin interface
kubectl port-forward <pod-name> 15000:15000
# Visit http://localhost:15000
```
