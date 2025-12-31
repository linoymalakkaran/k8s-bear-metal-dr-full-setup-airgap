# Airgap Installation Guide for Rancher Kubernetes

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Setting Up Private Registry](#setting-up-private-registry)
5. [Downloading Images and Artifacts](#downloading-images-and-artifacts)
6. [Transferring to Airgap Environment](#transferring-to-airgap-environment)
7. [Installing Rancher in Airgap](#installing-rancher-in-airgap)
8. [Installing RKE2 in Airgap](#installing-rke2-in-airgap)
9. [Helm Charts Repository](#helm-charts-repository)

## Overview

An airgap installation is required when Kubernetes infrastructure has no direct internet access due to security policies or regulatory requirements. This guide covers complete offline installation of Rancher and RKE2.

### What is Airgap?
```
┌─────────────────────────────────────────┐
│     Internet-Connected Environment      │
│  (Used for downloading artifacts)       │
│                                         │
│  - Download container images            │
│  - Download binaries                    │
│  - Download Helm charts                 │
│  - Package everything                   │
└────────────────┬────────────────────────┘
                 │
                 │ Physical Transfer
                 │ (USB, DVD, Secure FTP)
                 │
                 ▼
┌─────────────────────────────────────────┐
│    Airgap Environment (No Internet)     │
│                                         │
│  - Private container registry           │
│  - Private Helm repository              │
│  - Internal Kubernetes cluster          │
│  - All dependencies self-contained      │
└─────────────────────────────────────────┘
```

## Prerequisites

### Internet-Connected System (Bastion/Jump Box)
```yaml
requirements:
  os: "Linux (RHEL 8/Ubuntu 20.04+)"
  disk_space: "200 GB minimum"
  tools:
    - docker or podman
    - helm (v3.x)
    - kubectl
    - rancher CLI
    - wget/curl
  
  purpose: "Download and package all artifacts"
```

### Airgap Environment Requirements
```yaml
airgap_environment:
  private_registry:
    cpu: "4 vCPU"
    memory: "8 GB RAM"
    disk: "500 GB SSD"
    software: "Harbor or Docker Registry"
  
  artifact_storage:
    size: "100 GB minimum"
    type: "NFS or local storage"
```

## Architecture

### Airgap Infrastructure Components

```
┌──────────────────────────────────────────────────────────────┐
│                  Airgap Environment                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Private Container Registry (Harbor)            │ │
│  │         https://registry.k8s.internal                  │ │
│  │                                                        │ │
│  │  - rancher/rancher:v2.8.x                            │ │
│  │  - rancher/rke2-runtime:v1.28.x                      │ │
│  │  - All Kubernetes images                              │ │
│  │  - Application images                                 │ │
│  └────────────────┬───────────────────────────────────────┘ │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────────┐ │
│  │         Private Helm Chart Repository                  │ │
│  │         https://charts.k8s.internal                    │ │
│  │                                                        │ │
│  │  - Rancher Helm charts                                │ │
│  │  - Cert-manager charts                                │ │
│  │  - Monitoring charts                                  │ │
│  └────────────────┬───────────────────────────────────────┘ │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────────┐ │
│  │         Binary Repository / Web Server                 │ │
│  │         https://binaries.k8s.internal                  │ │
│  │                                                        │ │
│  │  - RKE2 installers                                    │ │
│  │  - containerd                                         │ │
│  │  - kubectl, helm binaries                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Setting Up Private Registry

### Option 1: Harbor (Recommended for Production)

#### Install Harbor
```bash
#!/bin/bash
# File: scripts/setup/install-harbor.sh

# Harbor - Enterprise-grade private registry
# More features: RBAC, scanning, replication

HARBOR_VERSION="2.9.0"
HARBOR_FQDN="registry.k8s.internal"
HARBOR_DATA_DIR="/data/harbor"

echo "Installing Harbor v${HARBOR_VERSION}..."

# Create data directory
mkdir -p ${HARBOR_DATA_DIR}

# Download Harbor offline installer (on internet-connected machine)
# wget https://github.com/goharbor/harbor/releases/download/v${HARBOR_VERSION}/harbor-offline-installer-v${HARBOR_VERSION}.tgz

# Extract Harbor
tar -xzf harbor-offline-installer-v${HARBOR_VERSION}.tgz
cd harbor

# Configure Harbor
cat > harbor.yml <<EOF
# Harbor configuration
hostname: ${HARBOR_FQDN}

# HTTP settings
http:
  port: 80

# HTTPS settings
https:
  port: 443
  certificate: /data/cert/server.crt
  private_key: /data/cert/server.key

# Harbor admin password
harbor_admin_password: YourSecurePassword123!

# Data directory
data_volume: ${HARBOR_DATA_DIR}

# Database settings
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

# Storage backend (local filesystem)
storage_service:
  filesystem:
    rootdirectory: ${HARBOR_DATA_DIR}/registry

# Logging
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor

# Automatically create projects
# Create 'library' and 'rancher' projects
EOF

# Generate self-signed certificate (or use your CA cert)
mkdir -p /data/cert
openssl req -newkey rsa:4096 -nodes -sha256 -keyout /data/cert/server.key \
  -x509 -days 365 -out /data/cert/server.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=${HARBOR_FQDN}"

# Install Harbor
./install.sh --with-trivy --with-chartmuseum

echo "Harbor installation complete!"
echo "Access Harbor at: https://${HARBOR_FQDN}"
echo "Default credentials: admin / YourSecurePassword123!"
```

#### Create Harbor Projects
```bash
#!/bin/bash
# File: scripts/setup/configure-harbor-projects.sh

HARBOR_URL="https://registry.k8s.internal"
HARBOR_USER="admin"
HARBOR_PASS="YourSecurePassword123!"

# Create projects in Harbor
# Projects: rancher, rke2, monitoring, applications

curl -u "${HARBOR_USER}:${HARBOR_PASS}" -X POST \
  "${HARBOR_URL}/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "rancher",
    "public": false
  }'

curl -u "${HARBOR_USER}:${HARBOR_PASS}" -X POST \
  "${HARBOR_URL}/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "rke2",
    "public": false
  }'

curl -u "${HARBOR_USER}:${HARBOR_PASS}" -X POST \
  "${HARBOR_URL}/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "monitoring",
    "public": false
  }'

echo "Harbor projects created successfully"
```

### Option 2: Docker Registry (Lightweight)

```bash
#!/bin/bash
# File: scripts/setup/install-docker-registry.sh

# Simple Docker Registry for smaller deployments

REGISTRY_PORT=5000
REGISTRY_DATA_DIR="/data/registry"

mkdir -p ${REGISTRY_DATA_DIR}

# Run Docker Registry
docker run -d \
  --name registry \
  --restart=always \
  -p ${REGISTRY_PORT}:5000 \
  -v ${REGISTRY_DATA_DIR}:/var/lib/registry \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:2

echo "Docker Registry running on port ${REGISTRY_PORT}"
```

## Downloading Images and Artifacts

### Download Rancher Images

```bash
#!/bin/bash
# File: scripts/airgap/01-download-rancher-images.sh

# Run this script on an internet-connected machine

RANCHER_VERSION="2.8.0"
CERT_MANAGER_VERSION="1.13.1"
OUTPUT_DIR="/airgap-artifacts"

mkdir -p ${OUTPUT_DIR}/images

echo "Downloading Rancher ${RANCHER_VERSION} airgap images..."

# Download Rancher images list
curl -sfL https://github.com/rancher/rancher/releases/download/v${RANCHER_VERSION}/rancher-images.txt \
  -o ${OUTPUT_DIR}/rancher-images.txt

# Download Rancher image save script
curl -sfL https://github.com/rancher/rancher/releases/download/v${RANCHER_VERSION}/rancher-save-images.sh \
  -o ${OUTPUT_DIR}/rancher-save-images.sh

chmod +x ${OUTPUT_DIR}/rancher-save-images.sh

# Download images and save to tar
${OUTPUT_DIR}/rancher-save-images.sh \
  --image-list ${OUTPUT_DIR}/rancher-images.txt \
  --images ${OUTPUT_DIR}/images/rancher-images.tar.gz

echo "Rancher images downloaded successfully"

# Download cert-manager images
echo "Downloading cert-manager images..."

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm pull jetstack/cert-manager \
  --version v${CERT_MANAGER_VERSION} \
  --untar \
  --untardir ${OUTPUT_DIR}/charts

# Extract cert-manager images
grep -oP 'repository:.*|tag:.*' ${OUTPUT_DIR}/charts/cert-manager/values.yaml \
  > ${OUTPUT_DIR}/cert-manager-images.txt

echo "Artifacts download complete at ${OUTPUT_DIR}"
```

### Download RKE2 Artifacts

```bash
#!/bin/bash
# File: scripts/airgap/02-download-rke2-artifacts.sh

RKE2_VERSION="v1.28.5+rke2r1"
OUTPUT_DIR="/airgap-artifacts/rke2"

mkdir -p ${OUTPUT_DIR}

echo "Downloading RKE2 ${RKE2_VERSION} artifacts..."

# Download RKE2 install script
curl -sfL https://get.rke2.io -o ${OUTPUT_DIR}/install.sh
chmod +x ${OUTPUT_DIR}/install.sh

# Download RKE2 binary (AMD64)
curl -sfL https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-amd64.tar.gz \
  -o ${OUTPUT_DIR}/rke2.linux-amd64.tar.gz

# Download RKE2 images for airgap
curl -sfL https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-amd64.tar.zst \
  -o ${OUTPUT_DIR}/rke2-images.linux-amd64.tar.zst

# Download additional images for ingress, storage, etc.
curl -sfL https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images-core.linux-amd64.tar.zst \
  -o ${OUTPUT_DIR}/rke2-images-core.linux-amd64.tar.zst

curl -sfL https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images-canal.linux-amd64.tar.zst \
  -o ${OUTPUT_DIR}/rke2-images-canal.linux-amd64.tar.zst

# Download SHA256 checksums
curl -sfL https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/sha256sum-amd64.txt \
  -o ${OUTPUT_DIR}/sha256sum-amd64.txt

echo "RKE2 artifacts downloaded successfully"
```

### Download Helm and kubectl

```bash
#!/bin/bash
# File: scripts/airgap/03-download-tools.sh

OUTPUT_DIR="/airgap-artifacts/tools"
mkdir -p ${OUTPUT_DIR}

# Download Helm
HELM_VERSION="3.13.2"
curl -sfL https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz \
  -o ${OUTPUT_DIR}/helm-v${HELM_VERSION}-linux-amd64.tar.gz

# Download kubectl
KUBECTL_VERSION="1.28.5"
curl -sfLO "https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
  -o ${OUTPUT_DIR}/kubectl

chmod +x ${OUTPUT_DIR}/kubectl

echo "Tools downloaded successfully"
```

### Download Helm Charts

```bash
#!/bin/bash
# File: scripts/airgap/04-download-helm-charts.sh

OUTPUT_DIR="/airgap-artifacts/charts"
mkdir -p ${OUTPUT_DIR}

# Add repositories
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Download Rancher chart
helm pull rancher-latest/rancher --destination ${OUTPUT_DIR}

# Download cert-manager
helm pull jetstack/cert-manager --destination ${OUTPUT_DIR}

# Download monitoring charts
helm pull prometheus-community/kube-prometheus-stack --destination ${OUTPUT_DIR}
helm pull prometheus-community/prometheus --destination ${OUTPUT_DIR}
helm pull grafana/grafana --destination ${OUTPUT_DIR}

# Download Longhorn (storage)
helm repo add longhorn https://charts.longhorn.io
helm pull longhorn/longhorn --destination ${OUTPUT_DIR}

echo "Helm charts downloaded to ${OUTPUT_DIR}"
```

## Transferring to Airgap Environment

### Create Transfer Package

```bash
#!/bin/bash
# File: scripts/airgap/05-create-transfer-package.sh

OUTPUT_DIR="/airgap-artifacts"
PACKAGE_NAME="k8s-airgap-bundle-$(date +%Y%m%d).tar.gz"

echo "Creating airgap transfer package..."

# Create comprehensive package
tar -czf ${PACKAGE_NAME} \
  ${OUTPUT_DIR}/images \
  ${OUTPUT_DIR}/rke2 \
  ${OUTPUT_DIR}/charts \
  ${OUTPUT_DIR}/tools \
  ${OUTPUT_DIR}/*.txt \
  ${OUTPUT_DIR}/*.sh

# Calculate checksum
sha256sum ${PACKAGE_NAME} > ${PACKAGE_NAME}.sha256

echo "Package created: ${PACKAGE_NAME}"
echo "Size: $(du -h ${PACKAGE_NAME} | cut -f1)"
echo "Checksum: $(cat ${PACKAGE_NAME}.sha256)"
echo ""
echo "Transfer this file to your airgap environment via:"
echo "  - USB drive"
echo "  - Secure file transfer"
echo "  - Physical media"
```

### Verify Transfer Package

```bash
#!/bin/bash
# File: scripts/airgap/06-verify-package.sh

PACKAGE_NAME=$1

if [ -z "$PACKAGE_NAME" ]; then
  echo "Usage: $0 <package-file.tar.gz>"
  exit 1
fi

echo "Verifying package integrity..."

# Verify checksum
sha256sum -c ${PACKAGE_NAME}.sha256

if [ $? -eq 0 ]; then
  echo "✓ Package verification successful"
  
  # Extract package
  tar -xzf ${PACKAGE_NAME}
  echo "✓ Package extracted successfully"
else
  echo "✗ Package verification failed!"
  exit 1
fi
```

## Installing Rancher in Airgap

### Load Images to Private Registry

```bash
#!/bin/bash
# File: scripts/airgap/07-load-images-to-registry.sh

# Run this in airgap environment after transferring artifacts

PRIVATE_REGISTRY="registry.k8s.internal"
IMAGES_TAR="/airgap-artifacts/images/rancher-images.tar.gz"
IMAGE_LIST="/airgap-artifacts/rancher-images.txt"

echo "Loading images to private registry ${PRIVATE_REGISTRY}..."

# Method 1: Using rancher-load-images.sh script
curl -sfL https://github.com/rancher/rancher/releases/download/v2.8.0/rancher-load-images.sh \
  -o rancher-load-images.sh

chmod +x rancher-load-images.sh

# Load images to registry
./rancher-load-images.sh \
  --image-list ${IMAGE_LIST} \
  --registry ${PRIVATE_REGISTRY}

echo "Images loaded successfully to ${PRIVATE_REGISTRY}"
```

### Configure Private Registry on Nodes

```bash
#!/bin/bash
# File: scripts/airgap/08-configure-private-registry.sh

# Configure containerd to use private registry
# Run on all Kubernetes nodes

PRIVATE_REGISTRY="registry.k8s.internal"

mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/registries.yaml <<EOF
mirrors:
  docker.io:
    endpoint:
      - "https://${PRIVATE_REGISTRY}"
  registry.k8s.internal:
    endpoint:
      - "https://${PRIVATE_REGISTRY}"

configs:
  "${PRIVATE_REGISTRY}":
    auth:
      username: admin
      password: YourSecurePassword123!
    tls:
      insecure_skip_verify: false
      ca_file: /etc/rancher/rke2/ca.crt
EOF

# Copy registry CA certificate
cp /path/to/registry-ca.crt /etc/rancher/rke2/ca.crt

echo "Private registry configured on this node"
```

### Install Rancher with Helm

```bash
#!/bin/bash
# File: scripts/airgap/09-install-rancher-airgap.sh

PRIVATE_REGISTRY="registry.k8s.internal"
RANCHER_VERSION="2.8.0"
RANCHER_HOSTNAME="rancher.k8s.internal"

# Install cert-manager first
kubectl create namespace cert-manager

helm install cert-manager /airgap-artifacts/charts/cert-manager-v1.13.1.tgz \
  --namespace cert-manager \
  --set image.repository=${PRIVATE_REGISTRY}/jetstack/cert-manager-controller \
  --set webhook.image.repository=${PRIVATE_REGISTRY}/jetstack/cert-manager-webhook \
  --set cainjector.image.repository=${PRIVATE_REGISTRY}/jetstack/cert-manager-cainjector \
  --set startupapicheck.image.repository=${PRIVATE_REGISTRY}/jetstack/cert-manager-ctl \
  --set installCRDs=true

# Wait for cert-manager
kubectl wait --for=condition=Available --timeout=300s \
  --namespace cert-manager \
  deployment/cert-manager

# Install Rancher
kubectl create namespace cattle-system

helm install rancher /airgap-artifacts/charts/rancher-${RANCHER_VERSION}.tgz \
  --namespace cattle-system \
  --set hostname=${RANCHER_HOSTNAME} \
  --set rancherImage=${PRIVATE_REGISTRY}/rancher/rancher \
  --set systemDefaultRegistry=${PRIVATE_REGISTRY} \
  --set useBundledSystemChart=true \
  --set bootstrapPassword=admin

echo "Rancher installation in airgap complete!"
echo "Access Rancher at: https://${RANCHER_HOSTNAME}"
```

## Installing RKE2 in Airgap

### Prepare RKE2 Airgap Directory

```bash
#!/bin/bash
# File: scripts/airgap/10-prepare-rke2-airgap.sh

RKE2_VERSION="v1.28.5+rke2r1"
AIRGAP_DIR="/var/lib/rancher/rke2/agent/images"

# Create airgap directory
mkdir -p ${AIRGAP_DIR}

# Copy RKE2 images
cp /airgap-artifacts/rke2/rke2-images.linux-amd64.tar.zst ${AIRGAP_DIR}/
cp /airgap-artifacts/rke2/rke2-images-core.linux-amd64.tar.zst ${AIRGAP_DIR}/
cp /airgap-artifacts/rke2/rke2-images-canal.linux-amd64.tar.zst ${AIRGAP_DIR}/

echo "RKE2 airgap images prepared"
```

### Install RKE2 Server (Master)

```bash
#!/bin/bash
# File: scripts/airgap/11-install-rke2-server-airgap.sh

# Run on master nodes

# Copy RKE2 binary
tar -xzf /airgap-artifacts/rke2/rke2.linux-amd64.tar.gz -C /usr/local

# Create RKE2 config
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
write-kubeconfig-mode: "0644"
tls-san:
  - "api.k8s.internal"
  - "192.168.1.100"

# Use private registry
system-default-registry: "registry.k8s.internal"

# Disable default components if needed
# disable: traefik

# Cluster CIDR
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
EOF

# Install RKE2 server
INSTALL_RKE2_ARTIFACT_PATH=/airgap-artifacts/rke2 \
  sh /airgap-artifacts/rke2/install.sh

# Enable and start RKE2
systemctl enable rke2-server.service
systemctl start rke2-server.service

# Wait for server to be ready
sleep 30

# Get token for worker nodes
cat /var/lib/rancher/rke2/server/node-token

echo "RKE2 server installed in airgap mode"
```

## Helm Charts Repository

### Setup ChartMuseum

```bash
#!/bin/bash
# File: scripts/airgap/12-setup-chartmuseum.sh

# ChartMuseum for hosting Helm charts

CHARTMUSEUM_PORT=8080
CHARTS_DIR="/data/charts"

mkdir -p ${CHARTS_DIR}

# Copy all downloaded charts
cp /airgap-artifacts/charts/*.tgz ${CHARTS_DIR}/

# Run ChartMuseum
docker run -d \
  --name chartmuseum \
  --restart=always \
  -p ${CHARTMUSEUM_PORT}:8080 \
  -e DEBUG=1 \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v ${CHARTS_DIR}:/charts \
  ghcr.io/helm/chartmuseum:v0.16.0

echo "ChartMuseum running at http://localhost:${CHARTMUSEUM_PORT}"
```

### Use Private Helm Repository

```bash
#!/bin/bash
# File: scripts/airgap/13-configure-helm-repo.sh

CHART_REPO_URL="http://charts.k8s.internal:8080"

# Add private chart repository
helm repo add private-charts ${CHART_REPO_URL}
helm repo update

# Search charts
helm search repo private-charts

# Install from private repository
helm install my-app private-charts/application-chart

echo "Helm configured to use private repository"
```

## Validation and Troubleshooting

```bash
#!/bin/bash
# File: scripts/airgap/14-validate-airgap.sh

echo "=== Airgap Installation Validation ==="

# Check private registry
echo "Checking private registry..."
curl -k https://registry.k8s.internal/v2/_catalog

# Check RKE2 status
echo "Checking RKE2 status..."
systemctl status rke2-server

# Check Rancher pods
echo "Checking Rancher pods..."
kubectl get pods -n cattle-system

# Verify no internet connectivity (should fail)
echo "Verifying airgap (should not reach internet)..."
curl -m 5 https://google.com && echo "WARNING: Internet access detected!" || echo "✓ Properly airgapped"

echo "=== Validation Complete ==="
```

## Next Steps

After airgap installation:
1. Configure backup strategy: [Backup & Restore](07-backup-restore.md)
2. Setup monitoring: [Monitoring](06-monitoring-observability.md)
3. Plan DR: [Disaster Recovery](05-disaster-recovery.md)
