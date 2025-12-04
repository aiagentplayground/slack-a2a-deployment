# Building a Kind Kagent Cluster

This directory contains everything you need to create a local Kubernetes cluster using Kind (Kubernetes in Docker) configured specifically for Kagent development and deployment.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration Details](#configuration-details)
- [Installing Kagent](#installing-kagent)
- [Troubleshooting](#troubleshooting)
- [Advanced Configuration](#advanced-configuration)

## Overview

This setup creates a fully functional Kubernetes cluster on your local machine with:

- **Kind cluster** named `kagent-demo` with 1 control plane and 3 worker nodes
- **Local Docker registry** on port 5001
- **MetalLB** for LoadBalancer services (IP range: 172.16.10.150-172.16.10.200)
- **Port forwarding** for HTTP (80), HTTPS (443), and registry (5001)
- **Network connectivity** between Kind cluster and local registry

## Prerequisites

### Required Tools

Install the following tools before proceeding:

```bash
# Docker Desktop (or equivalent)
# Download from: https://www.docker.com/products/docker-desktop

# Install core Kubernetes tools
brew install kubectl helm kind cloud-provider-kind
```

> **Note**: `cloud-provider-kind` provides LoadBalancer support for Kind clusters by automatically assigning IP addresses to LoadBalancer services.

### Start cloud-provider-kind

Before creating the cluster, start the cloud-provider-kind service (requires sudo):

```bash
# Run in a separate terminal - keep this running
sudo cloud-provider-kind

# This process must remain running to provide LoadBalancer IPs
```

### Verify Installation

```bash
docker --version
kubectl version --client
helm version
kind version
cloud-provider-kind --version
```

### Environment Variables

Set up your AI provider API keys:

```bash
# Required: Anthropic API key
export ANTHROPIC_API_KEY="your-anthropic-api-key-here"

# Optional: xAI API key for Grok integration
export XAI_API_KEY="your-xai-api-key-here"
```

Add these to your shell profile (`~/.zshrc` or `~/.bashrc`) to persist them:

```bash
echo 'export ANTHROPIC_API_KEY="your-key"' >> ~/.zshrc
source ~/.zshrc
```

## Quick Start

### Step 1: Create the Cluster

```bash
cd build
./setup-kind-cluster.sh
```

**What this does:**
1. Creates a Docker network for Kind and registry communication
2. Starts a local Docker registry on port 5001
3. Creates a Kind cluster with the configuration from `kind-config.yaml`
4. Connects the registry to the cluster network
5. Configures the cluster to use the local registry

**Expected output:**
```
Creating cluster "kagent-demo" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kagent-demo"
```

### Step 2: Verify the Cluster

```bash
# Check cluster is running
kubectl cluster-info

# Check nodes are ready
kubectl get nodes

# Verify registry is accessible
curl http://localhost:5001/v2/_catalog
```

Expected output:
```
NAME                        STATUS   ROLES           AGE   VERSION
kagent-demo-control-plane   Ready    control-plane   1m    v1.27.3
kagent-demo-worker          Ready    <none>          1m    v1.27.3
kagent-demo-worker2         Ready    <none>          1m    v1.27.3
kagent-demo-worker3         Ready    <none>          1m    v1.27.3
```

## Configuration Details

### kind-config.yaml

The cluster configuration includes:

#### Container Images
```yaml
nodes:
  - role: control-plane
    image: kindest/node:v1.31.0@sha256:53df588e04085fd41ae12de0c3fe4c72f7013bba32a20e7325357a1ac94ba865
```

#### Port Mappings
```yaml
extraPortMappings:
  - containerPort: 80     # HTTP traffic
    hostPort: 80
  - containerPort: 443    # HTTPS traffic
    hostPort: 443
  - containerPort: 5001   # Docker registry
    hostPort: 5001
```

#### Registry Configuration
```yaml
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5001"]
          endpoint = ["http://kind-registry:5001"]
```

### setup-kind-cluster.sh

The setup script performs the following steps:

1. **Create Docker network** (`kind`)
2. **Start local registry** (if not already running)
3. **Create Kind cluster** using `kind-config.yaml`
4. **Connect registry to network**
5. **Configure registry as insecure** in containerd

## Installing Kagent

After the cluster is ready, install Kagent:

### Step 1: Install Kagent CRDs

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace
```

### Step 2: Install Kagent with Anthropic

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --set providers.default=anthropic \
    --set providers.anthropic.model="claude-3-5-sonnet-20241022" \
    --set providers.anthropic.apiKey=$ANTHROPIC_API_KEY
```

### Step 3: Expose Kagent UI

```bash
# Change service type to LoadBalancer
kubectl patch svc kagent-ui -n kagent -p '{"spec": {"type": "LoadBalancer"}}'

# Get the LoadBalancer IP
kubectl get svc kagent-ui -n kagent

# Access the UI (typically http://172.16.10.150:8080)
```

### Step 4: (Optional) Configure Additional Providers

#### xAI/Grok Provider

```bash
# Create secret for xAI API key
kubectl create secret generic kagent-xai-provider -n kagent \
    --from-literal PROVIDER_API_KEY=$XAI_API_KEY

# Apply ModelConfig
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: xai-grok-config
  namespace: kagent
spec:
  apiKeySecret: kagent-xai-provider
  apiKeySecretKey: PROVIDER_API_KEY
  model: grok-4-fast-reasoning
  provider: OpenAI
  openAI:
    baseUrl: "https://api.x.ai/v1"
EOF
```

### Step 5: Verify Installation

```bash
# Check all pods are running
kubectl get pods -n kagent

# Check services
kubectl get svc -n kagent

# View logs
kubectl logs -n kagent -l app=kagent
```

## Troubleshooting

### Cluster Creation Issues

#### Error: "cluster already exists"

```bash
# Delete the existing cluster
kind delete cluster --name kagent-demo

# Re-run the setup script
./setup-kind-cluster.sh
```

#### Error: "context deadline exceeded"

This usually means Docker is not running properly:

```bash
# Restart Docker Desktop
# Then try again
./setup-kind-cluster.sh
```

### Registry Issues

#### Error: "connection refused" when accessing registry

```bash
# Check if registry is running
docker ps | grep registry

# If not running, start it manually
docker run -d --restart=always -p 5001:5000 --name kind-registry registry:2

# Connect it to the kind network
docker network connect kind kind-registry
```

#### Error: "http: server gave HTTP response to HTTPS client"

The registry is configured as insecure. Verify the containerd patch:

```bash
# Get containerd config from a node
docker exec kagent-demo-control-plane cat /etc/containerd/config.toml | grep -A 10 registry
```

### Kagent Installation Issues

#### Error: "no matches for kind ModelConfig"

The CRDs are not installed:

```bash
# Install CRDs first
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace
```

#### Error: "anthropic.apiKey is required"

Make sure the environment variable is set:

```bash
echo $ANTHROPIC_API_KEY

# If empty, set it
export ANTHROPIC_API_KEY="your-key-here"
```

### LoadBalancer Issues

#### LoadBalancer IP shows "pending"

MetalLB might not be installed:

```bash
# Check if MetalLB is running
kubectl get pods -n metallb-system

# If not, install it
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Configure IP pool
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.16.10.150-172.16.10.200
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-l2
  namespace: metallb-system
EOF
```

## Advanced Configuration

### Customizing the Cluster

#### Change Number of Worker Nodes

Edit `kind-config.yaml`:

```yaml
nodes:
  - role: control-plane
    # ... (keep existing config)
  - role: worker
  - role: worker
  # Add more worker nodes as needed
```

#### Change Port Mappings

Add more port mappings in `kind-config.yaml`:

```yaml
extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
    hostPort: 443
  - containerPort: 8080    # Add custom port
    hostPort: 8080
```

#### Change LoadBalancer IP Range

Modify the MetalLB configuration:

```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.16.10.100-172.16.10.250  # Your custom range
EOF
```

### Using a Different Registry

If you want to use an external registry (like Docker Hub):

1. **Login to Docker Hub:**
```bash
docker login
```

2. **Tag and push images:**
```bash
docker tag my-image:latest username/my-image:latest
docker push username/my-image:latest
```

3. **Update Kubernetes manifests** to use the external registry URL.

### Cluster Cleanup

```bash
# Delete the entire cluster
kind delete cluster --name kagent-demo

# Stop and remove the local registry
docker stop kind-registry
docker rm kind-registry

# Remove the kind network
docker network rm kind
```

## Next Steps

Now that your cluster is ready:

1. **Deploy MCP servers** - See [mcp-servers/README.md](../mcp-servers/README.md)
2. **Publish to Agent Registry** - See [registry/README.md](../registry/README.md)
3. **Create custom agents** - Check out the Kagent documentation

## Additional Resources

- [Kind Documentation](https://kind.sigs.k8s.io)
- [Kagent Documentation](https://docs.kagent.dev)
- [MetalLB Documentation](https://metallb.universe.tf)
- [Helm Documentation](https://helm.sh/docs)

---

**Need Help?**

- Check [Troubleshooting](#troubleshooting) above
- Review [GitHub Issues](https://github.com/sebbycorp/kagent-helm-kind/issues)
- Join the discussion in [GitHub Discussions](https://github.com/sebbycorp/kagent-helm-kind/discussions)
