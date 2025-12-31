# Installation Guide - Production Kubernetes Cluster with Rancher

## Table of Contents
1. [Pre-Installation Checklist](#pre-installation-checklist)
2. [Node Preparation](#node-preparation)
3. [Load Balancer Setup](#load-balancer-setup)
4. [RKE2 Cluster Installation](#rke2-cluster-installation)
5. [Rancher Installation](#rancher-installation)
6. [Post-Installation Configuration](#post-installation-configuration)
7. [Verification](#verification)

## Pre-Installation Checklist

```yaml
prerequisites_checklist:
  infrastructure:
    - [ ] All nodes provisioned and accessible
    - [ ] IP addresses assigned and documented
    - [ ] DNS records created
    - [ ] Network connectivity verified between all nodes
    - [ ] Load balancers configured
    
  software:
    - [ ] OS installed (RHEL 8+, Ubuntu 20.04+, Rocky Linux 8+)
    - [ ] OS updated to latest patches
    - [ ] Root or sudo access verified on all nodes
    - [ ] SSH keys configured for passwordless access
    
  network:
    - [ ] Firewall rules configured
    - [ ] Required ports opened
    - [ ] NTP synchronized across all nodes
    - [ ] DNS resolution working
    
  storage:
    - [ ] Sufficient disk space available
    - [ ] Storage backend chosen (Longhorn/Ceph/NFS)
    
  security:
    - [ ] SSL/TLS certificates obtained
    - [ ] Secrets management strategy defined
```

## Node Preparation

### Automated Node Preparation Script

```bash
#!/bin/bash
# File: scripts/setup/prepare-node.sh

# This script prepares a Linux node for Kubernetes/RKE2 installation
# Run on all master and worker nodes

set -e

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}=== Starting Node Preparation ===${NC}"

# Function to print status
print_status() {
    echo -e "${GREEN}✓${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}⚠${NC} $1"
}

print_error() {
    echo -e "${RED}✗${NC} $1"
}

# 1. Update system
echo "Updating system packages..."
if command -v yum &> /dev/null; then
    yum update -y
    print_status "System updated (RHEL/Rocky)"
elif command -v apt &> /dev/null; then
    apt update && apt upgrade -y
    print_status "System updated (Ubuntu/Debian)"
fi

# 2. Disable swap
echo "Disabling swap..."
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
print_status "Swap disabled"

# 3. Load required kernel modules
echo "Loading kernel modules..."
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

modprobe overlay
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

print_status "Kernel modules loaded"

# 4. Configure sysctl parameters
echo "Configuring sysctl parameters..."
cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
# Kubernetes sysctl settings
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.forwarding        = 1
net.ipv6.conf.all.forwarding        = 1

# Increase connection tracking
net.netfilter.nf_conntrack_max = 1000000
net.nf_conntrack_max = 1000000

# Performance tuning
vm.swappiness = 0
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1

# File descriptors
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
EOF

sysctl --system
print_status "Sysctl parameters configured"

# 5. Configure SELinux (set to permissive for K8s)
echo "Configuring SELinux..."
if command -v setenforce &> /dev/null; then
    setenforce 0
    sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    print_status "SELinux set to permissive"
else
    print_warning "SELinux not found (Ubuntu/Debian)"
fi

# 6. Configure firewall
echo "Configuring firewall..."
if command -v firewall-cmd &> /dev/null; then
    systemctl enable firewalld
    systemctl start firewalld
    
    # Basic ports - will be customized based on node role
    firewall-cmd --permanent --add-port=22/tcp  # SSH
    firewall-cmd --permanent --add-port=6443/tcp  # K8s API
    firewall-cmd --permanent --add-port=10250/tcp  # Kubelet
    firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
    firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort
    
    # CNI ports
    firewall-cmd --permanent --add-port=8472/udp  # Flannel VXLAN
    firewall-cmd --permanent --add-port=4789/udp  # Flannel VXLAN alt
    
    firewall-cmd --reload
    print_status "Firewall configured"
fi

# 7. Install required packages
echo "Installing required packages..."
if command -v yum &> /dev/null; then
    yum install -y \
        curl \
        wget \
        vim \
        git \
        iptables \
        socat \
        conntrack \
        ipvsadm \
        nfs-utils \
        chrony \
        jq \
        tar
elif command -v apt &> /dev/null; then
    apt install -y \
        curl \
        wget \
        vim \
        git \
        iptables \
        socat \
        conntrack \
        ipvsadm \
        nfs-common \
        chrony \
        jq \
        tar
fi
print_status "Required packages installed"

# 8. Configure NTP/Chrony
echo "Configuring time synchronization..."
systemctl enable chronyd
systemctl start chronyd

# Add NTP server if needed
# echo "server ntp.example.com iburst" >> /etc/chrony.conf
# systemctl restart chronyd

chronyc sources
print_status "Time synchronization configured"

# 9. Configure limits
echo "Configuring system limits..."
cat >> /etc/security/limits.conf <<EOF
# Kubernetes limits
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
EOF

print_status "System limits configured"

# 10. Create directories
echo "Creating required directories..."
mkdir -p /etc/rancher/rke2
mkdir -p /var/lib/rancher/rke2
mkdir -p /var/log/rancher

print_status "Directories created"

# 11. Disable unnecessary services
echo "Disabling unnecessary services..."
systemctl disable --now postfix 2>/dev/null || true
systemctl disable --now cups 2>/dev/null || true

# 12. Hostname configuration
echo "Current hostname: $(hostname)"
print_warning "Ensure hostname is set correctly and matches DNS records"

echo -e "${GREEN}=== Node Preparation Complete ===${NC}"
echo ""
echo "Next steps:"
echo "  1. Verify hostname: hostnamectl set-hostname <node-name>"
echo "  2. Update /etc/hosts with all cluster nodes"
echo "  3. Configure node-specific settings (master vs worker)"
echo "  4. Reboot the node to ensure all changes take effect"
echo ""
echo "Reboot now? (y/n)"
read -r response
if [[ "$response" =~ ^[Yy]$ ]]; then
    reboot
fi
```

### Configure /etc/hosts on All Nodes

```bash
#!/bin/bash
# File: scripts/setup/configure-hosts.sh

# Update /etc/hosts with all cluster nodes

cat >> /etc/hosts <<EOF
# Kubernetes Cluster Nodes

# Load Balancers
192.168.1.10    lb-01.k8s.internal lb-01
192.168.1.11    lb-02.k8s.internal lb-02
192.168.1.100   api.k8s.internal

# Master Nodes
192.168.10.10   master-01.k8s.internal master-01
192.168.10.11   master-02.k8s.internal master-02
192.168.10.12   master-03.k8s.internal master-03

# Worker Nodes
192.168.20.10   worker-01.k8s.internal worker-01
192.168.20.11   worker-02.k8s.internal worker-02
192.168.20.12   worker-03.k8s.internal worker-03

# Infrastructure
192.168.10.20   rancher.k8s.internal rancher
192.168.10.30   bastion.k8s.internal bastion
EOF

echo "/etc/hosts updated"
```

## Load Balancer Setup

The load balancer configuration is detailed in [Network Design](02-network-design.md). Ensure HAProxy and Keepalived are configured before proceeding.

## RKE2 Cluster Installation

### Install First Master Node

```bash
#!/bin/bash
# File: scripts/setup/install-rke2-first-master.sh

# Install RKE2 on the first master node

set -e

RKE2_VERSION="v1.28.5+rke2r1"
CLUSTER_VIP="192.168.1.100"  # Load balancer VIP
CLUSTER_NAME="production-k8s"

echo "Installing RKE2 first master node..."

# Create RKE2 configuration
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Configuration for First Master Node

# Write kubeconfig with proper permissions
write-kubeconfig-mode: "0640"

# TLS SANs for API server certificate
tls-san:
  - "api.k8s.internal"
  - "${CLUSTER_VIP}"
  - "master-01.k8s.internal"
  - "192.168.10.10"

# Cluster name
cluster-name: "${CLUSTER_NAME}"

# Network CIDRs
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"

# CNI Plugin (canal = calico + flannel)
cni: "canal"

# Disable default components if not needed
# disable:
#   - rke2-ingress-nginx  # Use custom ingress

# Cloud provider (none for on-prem)
cloud-provider-name: ""

# etcd configuration
etcd-snapshot-schedule-cron: "0 */6 * * *"  # Every 6 hours
etcd-snapshot-retention: 10

# Node labels
node-label:
  - "node-role.kubernetes.io/master=true"
  - "node-type=master"

# Node taints (optional - prevent workloads on masters)
node-taint:
  - "node-role.kubernetes.io/master=true:NoSchedule"

EOF

# Download and install RKE2
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=${RKE2_VERSION} sh -

# Enable and start RKE2 server
systemctl enable rke2-server.service
systemctl start rke2-server.service

echo "Waiting for RKE2 to start..."
sleep 30

# Check status
systemctl status rke2-server.service

# Set up kubectl access
mkdir -p ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config

# Add RKE2 binaries to PATH
cat >> ~/.bashrc <<'EOF'
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
EOF

source ~/.bashrc

# Verify cluster
echo "Verifying cluster..."
kubectl get nodes
kubectl get pods -A

# Get node token for additional nodes
echo ""
echo "=== Node Token for Additional Nodes ==="
cat /var/lib/rancher/rke2/server/node-token
echo ""
echo "Save this token for joining additional master and worker nodes"

echo "First master node installation complete!"
```

### Join Additional Master Nodes

```bash
#!/bin/bash
# File: scripts/setup/join-rke2-master.sh

# Join additional master nodes to the cluster

set -e

FIRST_MASTER_IP="192.168.10.10"
FIRST_MASTER_URL="https://${FIRST_MASTER_IP}:9345"
NODE_TOKEN="<paste-token-here>"  # Get from first master
CLUSTER_VIP="192.168.1.100"
CURRENT_NODE_IP=$(hostname -I | awk '{print $1}')

echo "Joining master node to cluster..."

# Create RKE2 configuration
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Configuration for Additional Master Node

server: ${FIRST_MASTER_URL}
token: ${NODE_TOKEN}

write-kubeconfig-mode: "0640"

tls-san:
  - "api.k8s.internal"
  - "${CLUSTER_VIP}"
  - "$(hostname)"
  - "${CURRENT_NODE_IP}"

# Node labels
node-label:
  - "node-role.kubernetes.io/master=true"
  - "node-type=master"

# Node taints
node-taint:
  - "node-role.kubernetes.io/master=true:NoSchedule"

EOF

# Install RKE2
curl -sfL https://get.rke2.io | sh -

# Enable and start
systemctl enable rke2-server.service
systemctl start rke2-server.service

echo "Waiting for node to join..."
sleep 30

# Configure kubectl
mkdir -p ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config

cat >> ~/.bashrc <<'EOF'
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
EOF

source ~/.bashrc

# Verify
kubectl get nodes

echo "Master node joined successfully!"
```

### Join Worker Nodes

```bash
#!/bin/bash
# File: scripts/setup/join-rke2-worker.sh

# Join worker nodes to the cluster

set -e

CLUSTER_VIP="192.168.1.100"  # Use load balancer VIP
CLUSTER_URL="https://${CLUSTER_VIP}:9345"
NODE_TOKEN="<paste-token-here>"  # Get from master

echo "Joining worker node to cluster..."

# Create RKE2 agent configuration
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Configuration for Worker Node

server: ${CLUSTER_URL}
token: ${NODE_TOKEN}

# Node labels
node-label:
  - "node-role.kubernetes.io/worker=true"
  - "node-type=worker"

EOF

# Install RKE2 agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

# Enable and start
systemctl enable rke2-agent.service
systemctl start rke2-agent.service

echo "Worker node joining cluster..."
sleep 20

echo "Worker node installation complete!"
echo "Verify from master node: kubectl get nodes"
```

## Rancher Installation

### Install cert-manager

```bash
#!/bin/bash
# File: scripts/setup/install-cert-manager.sh

# Install cert-manager for certificate management

set -e

CERT_MANAGER_VERSION="v1.13.1"

echo "Installing cert-manager ${CERT_MANAGER_VERSION}..."

# Add Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Create namespace
kubectl create namespace cert-manager

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version ${CERT_MANAGER_VERSION} \
  --set installCRDs=true \
  --set global.leaderElection.namespace=cert-manager

# Wait for cert-manager to be ready
echo "Waiting for cert-manager pods..."
kubectl wait --for=condition=Available --timeout=300s \
  --namespace cert-manager \
  deployment/cert-manager

kubectl wait --for=condition=Available --timeout=300s \
  --namespace cert-manager \
  deployment/cert-manager-webhook

kubectl wait --for=condition=Available --timeout=300s \
  --namespace cert-manager \
  deployment/cert-manager-cainjector

echo "cert-manager installed successfully!"
kubectl get pods -n cert-manager
```

### Install Rancher

```bash
#!/bin/bash
# File: scripts/setup/install-rancher.sh

# Install Rancher management platform

set -e

RANCHER_VERSION="2.8.0"
RANCHER_HOSTNAME="rancher.k8s.internal"
BOOTSTRAP_PASSWORD="YourSecureBootstrapPassword123!"

echo "Installing Rancher ${RANCHER_VERSION}..."

# Add Rancher Helm repository
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

# Create namespace
kubectl create namespace cattle-system

# Install Rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=${RANCHER_HOSTNAME} \
  --set bootstrapPassword=${BOOTSTRAP_PASSWORD} \
  --set replicas=3 \
  --set ingress.tls.source=rancher \
  --version ${RANCHER_VERSION}

echo "Waiting for Rancher deployment..."
kubectl -n cattle-system rollout status deploy/rancher

echo ""
echo "=== Rancher Installation Complete ==="
echo "Rancher URL: https://${RANCHER_HOSTNAME}"
echo "Bootstrap Password: ${BOOTSTRAP_PASSWORD}"
echo ""
echo "Wait a few minutes for Rancher to fully initialize..."
echo "Access the UI and complete setup"
```

## Post-Installation Configuration

### Install Longhorn Storage

```bash
#!/bin/bash
# File: scripts/setup/install-longhorn.sh

# Install Longhorn distributed storage

set -e

echo "Installing Longhorn storage..."

# Install dependencies on all nodes first
# (Run on each node)
# yum install -y iscsi-initiator-utils
# systemctl enable iscsid
# systemctl start iscsid

# Add Longhorn Helm repository
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Create namespace
kubectl create namespace longhorn-system

# Install Longhorn
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --set defaultSettings.defaultReplicaCount=3 \
  --set defaultSettings.defaultDataPath="/var/lib/longhorn"

echo "Waiting for Longhorn deployment..."
kubectl -n longhorn-system rollout status deployment/longhorn-driver-deployer

echo "Longhorn installed successfully!"
echo "Access Longhorn UI through Rancher or port-forward:"
echo "kubectl port-forward -n longhorn-system svc/longhorn-frontend 8000:80"
```

### Configure Storage Classes

```yaml
# File: kubernetes/storage/storageclass-longhorn.yaml

# Default storage class using Longhorn
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"  # 48 hours
  fromBackup: ""
  fsType: "ext4"
```

```bash
# Apply storage class
kubectl apply -f kubernetes/storage/storageclass-longhorn.yaml
```

## Verification

### Verification Script

```bash
#!/bin/bash
# File: scripts/setup/verify-installation.sh

# Comprehensive installation verification

set -e

echo "=== Kubernetes Cluster Verification ==="

# Check nodes
echo "1. Checking cluster nodes..."
kubectl get nodes -o wide

# Check system pods
echo ""
echo "2. Checking system pods..."
kubectl get pods -A

# Check Rancher
echo ""
echo "3. Checking Rancher..."
kubectl get pods -n cattle-system

# Check cert-manager
echo ""
echo "4. Checking cert-manager..."
kubectl get pods -n cert-manager

# Check storage
echo ""
echo "5. Checking storage class..."
kubectl get storageclass

# Check Longhorn
echo ""
echo "6. Checking Longhorn..."
kubectl get pods -n longhorn-system

# Test pod deployment
echo ""
echo "7. Testing pod deployment..."
kubectl run test-nginx --image=nginx:latest --port=80
sleep 10
kubectl get pod test-nginx
kubectl delete pod test-nginx

# Check API server connectivity
echo ""
echo "8. Testing API server connectivity..."
kubectl cluster-info

# Check component health
echo ""
echo "9. Checking component health..."
kubectl get componentstatuses 2>/dev/null || echo "ComponentStatus deprecated in newer versions"

# Check etcd
echo ""
echo "10. Checking etcd health..."
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health 2>/dev/null || echo "Run on master node"

echo ""
echo "=== Verification Complete ==="
echo ""
echo "Next steps:"
echo "  1. Access Rancher UI: https://rancher.k8s.internal"
echo "  2. Complete Rancher initial setup"
echo "  3. Import or create additional clusters"
echo "  4. Configure backup: See docs/07-backup-restore.md"
echo "  5. Setup monitoring: See docs/06-monitoring-observability.md"
echo "  6. Plan DR: See docs/05-disaster-recovery.md"
```

## Next Steps

1. **Configure Monitoring**: [Monitoring & Observability](06-monitoring-observability.md)
2. **Setup Backups**: [Backup & Restore](07-backup-restore.md)
3. **Plan Disaster Recovery**: [Disaster Recovery](05-disaster-recovery.md)
4. **Deploy Applications**: Use Helm charts from `helm-charts/` directory
5. **Setup CI/CD**: Configure pipelines from `pipelines/` directory

## Troubleshooting

### Common Issues

```bash
# RKE2 not starting
journalctl -u rke2-server -f

# Check RKE2 logs
tail -f /var/lib/rancher/rke2/agent/logs/kubelet.log

# Reset RKE2 (WARNING: Destructive)
/usr/local/bin/rke2-killall.sh
rm -rf /var/lib/rancher/rke2
rm -rf /etc/rancher/rke2

# Network issues
# Check CNI
kubectl get pods -n kube-system | grep canal

# Check DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

For additional help, consult the [Maintenance Operations](docs/08-maintenance-operations.md) guide.
