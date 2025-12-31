# Disaster Recovery (DR) Strategy and Implementation

## Table of Contents
1. [DR Overview](#dr-overview)
2. [DR Architecture](#dr-architecture)
3. [RTO and RPO Objectives](#rto-and-rpo-objectives)
4. [DR Site Setup](#dr-site-setup)
5. [Backup Strategies](#backup-strategies)
6. [Replication Configuration](#replication-configuration)
7. [Failover Procedures](#failover-procedures)
8. [Failback Procedures](#failback-procedures)
9. [DR Testing](#dr-testing)

## DR Overview

### Disaster Recovery Objectives

```yaml
dr_objectives:
  purpose: "Ensure business continuity in case of datacenter failure"
  
  targets:
    rto: "4 hours"  # Recovery Time Objective
    rpo: "1 hour"   # Recovery Point Objective
  
  scenarios_covered:
    - "Complete datacenter failure"
    - "Network partition"
    - "Hardware failures"
    - "Ransomware attacks"
    - "Natural disasters"
    - "Human errors"
  
  recovery_levels:
    - name: "Critical (Tier 1)"
      rto: "2 hours"
      rpo: "15 minutes"
      applications: ["Payment processing", "Core APIs"]
    
    - name: "Important (Tier 2)"
      rto: "4 hours"
      rpo: "1 hour"
      applications: ["Customer portal", "Reporting"]
    
    - name: "Normal (Tier 3)"
      rto: "8 hours"
      rpo: "4 hours"
      applications: ["Analytics", "Batch jobs"]
```

## DR Architecture

### Multi-Datacenter Design

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PRIMARY DATACENTER (DC1)                          │
│                         Region: US-East                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │           Production K8s Cluster (Active)                  │   │
│  │                                                            │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │   │ Master-1 │  │ Master-2 │  │ Master-3 │              │   │
│  │   └──────────┘  └──────────┘  └──────────┘              │   │
│  │                                                            │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │   │ Worker-1 │  │ Worker-2 │  │ Worker-N │              │   │
│  │   └──────────┘  └──────────┘  └──────────┘              │   │
│  │                                                            │   │
│  │   Rancher: rancher-dc1.k8s.internal                       │   │
│  │   API VIP: 192.168.1.100                                  │   │
│  └───────────────┬────────────────────────────────────────────   │
│                  │                                                 │
│  ┌───────────────▼────────────────────────────────────────────┐   │
│  │        Longhorn Storage (Replicated)                       │   │
│  │        etcd (Snapshots every 6 hours)                      │   │
│  └───────────────┬────────────────────────────────────────────┘   │
└──────────────────┼──────────────────────────────────────────────────┘
                   │
                   │  ┌─────────────────────────────────────┐
                   │  │  WAN Replication                    │
                   ├──│  - Longhorn cross-cluster sync      │
                   │  │  - etcd backup replication          │
                   │  │  - Velero backups to shared S3      │
                   │  │  - Git-based config sync            │
                   │  └─────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      DR DATACENTER (DC2)                             │
│                         Region: US-West                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │           DR K8s Cluster (Standby/Active)                  │   │
│  │                                                            │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │   │ Master-1 │  │ Master-2 │  │ Master-3 │              │   │
│  │   └──────────┘  └──────────┘  └──────────┘              │   │
│  │                                                            │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │   │ Worker-1 │  │ Worker-2 │  │ Worker-N │              │   │
│  │   └──────────┘  └──────────┘  └──────────┘              │   │
│  │                                                            │   │
│  │   Rancher: rancher-dc2.k8s.internal                       │   │
│  │   API VIP: 10.10.1.100                                    │   │
│  └───────────────┬────────────────────────────────────────────   │
│                  │                                                 │
│  ┌───────────────▼────────────────────────────────────────────┐   │
│  │        Longhorn Storage (Replica Target)                   │   │
│  │        etcd (Restored from backups)                        │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘

         ┌──────────────────────────────────────┐
         │   Global DNS / Traffic Management    │
         │                                      │
         │   app.example.com                    │
         │   ├─> DC1 (Primary) - Weight: 100   │
         │   └─> DC2 (DR) - Weight: 0          │
         │                                      │
         │   Failover: Update weights           │
         └──────────────────────────────────────┘
```

### DR Strategy Types

```yaml
dr_strategies:
  # Strategy 1: Hot Standby (Active-Active)
  hot_standby:
    description: "Both clusters running, accepting traffic"
    rto: "< 1 hour"
    rpo: "< 5 minutes"
    cost: "Highest"
    use_case: "Mission-critical applications"
    
    characteristics:
      - "Both clusters fully operational"
      - "Data continuously replicated"
      - "Traffic load-balanced or geo-routed"
      - "Immediate failover capability"
  
  # Strategy 2: Warm Standby (Active-Passive)
  warm_standby:
    description: "DR cluster running, not accepting traffic"
    rto: "2-4 hours"
    rpo: "15-60 minutes"
    cost: "Medium"
    use_case: "Production workloads (Recommended)"
    
    characteristics:
      - "DR cluster operational"
      - "Applications deployed but scaled down"
      - "Data replicated periodically"
      - "Quick activation possible"
  
  # Strategy 3: Cold Standby
  cold_standby:
    description: "DR infrastructure ready, cluster not running"
    rto: "8-24 hours"
    rpo: "4-12 hours"
    cost: "Lowest"
    use_case: "Non-critical systems"
    
    characteristics:
      - "Infrastructure provisioned"
      - "Cluster deployed on-demand"
      - "Backup-based recovery"
```

## RTO and RPO Objectives

### Calculating RTO

```yaml
rto_breakdown:
  detection_time: "15 minutes"  # Monitoring alerts
  decision_time: "30 minutes"   # Incident response team decision
  activation_time: "90 minutes" # Failover execution
  validation_time: "45 minutes" # Testing and verification
  
  total_rto: "3 hours (180 minutes)"
  target_rto: "4 hours"  # Buffer included
```

### Calculating RPO

```yaml
rpo_breakdown:
  continuous_replication:
    longhorn_sync: "15 minutes lag"
    database_replication: "5 minutes lag"
  
  scheduled_backups:
    etcd_snapshot: "6 hours"
    application_backup: "1 hour (Velero)"
    volume_snapshot: "4 hours"
  
  effective_rpo: "1 hour"  # Based on Velero backup frequency
```

## DR Site Setup

### Infrastructure Requirements for DR Site

```yaml
dr_site_infrastructure:
  location:
    requirement: "> 50 km from primary datacenter"
    recommended: "Different geographical region"
    network_connectivity: "Dedicated WAN link or VPN"
  
  compute_resources:
    masters:
      count: 3
      specs: "Same as primary"
    
    workers:
      count: "Same as primary (or N-1 for cost optimization)"
      specs: "Same as primary"
  
  storage:
    capacity: "Equal to primary + 20% buffer"
    type: "Longhorn or compatible with primary"
  
  network:
    bandwidth: "100 Mbps minimum"
    latency: "< 100ms to primary"
```

### Deploy DR Cluster

```bash
#!/bin/bash
# File: dr/scripts/01-deploy-dr-cluster.sh

# Deploy Kubernetes cluster in DR site
# Follow same steps as primary installation

set -e

DR_CLUSTER_NAME="dr-k8s-cluster"
DR_API_VIP="10.10.1.100"
DR_RANCHER_HOSTNAME="rancher-dr.k8s.internal"

echo "Deploying DR cluster: ${DR_CLUSTER_NAME}"

# 1. Prepare nodes (same as primary)
# Run prepare-node.sh on all DR nodes

# 2. Install first master
# Follow installation guide for first master

# 3. Join additional masters and workers

# 4. Label DR cluster
kubectl label nodes --all environment=dr
kubectl label nodes --all datacenter=dc2

echo "DR cluster deployed successfully"
```

## Backup Strategies

### etcd Backup Configuration

```yaml
# File: dr/configs/etcd-backup-config.yaml

# etcd backup is critical for cluster state recovery
apiVersion: v1
kind: ConfigMap
metadata:
  name: etcd-backup-config
  namespace: kube-system
data:
  # RKE2 automatically creates snapshots
  # Configuration in /etc/rancher/rke2/config.yaml
  schedule: "0 */6 * * *"  # Every 6 hours
  retention: "10"  # Keep last 10 snapshots
  location: "/var/lib/rancher/rke2/server/db/snapshots"
```

```bash
#!/bin/bash
# File: dr/scripts/02-backup-etcd.sh

# Manual etcd backup script

set -e

BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/etcd-snapshot-${DATE}.db"

mkdir -p ${BACKUP_DIR}

# Take etcd snapshot
ETCDCTL_API=3 etcdctl snapshot save ${SNAPSHOT_FILE} \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status ${SNAPSHOT_FILE} --write-out=table

# Sync to DR site
rsync -avz ${SNAPSHOT_FILE} dr-storage-server:/backup/etcd/

echo "etcd backup completed: ${SNAPSHOT_FILE}"

# Cleanup old backups (keep last 30 days)
find ${BACKUP_DIR} -name "etcd-snapshot-*.db" -mtime +30 -delete
```

### Velero for Application Backup

```bash
#!/bin/bash
# File: dr/scripts/03-install-velero.sh

# Install Velero for application and volume backups

set -e

VELERO_VERSION="1.12.0"
S3_BUCKET="k8s-velero-backups"
S3_REGION="us-east-1"
S3_ENDPOINT="https://s3.example.com"  # Use MinIO or AWS S3

echo "Installing Velero ${VELERO_VERSION}..."

# Download Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v${VELERO_VERSION}/velero-v${VELERO_VERSION}-linux-amd64.tar.gz
tar -xzf velero-v${VELERO_VERSION}-linux-amd64.tar.gz
mv velero-v${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
chmod +x /usr/local/bin/velero

# Create credentials file
cat > /tmp/velero-credentials <<EOF
[default]
aws_access_key_id=<ACCESS_KEY>
aws_secret_access_key=<SECRET_KEY>
EOF

# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket ${S3_BUCKET} \
  --secret-file /tmp/velero-credentials \
  --backup-location-config region=${S3_REGION},s3ForcePathStyle="true",s3Url=${S3_ENDPOINT} \
  --snapshot-location-config region=${S3_REGION} \
  --use-volume-snapshots=false

# Clean up credentials
rm /tmp/velero-credentials

echo "Velero installed successfully"

# Create backup schedule
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h \
  --include-namespaces production,staging

echo "Velero backup schedule created"
```

### Longhorn Cross-Cluster Replication

```bash
#!/bin/bash
# File: dr/scripts/04-configure-longhorn-replication.sh

# Configure Longhorn disaster recovery volumes

set -e

DR_CLUSTER_URL="https://10.10.1.100:6443"
DR_CLUSTER_TOKEN="<dr-cluster-token>"

echo "Configuring Longhorn cross-cluster replication..."

# This is done through Longhorn UI or API
# Steps:
# 1. In primary cluster Longhorn UI, create DR volume
# 2. Configure backup target (S3/NFS)
# 3. Set recurring backup schedule
# 4. In DR cluster, restore from backup target

# Example: Create backup target
cat <<EOF | kubectl apply -f -
apiVersion: longhorn.io/v1beta1
kind: BackupTarget
metadata:
  name: default
  namespace: longhorn-system
spec:
  backupTargetURL: s3://longhorn-backups@us-east-1/
  credentialSecret: longhorn-backup-secret
EOF

# Create backup schedule for critical volumes
cat <<EOF | kubectl apply -f -
apiVersion: longhorn.io/v1beta1
kind: RecurringJob
metadata:
  name: backup-critical-volumes
  namespace: longhorn-system
spec:
  cron: "0 */4 * * *"  # Every 4 hours
  task: "backup"
  groups:
    - critical
  retain: 10
  concurrency: 2
EOF

echo "Longhorn replication configured"
```

## Replication Configuration

### Database Replication

```yaml
# File: dr/configs/database-replication.yaml

# Example: PostgreSQL replication to DR site
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-replication-config
  namespace: production
data:
  # Primary database configuration
  primary_host: "postgres-primary.production.svc.cluster.local"
  replica_host: "postgres-replica-dr.production.svc.cluster.local"
  
  replication_mode: "async"  # or "sync" for zero data loss
  replication_lag_threshold: "30s"  # Alert if lag > 30 seconds
```

### Application Data Sync

```bash
#!/bin/bash
# File: dr/scripts/05-sync-application-data.sh

# Sync application data between primary and DR

set -e

PRIMARY_DATA_PATH="/data/applications"
DR_DATA_PATH="dr-server:/data/applications"

# Continuous rsync with compression
rsync -avz --delete \
  --exclude='*.tmp' \
  --exclude='cache/*' \
  ${PRIMARY_DATA_PATH}/ \
  ${DR_DATA_PATH}/

echo "Application data synced to DR site"
```

## Failover Procedures

### Automated Failover Detection

```bash
#!/bin/bash
# File: dr/scripts/06-monitor-and-failover.sh

# Monitor primary cluster health and trigger failover

set -e

PRIMARY_API="https://192.168.1.100:6443"
DR_API="https://10.10.1.100:6443"
FAILURE_THRESHOLD=3
FAILURE_COUNT=0

echo "Starting DR monitor..."

while true; do
  # Check primary cluster health
  if kubectl --server=${PRIMARY_API} get nodes &> /dev/null; then
    echo "$(date): Primary cluster healthy"
    FAILURE_COUNT=0
  else
    FAILURE_COUNT=$((FAILURE_COUNT + 1))
    echo "$(date): Primary cluster unreachable (${FAILURE_COUNT}/${FAILURE_THRESHOLD})"
    
    if [ ${FAILURE_COUNT} -ge ${FAILURE_THRESHOLD} ]; then
      echo "$(date): FAILURE THRESHOLD REACHED - INITIATING FAILOVER"
      
      # Trigger failover
      /dr/scripts/07-execute-failover.sh
      
      # Send alerts
      echo "DR FAILOVER INITIATED" | mail -s "CRITICAL: DR Failover" ops-team@example.com
      
      break
    fi
  fi
  
  sleep 60  # Check every minute
done
```

### Manual Failover Execution

```bash
#!/bin/bash
# File: dr/scripts/07-execute-failover.sh

# Execute DR failover to secondary datacenter

set -e

echo "=== EXECUTING DR FAILOVER ==="
echo "This will:"
echo "  1. Activate DR cluster applications"
echo "  2. Update DNS to point to DR cluster"
echo "  3. Scale up DR workloads"
echo ""
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Failover cancelled"
  exit 1
fi

# Log failover event
echo "$(date): DR failover initiated by $(whoami)" >> /var/log/dr-events.log

# 1. Switch kubectl context to DR cluster
kubectl config use-context dr-cluster

# 2. Scale up applications in DR cluster
echo "Scaling up DR applications..."

kubectl scale deployment --all --replicas=3 -n production
kubectl scale deployment --all --replicas=2 -n staging

# 3. Update DNS (example using AWS Route 53)
echo "Updating DNS records..."

# This example assumes you have AWS CLI configured
# Update your DNS provider accordingly

# Point app.example.com to DR load balancer
# aws route53 change-resource-record-sets \
#   --hosted-zone-id Z1234567890ABC \
#   --change-batch file:///dr/configs/dns-failover.json

# 4. Restore latest backups if needed
echo "Restoring latest Velero backups..."

LATEST_BACKUP=$(velero backup get --output=name | head -n1)
velero restore create --from-backup ${LATEST_BACKUP}

# 5. Verify applications
echo "Verifying applications..."

kubectl get pods -A
kubectl get svc -A

# 6. Health checks
echo "Running health checks..."

# Add your application-specific health checks here
# curl -f https://app-dr.example.com/health || echo "WARNING: Health check failed"

echo ""
echo "=== DR FAILOVER COMPLETE ==="
echo "Primary cluster is now: DR DATACENTER"
echo "Monitor applications and verify functionality"
echo ""
echo "To failback to primary:"
echo "  1. Repair primary datacenter issues"
echo "  2. Sync data from DR to primary"
echo "  3. Execute /dr/scripts/08-execute-failback.sh"
```

## Failback Procedures

```bash
#!/bin/bash
# File: dr/scripts/08-execute-failback.sh

# Failback to primary datacenter after recovery

set -e

echo "=== EXECUTING FAILBACK TO PRIMARY ==="

# 1. Verify primary cluster is healthy
kubectl --context=primary-cluster get nodes

if [ $? -ne 0 ]; then
  echo "ERROR: Primary cluster is not healthy!"
  exit 1
fi

# 2. Sync data from DR to primary
echo "Syncing data from DR to primary..."

# Restore latest DR backup to primary
LATEST_DR_BACKUP=$(velero backup get --context=dr-cluster --output=name | head -n1)
velero restore create --from-backup ${LATEST_DR_BACKUP} --context=primary-cluster

# 3. Update DNS to point back to primary
echo "Updating DNS to primary..."

# Update DNS records
# aws route53 change-resource-record-sets ...

# 4. Scale down DR cluster
echo "Scaling down DR cluster..."
kubectl --context=dr-cluster scale deployment --all --replicas=1 -n production

# 5. Monitor and verify
echo "Failback complete. Monitor primary cluster"

echo "=== FAILBACK COMPLETE ==="
```

## DR Testing

### DR Drill Schedule

```yaml
# File: dr/configs/dr-test-schedule.yaml

dr_testing:
  schedule:
    # Quarterly full DR drills
    - frequency: "Quarterly"
      type: "Full Failover Test"
      duration: "4 hours"
      participants: ["Ops Team", "Dev Team", "Management"]
      scope: "Complete datacenter failover"
    
    # Monthly partial tests
    - frequency: "Monthly"
      type: "Backup Restore Test"
      duration: "2 hours"
      participants: ["Ops Team"]
      scope: "Restore specific applications from backup"
    
    # Weekly validation
    - frequency: "Weekly"
      type: "Backup Verification"
      duration: "30 minutes"
      participants: ["On-call Engineer"]
      scope: "Verify backup completion and integrity"
```

### DR Test Runbook

```bash
#!/bin/bash
# File: dr/runbooks/dr-test-runbook.sh

# DR Test Runbook - Execute in test environment

set -e

echo "=== DR TEST RUNBOOK ==="

# Pre-test checklist
echo "Pre-Test Checklist:"
echo "[ ] All stakeholders notified"
echo "[ ] Test window scheduled"
echo "[ ] Primary cluster healthy"
echo "[ ] DR cluster ready"
echo "[ ] Backup jobs completed"
echo ""
read -p "All checks passed? (yes/no): " READY

if [ "$READY" != "yes" ]; then
  exit 1
fi

# Test execution
echo "Starting DR test at $(date)"

# 1. Document current state
kubectl get all -A > /tmp/dr-test-before.txt

# 2. Simulate primary failure
echo "Simulating primary datacenter failure..."
# In production: DO NOT actually shut down primary!
# For test: Use network policies or firewall rules

# 3. Execute failover
/dr/scripts/07-execute-failover.sh

# 4. Validate DR environment
echo "Validating DR environment..."
sleep 60

# Run smoke tests
/dr/scripts/run-smoke-tests.sh

# 5. Document results
kubectl get all -A > /tmp/dr-test-after.txt
diff /tmp/dr-test-before.txt /tmp/dr-test-after.txt > /tmp/dr-test-diff.txt

# 6. Failback to primary
/dr/scripts/08-execute-failback.sh

echo "=== DR TEST COMPLETE ==="
echo "Review results and document lessons learned"
```

## Next Steps

1. **Setup Monitoring**: [Monitoring & Observability](06-monitoring-observability.md)
2. **Configure Backups**: [Backup & Restore](07-backup-restore.md)
3. **Regular Testing**: Schedule quarterly DR drills
4. **Documentation**: Keep runbooks updated
5. **Training**: Ensure team is familiar with DR procedures
