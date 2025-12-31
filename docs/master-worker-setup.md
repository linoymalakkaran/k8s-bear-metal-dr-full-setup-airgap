# Master and Worker Node Setup with etcd

## Table of Contents
1. [Overview](#overview)
2. [Master Node Architecture](#master-node-architecture)
3. [First Master Node Bootstrap](#first-master-node-bootstrap)
4. [Additional Master Nodes](#additional-master-nodes)
5. [etcd Cluster Configuration](#etcd-cluster-configuration)
6. [Worker Node Setup](#worker-node-setup)
7. [Node Labeling and Taints](#node-labeling-and-taints)
8. [Troubleshooting](#troubleshooting)

## Overview

This guide provides detailed steps for:
- **Bootstrapping the first master node** (initializes the cluster and etcd)
- **Joining additional master nodes** (forms HA control plane with etcd cluster)
- **Configuring etcd cluster** (3-node quorum for high availability)
- **Adding worker nodes** (compute resources for workloads)

### Cluster Topology

```
┌─────────────────────────────────────────────────────────┐
│                   Control Plane (HA)                     │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  master-01  │  │  master-02  │  │  master-03  │     │
│  │             │  │             │  │             │     │
│  │ kube-api    │  │ kube-api    │  │ kube-api    │     │
│  │ controller  │  │ controller  │  │ controller  │     │
│  │ scheduler   │  │ scheduler   │  │ scheduler   │     │
│  │ etcd-1      │  │ etcd-2      │  │ etcd-3      │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         │                 │                 │           │
└─────────┼─────────────────┼─────────────────┼───────────┘
          │                 │                 │
          └─────────────────┴─────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                  │
┌─────────▼─────────┐            ┌──────────▼──────────┐
│   worker-01       │            │    worker-02        │
│                   │   . . .    │                     │
│  kubelet          │            │   kubelet           │
│  kube-proxy       │            │   kube-proxy        │
│  container runtime│            │   container runtime │
└───────────────────┘            └─────────────────────┘
```

## Master Node Architecture

### Components on Each Master Node

```yaml
control_plane_components:
  api_server:
    binary: kube-apiserver
    port: 6443
    purpose: "API endpoint for cluster operations"
    
  controller_manager:
    binary: kube-controller-manager
    purpose: "Runs controller loops (deployments, services, etc.)"
    
  scheduler:
    binary: kube-scheduler
    purpose: "Schedules pods to worker nodes"
    
  etcd:
    binary: etcd
    client_port: 2379
    peer_port: 2380
    data_dir: /var/lib/etcd
    purpose: "Distributed key-value store for cluster state"
```

### etcd Cluster Requirements

- **Minimum**: 3 nodes (tolerates 1 failure)
- **Recommended**: 3 nodes for production
- **Maximum recommended**: 5 nodes (tolerates 2 failures)
- **Quorum**: Majority needed (2 of 3, or 3 of 5)

> **Note**: etcd clusters must have an **odd number** of nodes for quorum

## First Master Node Bootstrap

### Step 1: Prepare Configuration

```bash
#!/bin/bash
# File: scripts/setup/bootstrap-first-master.sh

# Bootstrap the first master node (master-01)

set -e

NODE_NAME="master-01"
NODE_IP="192.168.10.10"
API_VIP="192.168.1.100"  # HAProxy VIP
CLUSTER_NAME="production-k8s"
SERVICE_CIDR="10.43.0.0/16"
POD_CIDR="10.42.0.0/16"

echo "=== Bootstrapping First Master Node: ${NODE_NAME} ==="

# Create RKE2 configuration directory
mkdir -p /etc/rancher/rke2

# Create RKE2 server configuration
cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Server Configuration - First Master Node
# This node will initialize the cluster and etcd

# Node identification
node-name: ${NODE_NAME}
node-ip: ${NODE_IP}

# Cluster endpoint (HAProxy VIP)
# Masters will register with this endpoint
server: https://${API_VIP}:9345
cluster-reset: false

# TLS SANs (additional names for API server certificate)
tls-san:
  - ${API_VIP}
  - api.k8s.internal
  - rancher.k8s.internal
  - ${NODE_IP}
  - ${NODE_NAME}
  - ${NODE_NAME}.k8s.internal

# Network configuration
cluster-cidr: ${POD_CIDR}
service-cidr: ${SERVICE_CIDR}
cluster-dns: 10.43.0.10
cluster-domain: cluster.local

# CNI selection
cni:
  - canal  # Flannel for networking + Calico for network policies

# Disable built-in components (we'll deploy manually)
disable:
  - rke2-ingress-nginx  # Using custom ingress later
  
# etcd configuration
etcd-expose-metrics: true
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 10
etcd-snapshot-dir: /var/lib/rancher/rke2/server/db/snapshots

# Kubelet configuration
kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=500m,memory=1Gi,ephemeral-storage=2Gi"
  - "system-reserved=cpu=500m,memory=1Gi,ephemeral-storage=2Gi"
  - "eviction-hard=memory.available<500Mi,nodefs.available<10%"

# Kube-apiserver arguments
kube-apiserver-arg:
  - "audit-log-path=/var/lib/rancher/rke2/server/logs/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "event-ttl=1h"

# Controller manager arguments
kube-controller-manager-arg:
  - "node-monitor-period=5s"
  - "node-monitor-grace-period=40s"
  - "pod-eviction-timeout=5m"

# Enable protection
protect-kernel-defaults: true

# Node labels and taints
node-label:
  - "node-role.kubernetes.io/master=true"
  - "node-type=control-plane"
node-taint:
  - "node-role.kubernetes.io/master:NoSchedule"

# Write kubeconfig with proper permissions
write-kubeconfig-mode: "0644"
EOF

echo "✓ RKE2 configuration created"
```

### Step 2: Install and Start RKE2 Server

```bash
# Continue in bootstrap-first-master.sh

# Download and install RKE2 (online installation)
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.28.5+rke2r1 sh -

# Enable and start RKE2 server
systemctl enable rke2-server.service
systemctl start rke2-server.service

echo "Waiting for RKE2 server to start..."
sleep 30

# Check status
systemctl status rke2-server.service --no-pager

# Wait for node to be ready
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
PATH=$PATH:/var/lib/rancher/rke2/bin

echo "Waiting for node to become Ready..."
while ! kubectl get nodes ${NODE_NAME} 2>/dev/null | grep -q "Ready"; do
    echo "  Node not ready yet, waiting..."
    sleep 10
done

echo "✓ First master node is Ready!"

# Display node status
kubectl get nodes -o wide

# Display cluster info
kubectl cluster-info

# Get server token (needed for joining other masters)
SERVER_TOKEN=$(cat /var/lib/rancher/rke2/server/node-token)

echo ""
echo "=== Bootstrap Complete ==="
echo "Server Token (save this for joining other nodes):"
echo "${SERVER_TOKEN}"
echo ""
echo "This token is also stored in: /var/lib/rancher/rke2/server/node-token"
```

### Step 3: Verify etcd

```bash
#!/bin/bash
# File: scripts/setup/verify-etcd.sh

# Verify etcd on first master

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

echo "=== Verifying etcd Cluster ==="

# Check etcd pod
echo "etcd pod status:"
kubectl -n kube-system get pods -l component=etcd

# Check etcd member list
echo ""
echo "etcd cluster members:"
kubectl -n kube-system exec -it etcd-master-01 -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    member list -w table

# Check etcd health
echo ""
echo "etcd cluster health:"
kubectl -n kube-system exec -it etcd-master-01 -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    endpoint health --cluster

# Check etcd status
echo ""
echo "etcd endpoint status:"
kubectl -n kube-system exec -it etcd-master-01 -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    endpoint status --cluster -w table
```

## Additional Master Nodes

### Step 1: Join Second Master (master-02)

```bash
#!/bin/bash
# File: scripts/setup/join-master.sh

# Join additional master nodes to the cluster

set -e

NODE_NAME="master-02"  # Change for each node
NODE_IP="192.168.10.11"  # Change for each node
API_VIP="192.168.1.100"
SERVER_TOKEN="<token-from-master-01>"  # From bootstrap output

echo "=== Joining Master Node: ${NODE_NAME} ==="

# Create RKE2 configuration
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Server Configuration - Additional Master Node
# This node will join existing cluster and etcd

# Server to join (HAProxy VIP)
server: https://${API_VIP}:9345
token: ${SERVER_TOKEN}

# Node identification
node-name: ${NODE_NAME}
node-ip: ${NODE_IP}

# TLS SANs
tls-san:
  - ${API_VIP}
  - api.k8s.internal
  - ${NODE_IP}
  - ${NODE_NAME}
  - ${NODE_NAME}.k8s.internal

# Network (must match cluster settings)
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
cluster-dns: 10.43.0.10

# CNI (must match cluster)
cni:
  - canal

# Disable same components
disable:
  - rke2-ingress-nginx

# etcd configuration (same as first master)
etcd-expose-metrics: true
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 10

# Kubelet configuration
kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=500m,memory=1Gi"
  - "system-reserved=cpu=500m,memory=1Gi"

# Protection
protect-kernel-defaults: true

# Node labels and taints
node-label:
  - "node-role.kubernetes.io/master=true"
  - "node-type=control-plane"
node-taint:
  - "node-role.kubernetes.io/master:NoSchedule"

write-kubeconfig-mode: "0644"
EOF

# Install RKE2
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.28.5+rke2r1 sh -

# Enable and start
systemctl enable rke2-server.service
systemctl start rke2-server.service

echo "Waiting for RKE2 server to start..."
sleep 30

# Verify
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

echo "Waiting for node to join..."
while ! kubectl get nodes ${NODE_NAME} 2>/dev/null | grep -q "Ready"; do
    echo "  Node not ready yet, waiting..."
    sleep 10
done

echo "✓ Master node ${NODE_NAME} joined successfully!"
kubectl get nodes -o wide
```

### Step 2: Join Third Master (master-03)

```bash
# Run the same script with different parameters:
NODE_NAME="master-03"
NODE_IP="192.168.10.12"

# Execute join-master.sh
./scripts/setup/join-master.sh
```

### Step 3: Verify 3-Node etcd Cluster

```bash
#!/bin/bash
# File: scripts/setup/verify-etcd-cluster.sh

# Verify all etcd members after joining all masters

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

echo "=== Verifying etcd Cluster Status ==="

# Get all etcd pods
echo "etcd pods:"
kubectl -n kube-system get pods -l component=etcd -o wide

# Member list from first master
POD_NAME=$(kubectl -n kube-system get pods -l component=etcd --field-selector spec.nodeName=master-01 -o jsonpath='{.items[0].metadata.name}')

echo ""
echo "etcd cluster members:"
kubectl -n kube-system exec ${POD_NAME} -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    member list -w table

# Health check
echo ""
echo "etcd cluster health:"
kubectl -n kube-system exec ${POD_NAME} -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    endpoint health --cluster -w table

# Expected output should show 3 members, all healthy
```

## etcd Cluster Configuration

### etcd Data Directory Structure

```bash
# etcd data is stored on dedicated volume
/var/lib/etcd/
├── member/
│   ├── snap/          # Database snapshots
│   └── wal/           # Write-ahead log
└── member.yaml        # Member configuration

# RKE2 etcd snapshots
/var/lib/rancher/rke2/server/db/snapshots/
├── etcd-snapshot-master-01-<timestamp>
├── etcd-snapshot-master-01-<timestamp>
└── ...
```

### Manual etcd Backup

```bash
#!/bin/bash
# File: scripts/backup/etcd-backup.sh

# Create manual etcd snapshot

set -e

SNAPSHOT_DIR="/var/lib/rancher/rke2/server/db/snapshots"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_NAME="etcd-snapshot-manual-${TIMESTAMP}"

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

# Get etcd pod on local node
POD_NAME=$(kubectl -n kube-system get pods -l component=etcd --field-selector spec.nodeName=$(hostname) -o jsonpath='{.items[0].metadata.name}')

echo "Creating etcd snapshot: ${SNAPSHOT_NAME}"

kubectl -n kube-system exec ${POD_NAME} -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    snapshot save ${SNAPSHOT_DIR}/${SNAPSHOT_NAME}

echo "✓ Snapshot created: ${SNAPSHOT_DIR}/${SNAPSHOT_NAME}"

# Verify snapshot
kubectl -n kube-system exec ${POD_NAME} -- etcdctl \
    snapshot status ${SNAPSHOT_DIR}/${SNAPSHOT_NAME} -w table
```

### etcd Restore Procedure

```bash
#!/bin/bash
# File: scripts/restore/etcd-restore.sh

# Restore etcd from snapshot
# WARNING: This will restore cluster state to the snapshot time

set -e

SNAPSHOT_FILE="/var/lib/rancher/rke2/server/db/snapshots/etcd-snapshot-manual-20240101-120000"

echo "WARNING: This will restore etcd to snapshot: ${SNAPSHOT_FILE}"
echo "All changes after snapshot creation will be LOST!"
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted."
    exit 1
fi

# Stop RKE2 on all master nodes first
for node in master-01 master-02 master-03; do
    echo "Stopping RKE2 on ${node}..."
    ssh ${node} "systemctl stop rke2-server"
done

# On first master, restore from snapshot
echo "Restoring from snapshot on master-01..."

# The restore process:
# 1. RKE2 automatically detects snapshots in the snapshot directory
# 2. Use rke2 server --cluster-reset to restore

# Create restore configuration
cat > /etc/rancher/rke2/config.yaml.restore <<EOF
cluster-reset: true
cluster-reset-restore-path: ${SNAPSHOT_FILE}
EOF

# Start RKE2 with restore flag
mv /etc/rancher/rke2/config.yaml /etc/rancher/rke2/config.yaml.bak
mv /etc/rancher/rke2/config.yaml.restore /etc/rancher/rke2/config.yaml

systemctl start rke2-server

echo "Waiting for restore to complete..."
sleep 60

# Restore original config
mv /etc/rancher/rke2/config.yaml.bak /etc/rancher/rke2/config.yaml
systemctl restart rke2-server

# Start other masters
for node in master-02 master-03; do
    echo "Starting RKE2 on ${node}..."
    ssh ${node} "systemctl start rke2-server"
done

echo "✓ etcd restore complete"
```

### etcd Performance Tuning

```yaml
# File: /etc/rancher/rke2/config.yaml
# etcd performance settings for production

etcd-arg:
  # Increase snapshot count threshold
  - "snapshot-count=10000"
  
  # Heartbeat and election timeouts (milliseconds)
  # Increase for high-latency networks
  - "heartbeat-interval=500"
  - "election-timeout=5000"
  
  # Database size limits
  - "quota-backend-bytes=8589934592"  # 8GB
  
  # Compaction settings
  - "auto-compaction-mode=periodic"
  - "auto-compaction-retention=1h"
  
  # Performance monitoring
  - "metrics=extensive"
```

## Worker Node Setup

### Step 1: Join First Worker

```bash
#!/bin/bash
# File: scripts/setup/join-worker.sh

# Join worker node to the cluster

set -e

NODE_NAME="worker-01"  # Change for each worker
NODE_IP="192.168.20.10"  # Change for each worker
API_VIP="192.168.1.100"
SERVER_TOKEN="<token-from-master-01>"

echo "=== Joining Worker Node: ${NODE_NAME} ==="

# Create RKE2 agent configuration
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
# RKE2 Agent Configuration - Worker Node

# Server to join (HAProxy VIP)
server: https://${API_VIP}:9345
token: ${SERVER_TOKEN}

# Node identification
node-name: ${NODE_NAME}
node-ip: ${NODE_IP}

# Kubelet configuration
kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=500m,memory=1Gi,ephemeral-storage=2Gi"
  - "system-reserved=cpu=500m,memory=1Gi,ephemeral-storage=2Gi"
  - "eviction-hard=memory.available<1Gi,nodefs.available<10%"
  
# Protection
protect-kernel-defaults: true

# Node labels
node-label:
  - "node-role.kubernetes.io/worker=true"
  - "node-type=worker"
  - "workload-type=general"  # Can customize per worker

# No taints on workers (allow scheduling)
EOF

# Install RKE2 agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION=v1.28.5+rke2r1 sh -

# Enable and start
systemctl enable rke2-agent.service
systemctl start rke2-agent.service

echo "Waiting for node to join..."
sleep 30

# Verify from master (need to run on master or with kubeconfig)
# kubectl get nodes ${NODE_NAME}

echo "✓ Worker node ${NODE_NAME} installation started"
echo "Verify from master node with: kubectl get nodes"
```

### Step 2: Join Additional Workers

```bash
# Join worker-02
NODE_NAME="worker-02"
NODE_IP="192.168.20.11"
./scripts/setup/join-worker.sh

# Join worker-03
NODE_NAME="worker-03"
NODE_IP="192.168.20.12"
./scripts/setup/join-worker.sh

# Continue for all workers...
```

### Step 3: Verify All Nodes

```bash
#!/bin/bash
# File: scripts/setup/verify-cluster.sh

# Verify complete cluster

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

echo "=== Cluster Node Status ==="
kubectl get nodes -o wide

echo ""
echo "=== Master Nodes ==="
kubectl get nodes -l node-role.kubernetes.io/master=true

echo ""
echo "=== Worker Nodes ==="
kubectl get nodes -l node-role.kubernetes.io/worker=true

echo ""
echo "=== Node Resource Capacity ==="
kubectl top nodes

echo ""
echo "=== System Pods ==="
kubectl -n kube-system get pods -o wide

echo ""
echo "=== Cluster Info ==="
kubectl cluster-info

echo ""
echo "=== Component Status ==="
kubectl get componentstatuses 2>/dev/null || echo "ComponentStatus API deprecated in v1.19+"
```

## Node Labeling and Taints

### Standard Labels

```bash
#!/bin/bash
# File: scripts/setup/label-nodes.sh

# Apply standard labels to nodes

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# Label masters
kubectl label nodes master-01 master-02 master-03 \
    node-role.kubernetes.io/control-plane=true \
    node-type=control-plane \
    --overwrite

# Label workers by type
kubectl label nodes worker-01 worker-02 \
    workload-type=general \
    node-type=worker \
    --overwrite

kubectl label nodes worker-03 worker-04 \
    workload-type=database \
    node-type=worker \
    disk-type=ssd \
    --overwrite

# Label nodes with special hardware
kubectl label nodes worker-05 \
    workload-type=gpu \
    gpu=nvidia-t4 \
    --overwrite

# Label nodes by zone/rack (for pod affinity)
kubectl label nodes master-01 worker-01 worker-03 \
    topology.kubernetes.io/zone=rack-1 \
    --overwrite

kubectl label nodes master-02 worker-02 worker-04 \
    topology.kubernetes.io/zone=rack-2 \
    --overwrite

kubectl label nodes master-03 worker-05 \
    topology.kubernetes.io/zone=rack-3 \
    --overwrite

echo "✓ Node labels applied"
kubectl get nodes --show-labels
```

### Custom Taints

```bash
#!/bin/bash
# File: scripts/setup/taint-nodes.sh

# Apply taints to control workload placement

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# Taint database workers (only pods with toleration can schedule)
kubectl taint nodes worker-03 worker-04 \
    workload=database:NoSchedule \
    --overwrite

# Taint GPU node (only GPU workloads)
kubectl taint nodes worker-05 \
    gpu=true:NoSchedule \
    --overwrite

# Remove taint from a node
# kubectl taint nodes worker-01 workload=database:NoSchedule-

echo "✓ Node taints applied"

# Show node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Example Pod with Node Selection

```yaml
# File: examples/pod-with-node-selection.yaml

apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  # Node selector - only schedule on labeled nodes
  nodeSelector:
    workload-type: database
    disk-type: ssd
  
  # Toleration - allow scheduling on tainted nodes
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  
  # Node affinity - preferred scheduling rules
  affinity:
    nodeAffinity:
      # MUST schedule on nodes with these labels
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - worker
      
      # PREFER to schedule on nodes with these labels
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - rack-1
  
  containers:
  - name: postgres
    image: postgres:15
    resources:
      requests:
        memory: "4Gi"
        cpu: "2000m"
      limits:
        memory: "8Gi"
        cpu: "4000m"
```

## Troubleshooting

### Node Not Joining

```bash
# On the node having issues:

# Check RKE2 service status
systemctl status rke2-server  # For master
systemctl status rke2-agent   # For worker

# Check logs
journalctl -u rke2-server -f  # For master
journalctl -u rke2-agent -f   # For worker

# Common issues:
# 1. Firewall blocking ports
systemctl stop firewalld
# Or open required ports (see network-design.md)

# 2. Time sync issues
timedatectl status
chronyc tracking

# 3. Token mismatch
# Get correct token from master-01:
cat /var/lib/rancher/rke2/server/node-token

# 4. Network connectivity
ping master-01
telnet 192.168.1.100 9345
```

### etcd Issues

```bash
# Check etcd pod logs
kubectl -n kube-system logs -l component=etcd --tail=100

# Check if etcd is in quorum
kubectl -n kube-system exec etcd-master-01 -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    endpoint health --cluster

# Remove failed etcd member
MEMBER_ID="<member-id-from-member-list>"
kubectl -n kube-system exec etcd-master-01 -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    member remove ${MEMBER_ID}
```

### Cluster Reset (Last Resort)

```bash
#!/bin/bash
# File: scripts/setup/reset-cluster.sh

# WARNING: This completely removes RKE2 and resets the node

echo "WARNING: This will completely remove RKE2 from this node!"
read -p "Continue? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then exit 1; fi

# Stop RKE2
systemctl stop rke2-server 2>/dev/null || true
systemctl stop rke2-agent 2>/dev/null || true

# Run uninstall script
/usr/local/bin/rke2-uninstall.sh 2>/dev/null || true
/usr/local/bin/rke2-agent-uninstall.sh 2>/dev/null || true

# Clean up directories
rm -rf /etc/rancher/rke2
rm -rf /var/lib/rancher/rke2
rm -rf /var/lib/kubelet
rm -rf /etc/cni/net.d

echo "✓ Node reset complete. Ready for fresh installation."
```

## Summary Checklist

```yaml
cluster_setup_checklist:
  master_nodes:
    - [ ] Bootstrap first master (master-01)
    - [ ] Verify etcd started on master-01
    - [ ] Save server token
    - [ ] Join master-02
    - [ ] Join master-03
    - [ ] Verify 3-node etcd cluster
    - [ ] Verify all masters are Ready
    - [ ] Configure HAProxy to point to all masters
  
  worker_nodes:
    - [ ] Join worker-01
    - [ ] Join worker-02
    - [ ] Join worker-03
    - [ ] Verify all workers are Ready
    - [ ] Apply node labels
    - [ ] Apply node taints (if needed)
  
  verification:
    - [ ] All nodes show "Ready" status
    - [ ] etcd cluster healthy (3 members)
    - [ ] System pods running
    - [ ] Can deploy test workload
    - [ ] etcd backups working
    - [ ] HA failover tested
```

## Next Steps

After cluster is established:
1. ➡️ Install Rancher Management Server
2. 📦 Deploy Longhorn storage
3. 📊 Setup monitoring stack
4. 🔐 Configure RBAC and security policies
5. 🚀 Deploy first applications
