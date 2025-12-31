# Airgap Installation - Complete Image Manifest

## Table of Contents
1. [Overview](#overview)
2. [Complete Image List](#complete-image-list)
3. [Download Scripts](#download-scripts)
4. [Storage Requirements](#storage-requirements)
5. [Verification Procedures](#verification-procedures)

## Overview

This document provides a **complete manifest** of all container images, binaries, and artifacts required for a fully airgapped Kubernetes cluster installation with Rancher.

### Total Storage Requirements

```yaml
storage_summary:
  rke2_artifacts: "~8 GB"
  container_images: "~35 GB"
  rancher_images: "~5 GB"
  cert_manager_images: "~500 MB"
  monitoring_stack_images: "~10 GB"
  longhorn_images: "~2 GB"
  harbor_registry: "~3 GB"
  helm_charts: "~100 MB"
  
  total_minimum: "~65 GB"
  recommended_with_buffer: "~100 GB"
```

## Complete Image List

### 1. RKE2 Core Images (v1.28.5+rke2r1)

#### RKE2 Binaries

```bash
# File: airgap-artifacts/rke2-binaries.txt

# RKE2 Install Tarball (contains all binaries)
https://github.com/rancher/rke2/releases/download/v1.28.5+rke2r1/rke2.linux-amd64.tar.gz
Size: 32 MB
SHA256: <checksum-from-rke2-releases>

# RKE2 Images Tarball (all container images)
https://github.com/rancher/rke2/releases/download/v1.28.5+rke2r1/rke2-images.linux-amd64.tar.zst
Size: 3.5 GB
SHA256: <checksum>

# RKE2 Install Script
https://get.rke2.io
Size: 50 KB

# SHA256 Checksums File
https://github.com/rancher/rke2/releases/download/v1.28.5+rke2r1/sha256sum-amd64.txt
```

#### RKE2 Container Images (included in rke2-images tarball)

```yaml
# File: airgap-artifacts/rke2-images.yaml
# These are bundled in rke2-images.linux-amd64.tar.zst

rke2_core_images:
  kubernetes_core:
    - image: "rancher/hardened-kubernetes:v1.28.5-rke2r1-build20231113"
      size: "120 MB"
    
    - image: "rancher/hardened-coredns:v1.10.1-build20230406"
      size: "45 MB"
    
    - image: "rancher/hardened-etcd:v3.5.9-k3s1-build20230802"
      size: "80 MB"
    
    - image: "rancher/hardened-k8s-metrics-server:v0.6.3-build20230802"
      size: "60 MB"
  
  cni_networking:
    - image: "rancher/hardened-calico:v3.26.3-build20231009"
      size: "220 MB"
    
    - image: "rancher/hardened-flannel:v0.24.0-build20231011"
      size: "70 MB"
    
    - image: "rancher/hardened-cluster-autoscaler:v1.8.10-build20231009"
      size: "55 MB"
  
  ingress_controller:
    - image: "rancher/nginx-ingress-controller:v1.9.3-hardened1"
      size: "280 MB"
    
    - image: "rancher/hardened-klipper-lb:v0.4.4-build20231011"
      size: "15 MB"
  
  cloud_provider:
    - image: "rancher/mirrored-cloud-provider-vsphere-cpi-release-manager:v1.28.0"
      size: "120 MB"
    
    - image: "rancher/mirrored-cloud-provider-vsphere-csi-release-driver:v3.1.2"
      size: "150 MB"
  
  pause_container:
    - image: "rancher/mirrored-pause:3.9"
      size: "750 KB"
```

### 2. Rancher Server Images (v2.8.0)

```yaml
# File: airgap-artifacts/rancher-images.yaml

rancher_server:
  core:
    - image: "rancher/rancher:v2.8.0"
      size: "1.2 GB"
      pull_command: "docker pull rancher/rancher:v2.8.0"
    
    - image: "rancher/rancher-agent:v2.8.0"
      size: "600 MB"
    
    - image: "rancher/rancher-webhook:v0.4.1"
      size: "45 MB"
  
  shell_tools:
    - image: "rancher/shell:v0.1.23"
      size: "1.1 GB"
    
    - image: "rancher/kubectl:v1.28.3"
      size: "120 MB"
  
  fleet:  # GitOps
    - image: "rancher/fleet:v0.9.0"
      size: "80 MB"
    
    - image: "rancher/fleet-agent:v0.9.0"
      size: "75 MB"
    
    - image: "rancher/gitjob:v0.9.0"
      size: "70 MB"
  
  system_tools:
    - image: "rancher/system-upgrade-controller:v0.13.2"
      size: "35 MB"
    
    - image: "rancher/mirrored-cluster-api-controller:v1.5.3"
      size: "95 MB"
```

### 3. Cert-Manager Images (v1.13.2)

```yaml
# File: airgap-artifacts/cert-manager-images.yaml

cert_manager:
  - image: "quay.io/jetstack/cert-manager-controller:v1.13.2"
    size: "65 MB"
    pull_command: "docker pull quay.io/jetstack/cert-manager-controller:v1.13.2"
  
  - image: "quay.io/jetstack/cert-manager-webhook:v1.13.2"
    size: "50 MB"
  
  - image: "quay.io/jetstack/cert-manager-cainjector:v1.13.2"
    size: "55 MB"
  
  - image: "quay.io/jetstack/cert-manager-acmesolver:v1.13.2"
    size: "45 MB"
  
  - image: "quay.io/jetstack/cert-manager-ctl:v1.13.2"
    size: "40 MB"
```

### 4. Monitoring Stack Images

#### Prometheus Operator & Components

```yaml
# File: airgap-artifacts/monitoring-images.yaml

prometheus_stack:
  operator:
    - image: "quay.io/prometheus-operator/prometheus-operator:v0.70.0"
      size: "65 MB"
    
    - image: "quay.io/prometheus-operator/prometheus-config-reloader:v0.70.0"
      size: "25 MB"
  
  prometheus:
    - image: "quay.io/prometheus/prometheus:v2.48.0"
      size: "240 MB"
    
    - image: "quay.io/prometheus/alertmanager:v0.26.0"
      size: "70 MB"
    
    - image: "quay.io/prometheus/node-exporter:v1.7.0"
      size: "25 MB"
    
    - image: "quay.io/prometheus/blackbox-exporter:v0.24.0"
      size: "20 MB"
  
  kube_state_metrics:
    - image: "registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1"
      size: "35 MB"
  
  grafana:
    - image: "grafana/grafana:10.2.2"
      size: "380 MB"
    
    - image: "grafana/grafana-image-renderer:3.9.0"
      size: "450 MB"
  
  loki:
    - image: "grafana/loki:2.9.3"
      size: "85 MB"
    
    - image: "grafana/promtail:2.9.3"
      size: "180 MB"
  
  thanos:  # Optional: For long-term storage
    - image: "quay.io/thanos/thanos:v0.32.5"
      size: "120 MB"
```

### 5. Longhorn Distributed Storage (v1.5.3)

```yaml
# File: airgap-artifacts/longhorn-images.yaml

longhorn:
  core:
    - image: "longhornio/longhorn-manager:v1.5.3"
      size: "270 MB"
    
    - image: "longhornio/longhorn-engine:v1.5.3"
      size: "90 MB"
    
    - image: "longhornio/longhorn-instance-manager:v1.5.3"
      size: "85 MB"
    
    - image: "longhornio/longhorn-share-manager:v1.5.3"
      size: "80 MB"
    
    - image: "longhornio/backing-image-manager:v1.5.3"
      size: "75 MB"
  
  csi:
    - image: "longhornio/csi-attacher:v4.2.0"
      size: "55 MB"
    
    - image: "longhornio/csi-provisioner:v3.4.1"
      size: "60 MB"
    
    - image: "longhornio/csi-resizer:v1.7.0"
      size: "58 MB"
    
    - image: "longhornio/csi-snapshotter:v6.2.1"
      size: "62 MB"
    
    - image: "longhornio/csi-node-driver-registrar:v2.7.0"
      size: "20 MB"
    
    - image: "longhornio/livenessprobe:v2.9.0"
      size: "18 MB"
  
  ui:
    - image: "longhornio/longhorn-ui:v1.5.3"
      size: "150 MB"
```

### 6. Harbor Registry (v2.9.0)

```yaml
# File: airgap-artifacts/harbor-images.yaml

harbor:
  core_services:
    - image: "goharbor/harbor-core:v2.9.0"
      size: "180 MB"
    
    - image: "goharbor/harbor-portal:v2.9.0"
      size: "55 MB"
    
    - image: "goharbor/harbor-jobservice:v2.9.0"
      size: "160 MB"
    
    - image: "goharbor/harbor-registryctl:v2.9.0"
      size: "145 MB"
  
  registry:
    - image: "goharbor/registry-photon:v2.9.0"
      size: "85 MB"
  
  database:
    - image: "goharbor/harbor-db:v2.9.0"
      size: "240 MB"
  
  cache:
    - image: "goharbor/redis-photon:v2.9.0"
      size: "75 MB"
  
  proxy:
    - image: "goharbor/nginx-photon:v2.9.0"
      size: "45 MB"
  
  scanning:
    - image: "goharbor/trivy-adapter-photon:v2.9.0"
      size: "155 MB"
  
  notary:  # Image signing
    - image: "goharbor/notary-server-photon:v2.9.0"
      size: "115 MB"
    
    - image: "goharbor/notary-signer-photon:v2.9.0"
      size: "110 MB"
  
  chart_museum:
    - image: "goharbor/chartmuseum-photon:v2.9.0"
      size: "135 MB"
```

### 7. Backup and DR Images

```yaml
# File: airgap-artifacts/backup-images.yaml

velero:
  - image: "velero/velero:v1.12.1"
    size: "140 MB"
  
  - image: "velero/velero-plugin-for-aws:v1.8.1"
    size: "95 MB"
  
  - image: "velero/velero-plugin-for-csi:v0.6.1"
    size: "85 MB"

restic:
  - image: "velero/velero-restic-restore-helper:v1.12.1"
    size: "40 MB"

minio:  # S3-compatible storage for backups
  - image: "minio/minio:RELEASE.2023-11-15T20-43-25Z"
    size: "280 MB"
  
  - image: "minio/mc:RELEASE.2023-11-15T15-23-23Z"
    size: "90 MB"
```

### 8. Security and Policy Images

```yaml
# File: airgap-artifacts/security-images.yaml

falco:  # Runtime security
  - image: "falcosecurity/falco:0.36.2"
    size: "95 MB"
  
  - image: "falcosecurity/falco-driver-loader:0.36.2"
    size: "850 MB"
  
  - image: "falcosecurity/falcosidekick:2.28.0"
    size: "65 MB"

kyverno:  # Policy engine
  - image: "ghcr.io/kyverno/kyverno:v1.11.1"
    size: "120 MB"
  
  - image: "ghcr.io/kyverno/kyvernopre:v1.11.1"
    size: "85 MB"
  
  - image: "ghcr.io/kyverno/background-controller:v1.11.1"
    size: "80 MB"
  
  - image: "ghcr.io/kyverno/cleanup-controller:v1.11.1"
    size: "75 MB"
  
  - image: "ghcr.io/kyverno/reports-controller:v1.11.1"
    size: "78 MB"
```

### 9. CI/CD Images

```yaml
# File: airgap-artifacts/cicd-images.yaml

jenkins:
  - image: "jenkins/jenkins:2.426.1-lts-jdk17"
    size: "480 MB"
  
  - image: "jenkins/inbound-agent:latest-jdk17"
    size: "380 MB"

kaniko:  # Container builds
  - image: "gcr.io/kaniko-project/executor:v1.19.0"
    size: "85 MB"

buildah:  # Alternative builder
  - image: "quay.io/buildah/stable:v1.33"
    size: "390 MB"
```

## Download Scripts

### Complete Download Script

```bash
#!/bin/bash
# File: scripts/airgap/download-all-artifacts.sh

set -e

DOWNLOAD_DIR="/opt/airgap-artifacts"
REGISTRY_URL="harbor.internal.k8s"  # Your Harbor registry

mkdir -p ${DOWNLOAD_DIR}/{binaries,images,charts,scripts}

echo "=== Downloading RKE2 Artifacts ==="

# RKE2 version
RKE2_VERSION="v1.28.5+rke2r1"

# Download RKE2 binaries
wget -P ${DOWNLOAD_DIR}/binaries \
  https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-amd64.tar.gz \
  https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-amd64.tar.zst \
  https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/sha256sum-amd64.txt

# Download install script
wget -O ${DOWNLOAD_DIR}/scripts/rke2-install.sh https://get.rke2.io

# Verify checksums
cd ${DOWNLOAD_DIR}/binaries
sha256sum -c sha256sum-amd64.txt --ignore-missing
cd -

echo "=== Downloading Container Images ==="

# Function to pull, save, and compress images
pull_and_save_images() {
    local IMAGE_LIST=$1
    local OUTPUT_FILE=$2
    
    # Pull all images
    while IFS= read -r IMAGE; do
        [[ -z "$IMAGE" || "$IMAGE" =~ ^# ]] && continue
        echo "Pulling: $IMAGE"
        docker pull "$IMAGE" || echo "Failed to pull $IMAGE"
    done < "$IMAGE_LIST"
    
    # Save all images to tarball
    IMAGES=$(grep -v '^#' "$IMAGE_LIST" | grep -v '^$' | tr '\n' ' ')
    docker save $IMAGES | gzip > "${DOWNLOAD_DIR}/images/${OUTPUT_FILE}"
    
    echo "Saved images to ${OUTPUT_FILE}"
}

# Create image lists
cat > ${DOWNLOAD_DIR}/rancher-images.txt <<EOF
# Rancher Server
rancher/rancher:v2.8.0
rancher/rancher-agent:v2.8.0
rancher/rancher-webhook:v0.4.1
rancher/shell:v0.1.23
rancher/kubectl:v1.28.3
rancher/fleet:v0.9.0
rancher/fleet-agent:v0.9.0
rancher/gitjob:v0.9.0
rancher/system-upgrade-controller:v0.13.2
rancher/mirrored-cluster-api-controller:v1.5.3
EOF

cat > ${DOWNLOAD_DIR}/cert-manager-images.txt <<EOF
# Cert-Manager
quay.io/jetstack/cert-manager-controller:v1.13.2
quay.io/jetstack/cert-manager-webhook:v1.13.2
quay.io/jetstack/cert-manager-cainjector:v1.13.2
quay.io/jetstack/cert-manager-acmesolver:v1.13.2
EOF

cat > ${DOWNLOAD_DIR}/monitoring-images.txt <<EOF
# Prometheus Stack
quay.io/prometheus-operator/prometheus-operator:v0.70.0
quay.io/prometheus-operator/prometheus-config-reloader:v0.70.0
quay.io/prometheus/prometheus:v2.48.0
quay.io/prometheus/alertmanager:v0.26.0
quay.io/prometheus/node-exporter:v1.7.0
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1

# Grafana & Loki
grafana/grafana:10.2.2
grafana/loki:2.9.3
grafana/promtail:2.9.3
EOF

cat > ${DOWNLOAD_DIR}/longhorn-images.txt <<EOF
# Longhorn Storage
longhornio/longhorn-manager:v1.5.3
longhornio/longhorn-engine:v1.5.3
longhornio/longhorn-instance-manager:v1.5.3
longhornio/longhorn-share-manager:v1.5.3
longhornio/backing-image-manager:v1.5.3
longhornio/csi-attacher:v4.2.0
longhornio/csi-provisioner:v3.4.1
longhornio/csi-resizer:v1.7.0
longhornio/csi-snapshotter:v6.2.1
longhornio/csi-node-driver-registrar:v2.7.0
longhornio/livenessprobe:v2.9.0
longhornio/longhorn-ui:v1.5.3
EOF

cat > ${DOWNLOAD_DIR}/harbor-images.txt <<EOF
# Harbor Registry
goharbor/harbor-core:v2.9.0
goharbor/harbor-portal:v2.9.0
goharbor/harbor-jobservice:v2.9.0
goharbor/harbor-registryctl:v2.9.0
goharbor/registry-photon:v2.9.0
goharbor/harbor-db:v2.9.0
goharbor/redis-photon:v2.9.0
goharbor/nginx-photon:v2.9.0
goharbor/trivy-adapter-photon:v2.9.0
goharbor/chartmuseum-photon:v2.9.0
EOF

cat > ${DOWNLOAD_DIR}/backup-images.txt <<EOF
# Velero Backup
velero/velero:v1.12.1
velero/velero-plugin-for-aws:v1.8.1
velero/velero-plugin-for-csi:v0.6.1
velero/velero-restic-restore-helper:v1.12.1

# MinIO
minio/minio:RELEASE.2023-11-15T20-43-25Z
minio/mc:RELEASE.2023-11-15T15-23-23Z
EOF

# Download all image sets
pull_and_save_images "${DOWNLOAD_DIR}/rancher-images.txt" "rancher-images.tar.gz"
pull_and_save_images "${DOWNLOAD_DIR}/cert-manager-images.txt" "cert-manager-images.tar.gz"
pull_and_save_images "${DOWNLOAD_DIR}/monitoring-images.txt" "monitoring-images.tar.gz"
pull_and_save_images "${DOWNLOAD_DIR}/longhorn-images.txt" "longhorn-images.tar.gz"
pull_and_save_images "${DOWNLOAD_DIR}/harbor-images.txt" "harbor-images.tar.gz"
pull_and_save_images "${DOWNLOAD_DIR}/backup-images.txt" "backup-images.tar.gz"

echo "=== Downloading Helm Charts ==="

# Add Helm repositories
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add longhorn https://charts.longhorn.io
helm repo add harbor https://helm.goharbor.io
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# Download charts
helm pull rancher-stable/rancher --version 2.8.0 -d ${DOWNLOAD_DIR}/charts
helm pull jetstack/cert-manager --version v1.13.2 -d ${DOWNLOAD_DIR}/charts
helm pull prometheus-community/kube-prometheus-stack --version 54.2.2 -d ${DOWNLOAD_DIR}/charts
helm pull longhorn/longhorn --version 1.5.3 -d ${DOWNLOAD_DIR}/charts
helm pull harbor/harbor --version 1.13.1 -d ${DOWNLOAD_DIR}/charts
helm pull vmware-tanzu/velero --version 5.1.4 -d ${DOWNLOAD_DIR}/charts

echo "=== Download Summary ==="
echo "Binaries: $(du -sh ${DOWNLOAD_DIR}/binaries | cut -f1)"
echo "Images: $(du -sh ${DOWNLOAD_DIR}/images | cut -f1)"
echo "Charts: $(du -sh ${DOWNLOAD_DIR}/charts | cut -f1)"
echo "Total: $(du -sh ${DOWNLOAD_DIR} | cut -f1)"

echo ""
echo "✓ All artifacts downloaded successfully!"
echo "Transfer ${DOWNLOAD_DIR} to your airgapped environment"
```

### Image Size Calculation Script

```bash
#!/bin/bash
# File: scripts/airgap/calculate-image-sizes.sh

# Calculate sizes of all downloaded images

IMAGE_DIR="/opt/airgap-artifacts/images"

echo "=== Image Archive Sizes ==="
echo ""

total_size=0

for tarball in ${IMAGE_DIR}/*.tar.gz; do
    if [ -f "$tarball" ]; then
        size=$(stat -c%s "$tarball" 2>/dev/null || stat -f%z "$tarball")
        size_mb=$((size / 1024 / 1024))
        total_size=$((total_size + size))
        
        filename=$(basename "$tarball")
        printf "%-40s %8s MB\n" "$filename" "$size_mb"
    fi
done

echo ""
echo "----------------------------------------"
total_mb=$((total_size / 1024 / 1024))
total_gb=$((total_size / 1024 / 1024 / 1024))
printf "%-40s %8s MB (%s GB)\n" "TOTAL" "$total_mb" "$total_gb"
```

## Upload to Harbor Registry

```bash
#!/bin/bash
# File: scripts/airgap/upload-to-harbor.sh

set -e

HARBOR_URL="harbor.internal.k8s"
HARBOR_USER="admin"
HARBOR_PASSWORD="HarborPassword123!"
IMAGE_DIR="/opt/airgap-artifacts/images"

echo "Logging into Harbor..."
echo "$HARBOR_PASSWORD" | docker login $HARBOR_URL -u $HARBOR_USER --password-stdin

echo "Loading and pushing images to Harbor..."

for tarball in ${IMAGE_DIR}/*.tar.gz; do
    if [ -f "$tarball" ]; then
        echo "Processing: $(basename $tarball)"
        
        # Load images into Docker
        gunzip -c "$tarball" | docker load
    fi
done

# Get list of all loaded images
IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>")

# Re-tag and push to Harbor
for IMAGE in $IMAGES; do
    # Extract image name without registry
    IMAGE_NAME=$(echo $IMAGE | sed 's|^[^/]*/||' | sed 's|^[^/]*/||')
    
    # Determine project based on image
    if [[ $IMAGE == *"rancher"* ]]; then
        PROJECT="rancher"
    elif [[ $IMAGE == *"longhorn"* ]]; then
        PROJECT="longhorn"
    elif [[ $IMAGE == *"prometheus"* ]] || [[ $IMAGE == *"grafana"* ]]; then
        PROJECT="monitoring"
    elif [[ $IMAGE == *"cert-manager"* ]]; then
        PROJECT="cert-manager"
    elif [[ $IMAGE == *"harbor"* ]]; then
        PROJECT="harbor"
    elif [[ $IMAGE == *"velero"* ]] || [[ $IMAGE == *"minio"* ]]; then
        PROJECT="backup"
    else
        PROJECT="library"
    fi
    
    # Tag for Harbor
    NEW_TAG="${HARBOR_URL}/${PROJECT}/${IMAGE_NAME}"
    
    echo "Tagging: $IMAGE -> $NEW_TAG"
    docker tag "$IMAGE" "$NEW_TAG"
    
    echo "Pushing: $NEW_TAG"
    docker push "$NEW_TAG"
done

echo "✓ All images uploaded to Harbor"
```

## Verification Procedures

### Verify Downloaded Artifacts

```bash
#!/bin/bash
# File: scripts/airgap/verify-artifacts.sh

DOWNLOAD_DIR="/opt/airgap-artifacts"

echo "=== Verifying Downloaded Artifacts ==="

ERRORS=0

# Check RKE2 binaries
echo -n "Checking RKE2 tarball... "
if [ -f "${DOWNLOAD_DIR}/binaries/rke2.linux-amd64.tar.gz" ]; then
    echo "✓"
else
    echo "✗ MISSING"
    ((ERRORS++))
fi

echo -n "Checking RKE2 images... "
if [ -f "${DOWNLOAD_DIR}/binaries/rke2-images.linux-amd64.tar.zst" ]; then
    SIZE=$(stat -c%s "${DOWNLOAD_DIR}/binaries/rke2-images.linux-amd64.tar.zst" 2>/dev/null || stat -f%z "${DOWNLOAD_DIR}/binaries/rke2-images.linux-amd64.tar.zst")
    SIZE_GB=$((SIZE / 1024 / 1024 / 1024))
    echo "✓ (${SIZE_GB}GB)"
else
    echo "✗ MISSING"
    ((ERRORS++))
fi

# Verify checksums
echo -n "Verifying RKE2 checksums... "
cd ${DOWNLOAD_DIR}/binaries
if sha256sum -c sha256sum-amd64.txt --ignore-missing &>/dev/null; then
    echo "✓"
else
    echo "✗ FAILED"
    ((ERRORS++))
fi
cd - > /dev/null

# Check image tarballs
echo ""
echo "Checking image tarballs:"
for TARBALL in rancher-images cert-manager-images monitoring-images longhorn-images harbor-images backup-images; do
    echo -n "  ${TARBALL}.tar.gz... "
    if [ -f "${DOWNLOAD_DIR}/images/${TARBALL}.tar.gz" ]; then
        SIZE=$(stat -c%s "${DOWNLOAD_DIR}/images/${TARBALL}.tar.gz" 2>/dev/null || stat -f%z "${DOWNLOAD_DIR}/images/${TARBALL}.tar.gz")
        SIZE_MB=$((SIZE / 1024 / 1024))
        echo "✓ (${SIZE_MB}MB)"
    else
        echo "✗ MISSING"
        ((ERRORS++))
    fi
done

# Check Helm charts
echo ""
echo "Checking Helm charts:"
REQUIRED_CHARTS=("rancher" "cert-manager" "kube-prometheus-stack" "longhorn" "harbor" "velero")
for CHART in "${REQUIRED_CHARTS[@]}"; do
    echo -n "  ${CHART}... "
    if ls ${DOWNLOAD_DIR}/charts/${CHART}-*.tgz 1> /dev/null 2>&1; then
        echo "✓"
    else
        echo "✗ MISSING"
        ((ERRORS++))
    fi
done

# Summary
echo ""
echo "=== Verification Summary ==="
if [ $ERRORS -eq 0 ]; then
    echo "✓ All artifacts present and verified!"
    exit 0
else
    echo "✗ Found $ERRORS missing or invalid artifact(s)"
    exit 1
fi
```

### Verify Harbor Registry

```bash
#!/bin/bash
# File: scripts/airgap/verify-harbor-images.sh

HARBOR_URL="harbor.internal.k8s"
HARBOR_USER="admin"
HARBOR_PASSWORD="HarborPassword123!"

echo "=== Verifying Images in Harbor Registry ==="

# Login
echo "$HARBOR_PASSWORD" | docker login $HARBOR_URL -u $HARBOR_USER --password-stdin

# Expected image counts per project
declare -A EXPECTED_COUNTS=(
    [rancher]=10
    [cert-manager]=4
    [monitoring]=8
    [longhorn]=12
    [harbor]=10
    [backup]=6
)

ERRORS=0

for PROJECT in "${!EXPECTED_COUNTS[@]}"; do
    echo -n "Checking project: $PROJECT... "
    
    # Use Harbor API to get image count
    IMAGE_COUNT=$(curl -s -u ${HARBOR_USER}:${HARBOR_PASSWORD} \
        "https://${HARBOR_URL}/api/v2.0/projects/${PROJECT}/repositories" \
        | jq '. | length')
    
    EXPECTED=${EXPECTED_COUNTS[$PROJECT]}
    
    if [ "$IMAGE_COUNT" -ge "$EXPECTED" ]; then
        echo "✓ ($IMAGE_COUNT/$EXPECTED images)"
    else
        echo "✗ ($IMAGE_COUNT/$EXPECTED images)"
        ((ERRORS++))
    fi
done

if [ $ERRORS -eq 0 ]; then
    echo ""
    echo "✓ All images verified in Harbor"
    exit 0
else
    echo ""
    echo "✗ Some images missing from Harbor"
    exit 1
fi
```

## Storage Planning

### Disk Space Calculator

```bash
#!/bin/bash
# File: scripts/airgap/calculate-storage-needs.sh

# Calculate storage requirements for airgap installation

echo "=== Airgap Storage Requirements Calculator ==="
echo ""

# Base requirements (in GB)
RKE2_BINARIES=8
CONTAINER_IMAGES=35
RANCHER=5
CERT_MANAGER=0.5
MONITORING=10
LONGHORN=2
HARBOR=3
HELM_CHARTS=0.1
MISC=2

TOTAL_ARTIFACTS=$((RKE2_BINARIES + CONTAINER_IMAGES + RANCHER + CERT_MANAGER + MONITORING + LONGHORN + HARBOR + HELM_CHARTS + MISC))

# Harbor registry storage (stores all images)
HARBOR_STORAGE=$((CONTAINER_IMAGES + RANCHER + CERT_MANAGER + MONITORING + LONGHORN + 10))  # +10GB buffer

# Node-local storage for cached images
NODE_STORAGE=$((CONTAINER_IMAGES / 2))  # Estimate per node

echo "Component Storage Requirements:"
echo "  RKE2 binaries:           ${RKE2_BINARIES} GB"
echo "  Container images:        ${CONTAINER_IMAGES} GB"
echo "  Rancher images:          ${RANCHER} GB"
echo "  Cert-manager:            ${CERT_MANAGER} GB"
echo "  Monitoring stack:        ${MONITORING} GB"
echo "  Longhorn storage:        ${LONGHORN} GB"
echo "  Harbor registry:         ${HARBOR} GB (software)"
echo "  Helm charts:             ${HELM_CHARTS} GB"
echo "  Misc artifacts:          ${MISC} GB"
echo "  ------------------------"
echo "  Total download:          ${TOTAL_ARTIFACTS} GB"
echo ""
echo "Infrastructure Storage Requirements:"
echo "  Transfer media/disk:     $((TOTAL_ARTIFACTS + 20)) GB (with buffer)"
echo "  Harbor registry storage: ${HARBOR_STORAGE} GB (persistent volume)"
echo "  Per-node cache:          ~${NODE_STORAGE} GB (containerd)"
echo ""
echo "Recommended:"
echo "  - USB drive for transfer: 128 GB minimum"
echo "  - Harbor PV:             100 GB minimum"
echo "  - Per-node /var/lib:     200 GB minimum"
```

## Next Steps

After downloading and verifying all artifacts:

1. ✅ Transfer artifacts to airgapped environment
2. ➡️ Setup Harbor registry (see [Airgap Installation Guide](04-airgap-installation.md))
3. 📤 Upload images to Harbor
4. 🚀 Begin RKE2 installation with custom registry

## Quick Reference

```bash
# Download everything
./scripts/airgap/download-all-artifacts.sh

# Verify downloads
./scripts/airgap/verify-artifacts.sh

# Calculate sizes
./scripts/airgap/calculate-image-sizes.sh

# Transfer to airgapped environment (adjust path)
rsync -avz --progress /opt/airgap-artifacts/ airgap-server:/opt/airgap-artifacts/

# On airgapped side, upload to Harbor
./scripts/airgap/upload-to-harbor.sh

# Verify Harbor has all images
./scripts/airgap/verify-harbor-images.sh
```
