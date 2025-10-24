# Homelab Kubernetes Cluster

A Kubernetes homelab running on a Lenovo T470, demonstrating GitOps practices, secret management, and modern cloud-native application deployment patterns.

## Infrastructure

### Hardware
- **Device**: Lenovo T470 Laptop
- **OS**: Ubuntu Server
- **Kubernetes Distribution**: k3s (lightweight Kubernetes)
- **Architecture**: Single-node cluster

### Core Technologies
- **GitOps**: Flux CD for automated deployments
- **Secret Management**: SOPS with age encryption
- **Ingress**: Nginx Ingress Controller
- **Tunnel**: Cloudflare Tunnels for secure external access
- **Monitoring**: Prometheus & Grafana stack

##  Table of Contents
- [Applications](#applications)
- [Key Features](#key-features)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [GitOps Workflow](#gitops-workflow)
- [Secret Management](#secret-management)
- [Networking](#networking)
- [Project Structure](#project-structure)
- [Kubernetes Concepts Demonstrated](#kubernetes-concepts-demonstrated)

##  Applications

Currently deployed applications:

| Application | Description | Access Method |
|------------|-------------|---------------|
| **Prometheus** | Metrics collection and monitoring | Ingress |
| **Grafana** | Metrics visualization and dashboards | Ingress |
| **Linkding** | Bookmark manager | Cloudflare Tunnel |
| **Audiobookshelf** | Audiobook and podcast server | Cloudflare Tunnel |

*More applications coming soon!*

## Key Features

### GitOps Workflow
- **Declarative Infrastructure**: All Kubernetes manifests stored in Git
- **Automated Deployments**: Flux CD continuously monitors repository and applies changes
- **Version Control**: Full audit trail of all infrastructure changes
- **No Manual kubectl**: All changes committed to Git, then automatically applied

### Security
- **Encrypted Secrets**: Sensitive data encrypted with SOPS/age before committing to Git
- **Secure External Access**: Cloudflare Tunnels for secure ingress without exposing ports

### Kubernetes Fundamentals
- **Deployments**: Application lifecycle management
- **Services**: Internal networking and service discovery
- **Ingress**: HTTP/HTTPS routing and load balancing
- **Persistent Volume Claims (PVCs)**: Stateful application data persistence
- **Namespaces**: Resource organization and isolation
- **ConfigMaps & Secrets**: Application configuration management

## Prerequisites

- Ubuntu Server (20.04 or newer)
- Minimum 4GB RAM, 2 CPU cores
- 50GB+ storage
- Domain name (for Cloudflare Tunnels)
- Basic understanding of Kubernetes concepts

## Setup

### 1. Install k3s

```bash
# Install k3s (lightweight Kubernetes)
curl -sfL https://get.k3s.io | sh -

# Verify installation
sudo k3s kubectl get nodes

# Set up kubeconfig for non-root user
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

### 2. Install Flux CLI

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify installation
flux --version
```

### 3. Bootstrap Flux

```bash
# Export GitHub token
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=homelab \
  --branch=main \
  --path=./clusters/homelab \
  --personal
```

### 4. Set Up SOPS/age for Secret Encryption

```bash
# Install age
sudo apt install age

# Generate age key
age-keygen -o age.agekey

# Create secret in Kubernetes
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# Configure SOPS (create .sops.yaml in repo root)
```

### 5. Configure Cloudflare Tunnel

```bash
# Install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create tunnelname

# Deploy tunnel as Kubernetes deployment
```

## GitOps Workflow

### How It Works

1. **Commit Changes**: Push Kubernetes manifests to Git repository
2. **Flux Detects**: Flux polls repository every 5 minutes (configurable)
3. **Automatic Sync**: Flux applies changes to the cluster automatically
4. **Reconciliation**: Flux ensures cluster state matches Git state

### Making Changes

```bash
# 1. Edit manifest files locally, example
vim apps/base/linkding/deployment.yaml

# 2. Commit and push
git add apps/base/linkding/deployment.yaml
git commit -m "Update linkding image version"
git push origin main

# 3. Flux automatically applies changes (within 5 minutes)
# Or force immediate reconciliation:
flux reconcile kustomization flux-system --with-source
```

### Benefits
- ✅ Full audit trail of all changes
- ✅ Easy rollbacks (revert Git commit)
- ✅ No direct cluster access needed for deployments
- ✅ Consistent, repeatable deployments

## Secret Management

All secrets are encrypted using SOPS with age before being committed to Git.

### Encrypting Secrets

```bash
# Install sops and age
brew install sops age

# Generate our public and private key
age-keygen -o age.agekey
cat age.agekey
# Store your public key in variable $AGE_PUBLIC

# Generate a test secret yaml file
kubectl create secret generic test-secret \
--from-literal=user=admin \
--from-literal=password=testing \
--dry-run=client \
-o yaml > test-secret.yaml

# Encrypt with SOPS
sops --age=$AGE_PUBLIC --encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml

# Place secret in appropriate pathCommit encrypted file to Git
git add secret.enc.yaml
git commit -m "Add encrypted secret"
git push
```

### How Flux Decrypts

Flux automatically decrypts SOPS-encrypted files using the age key stored in the `sops-age` Kubernetes secret during reconciliation.

```bash
# Taking our key from earlier, we create a sops-age secret
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin

# Add .sops.yaml within cluster

creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: $AGE_PUBLIC

# in the apps.yaml Kustomization file, add under spec:

decryption:
    provider: sops
    secretRef:
      name: sops-age
# References our secret created to decrypt any of our encrypted secrets

```


## Networking

### External Access (Cloudflare Tunnels)
- **Linkding**: `https://ldpi.hasnainshomelab.com
- **Audiobookshelf**: `https://audio.hasnainshomelab.com

### Network Flow
```
Internet → Cloudflare Tunnel → Kubernetes Service → Pod
Internal → Ingress Controller → Service → Pod
```

## Project Structure

Inspiration from Flux repo structure, some recommended reading: 
https://fluxcd.io/flux/guides/repository-structure/


## Kubernetes Concepts Demonstrated

### Core Workloads
- **Deployments**: Managing application replicas and updates
- **StatefulSets**: For stateful applications (if applicable)
- **DaemonSets**: Node-level services

### Configuration
- **ConfigMaps**: Application configuration
- **Secrets**: Sensitive data (encrypted with SOPS)

### Storage
- **PersistentVolumes (PV)**: Cluster storage resources
- **PersistentVolumeClaims (PVC)**: Storage requests by pods
- **StorageClasses**: Dynamic volume provisioning

### Networking
- **Services**: ClusterIP, NodePort, LoadBalancer
- **Ingress**: HTTP/HTTPS routing
- **NetworkPolicies**: Pod-to-pod communication rules

### Advanced Patterns
- **GitOps**: Infrastructure as Code with Flux
- **Secret Management**: SOPS/age encryption
- **Kustomize**: YAML templating and overlays
- **HelmReleases**: Package management via Flux

## Monitoring & Observability

### Prometheus and Grafana
- Scrapes metrics from all cluster components
- Stores time-series data for analysis
- Alerting rules for proactive monitoring
- Pre-built dashboards for cluster health
- Custom dashboards for application metrics
- Visualization of Prometheus data

##  Roadmap

Future additions planned:
- [ ] cert-manager for automatic TLS certificates
- [ ] Multi-node cluster expansion
- [ ] CI/CD pipelines with Github actions

## Learning Resources

- [Flux Documentation](https://fluxcd.io/docs/)
- [SOPS Documentation](https://github.com/mozilla/sops)
- [k3s Documentation](https://docs.k3s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

##  Notes

This homelab serves as a learning platform and portfolio piece demonstrating:
- Production-ready GitOps workflows
- Security best practices for Kubernetes
- Modern cloud-native application deployment
- Infrastructure as Code principles
- Real-world Kubernetes administration
  
## Contact

**GitHub**: [@hwaris347](https://github.com/hwaris347)
**LinkedIn** www.linkedin.com/in/hasnainwaris

---

*This homelab is continuously evolving. Check back for updates and new applications!*
