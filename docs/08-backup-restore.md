# Backup and Restore Procedures

## Table of Contents
1. [Backup Strategy](#backup-strategy)
2. [Backup Components](#backup-components)
3. [Velero Installation](#velero-installation)
4. [etcd Backup](#etcd-backup)
5. [Persistent Volume Backup](#persistent-volume-backup)
6. [Application Backup](#application-backup)
7. [Restore Procedures](#restore-procedures)
8. [Automated Backup Schedule](#automated-backup-schedule)
9. [Backup Verification](#backup-verification)

## Backup Strategy

### Backup Requirements

```yaml
backup_strategy:
  objectives:
    rpo: "1 hour"  # Recovery Point Objective
    rto: "4 hours" # Recovery Time Objective
    retention:
      daily: "30 days"
      weekly: "12 weeks"
      monthly: "12 months"
      yearly: "7 years"
  
  what_to_backup:
    cluster_state:
      - "etcd database (cluster state)"
      - "Kubernetes manifests"
      - "ConfigMaps and Secrets"
      - "RBAC policies"
    
    application_data:
      - "Persistent Volumes"
      - "Application databases"
      - "User data"
    
    configuration:
      - "Helm charts and values"
      - "Infrastructure as Code"
      - "Network policies"
      - "Monitoring configurations"
```

### Backup Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                         │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Velero                                                 │ │
│  │  (Backup Orchestrator)                                  │ │
│  │                                                         │ │
│  │  - Schedule backups                                     │ │
│  │  - Manage retention                                     │ │
│  │  - Coordinate restores                                  │ │
│  └────┬─────────────────────────────────┬─────────────────┘ │
│       │                                  │                   │
│       │                                  │                   │
│  ┌────▼──────────────┐            ┌─────▼───────────────┐  │
│  │  etcd Snapshots   │            │  Restic Volumes     │  │
│  │  (Cluster State)  │            │  (PV Snapshots)     │  │
│  └────┬──────────────┘            └─────┬───────────────┘  │
│       │                                  │                   │
└───────┼──────────────────────────────────┼───────────────────┘
        │                                  │
        │                                  │
        ▼                                  ▼
┌──────────────────────────────────────────────────────────────┐
│                  Backup Storage (S3-Compatible)              │
│                                                              │
│  ┌────────────────────┐  ┌────────────────────────────────┐ │
│  │  MinIO / S3        │  │  NFS / Object Storage           │ │
│  │                    │  │                                  │ │
│  │  - Versioning      │  │  - Geo-replicated                │ │
│  │  - Encryption      │  │  - Immutable storage             │ │
│  │  - Lifecycle       │  │  - Retention policies            │ │
│  └────────────────────┘  └────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## Backup Components

### 1. etcd Backup (Cluster State)

The etcd database stores ALL Kubernetes cluster state:
- All Kubernetes objects (Deployments, Services, Pods, etc.)
- ConfigMaps and Secrets
- RBAC policies
- Custom Resource Definitions (CRDs)

### 2. Persistent Volume Backup (Application Data)

Backup actual data stored in:
- Databases (PostgreSQL, MySQL, MongoDB)
- File storage (uploaded files, documents)
- Longhorn volumes
- NFS shares

### 3. Application-Level Backup

Application-specific backups:
- Database dumps (pg_dump, mysqldump)
- Application configuration
- Cache data (if needed)

## Velero Installation

### Prerequisites

```bash
# Install MinIO for S3-compatible storage
# See: docs/04-airgap-installation.md for airgap setup
```

### Install Velero

```bash
#!/bin/bash
# File: scripts/backup/install-velero.sh

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

VELERO_VERSION="v1.12.1"
MINIO_ENDPOINT="minio.k8s.internal:9000"
BUCKET_NAME="k8s-backups"

echo "=== Installing Velero ==="

# Download Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz
tar -xvf velero-${VELERO_VERSION}-linux-amd64.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
rm -rf velero-${VELERO_VERSION}-linux-amd64*

# Create MinIO credentials file
cat > /tmp/credentials-velero <<EOF
[default]
aws_access_key_id = minio-admin
aws_secret_access_key = minio-password-change-me
EOF

# Install Velero server
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.8.1 \
    --bucket ${BUCKET_NAME} \
    --secret-file /tmp/credentials-velero \
    --use-volume-snapshots=true \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://${MINIO_ENDPOINT} \
    --snapshot-location-config region=minio \
    --use-node-agent \
    --uploader-type=restic

# Clean up credentials
rm /tmp/credentials-velero

# Wait for Velero to be ready
kubectl wait --for=condition=ready pod -l component=velero -n velero --timeout=300s

echo "✓ Velero installed successfully"

# Verify installation
velero version
```

### Install Velero with Longhorn Integration

```bash
#!/bin/bash
# File: scripts/backup/install-velero-longhorn.sh

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Installing Velero with Longhorn CSI Plugin ==="

# Install Velero with CSI plugin
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.8.1,velero/velero-plugin-for-csi:v0.6.1 \
    --bucket k8s-backups \
    --secret-file /tmp/credentials-velero \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.k8s.internal:9000 \
    --use-node-agent \
    --features=EnableCSI \
    --uploader-type=restic

# Configure Longhorn VolumeSnapshotClass
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: longhorn-snapshot-vsc
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: snap
EOF

echo "✓ Velero with Longhorn integration installed"
```

## etcd Backup

### Automated etcd Snapshots (Built into RKE2)

RKE2 automatically takes etcd snapshots. Configure in RKE2 config:

```yaml
# File: /etc/rancher/rke2/config.yaml

etcd-snapshot-schedule-cron: "0 */6 * * *"  # Every 6 hours
etcd-snapshot-retention: 10  # Keep 10 snapshots
etcd-snapshot-dir: /var/lib/rancher/rke2/server/db/snapshots
etcd-s3: true  # Upload to S3
etcd-s3-endpoint: "minio.k8s.internal:9000"
etcd-s3-bucket: "etcd-backups"
etcd-s3-access-key: "minio-admin"
etcd-s3-secret-key: "minio-password"
etcd-s3-skip-ssl-verify: true  # For self-signed certs
```

### Manual etcd Backup

```bash
#!/bin/bash
# File: scripts/backup/backup-etcd.sh

# Manual etcd backup script

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
SNAPSHOT_DIR="/var/lib/rancher/rke2/server/db/snapshots"
BACKUP_NAME="etcd-manual-$(date +%Y%m%d-%H%M%S)"

echo "=== Creating etcd Snapshot ==="

# Get etcd pod name
ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Create snapshot
kubectl -n kube-system exec ${ETCD_POD} -- etcdctl \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    snapshot save ${SNAPSHOT_DIR}/${BACKUP_NAME}

# Verify snapshot
kubectl -n kube-system exec ${ETCD_POD} -- etcdctl \
    snapshot status ${SNAPSHOT_DIR}/${BACKUP_NAME} -w table

echo "✓ etcd snapshot created: ${BACKUP_NAME}"

# Copy to backup location
BACKUP_HOST="backup-server.k8s.internal"
scp ${SNAPSHOT_DIR}/${BACKUP_NAME} root@${BACKUP_HOST}:/backups/etcd/

echo "✓ Snapshot copied to backup server"
```

### Restore etcd from Snapshot

```bash
#!/bin/bash
# File: scripts/restore/restore-etcd.sh

# Restore etcd from snapshot

set -e

SNAPSHOT_FILE=$1

if [ -z "$SNAPSHOT_FILE" ]; then
    echo "Usage: $0 <snapshot-file>"
    exit 1
fi

echo "WARNING: This will restore etcd to the state in: $SNAPSHOT_FILE"
echo "All changes after the snapshot will be LOST!"
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Restore cancelled"
    exit 0
fi

# Stop RKE2 on all master nodes
echo "Stopping RKE2 on all masters..."
for master in master-01 master-02 master-03; do
    ssh $master "systemctl stop rke2-server"
done

# On first master, restore from snapshot
echo "Restoring from snapshot on master-01..."
ssh master-01 "rke2 server \
    --cluster-reset \
    --cluster-reset-restore-path=${SNAPSHOT_FILE}"

# Wait for restore to complete
sleep 30

# Start RKE2 on first master
ssh master-01 "systemctl start rke2-server"

# Wait for first master to be ready
echo "Waiting for master-01 to be ready..."
sleep 60

# Start RKE2 on other masters
for master in master-02 master-03; do
    echo "Starting RKE2 on $master..."
    ssh $master "systemctl start rke2-server"
    sleep 30
done

echo "✓ etcd restore complete"
echo "Verify cluster state with: kubectl get nodes"
```

## Persistent Volume Backup

### Create Backup with Velero

```bash
#!/bin/bash
# File: scripts/backup/backup-namespace.sh

# Backup entire namespace including PVs

NAMESPACE=$1

if [ -z "$NAMESPACE" ]; then
    echo "Usage: $0 <namespace>"
    exit 1
fi

BACKUP_NAME="${NAMESPACE}-$(date +%Y%m%d-%H%M%S)"

echo "=== Backing up namespace: ${NAMESPACE} ==="

velero backup create ${BACKUP_NAME} \
    --include-namespaces ${NAMESPACE} \
    --default-volumes-to-fs-backup \
    --wait

# Check backup status
velero backup describe ${BACKUP_NAME}

echo "✓ Backup complete: ${BACKUP_NAME}"
```

### Backup Specific Resources

```bash
#!/bin/bash
# File: scripts/backup/backup-resources.sh

# Backup specific resources with labels

LABEL=$1
BACKUP_NAME="labeled-backup-$(date +%Y%m%d-%H%M%S)"

if [ -z "$LABEL" ]; then
    echo "Usage: $0 <label-selector>"
    echo "Example: $0 app=database"
    exit 1
fi

echo "=== Backing up resources with label: ${LABEL} ==="

velero backup create ${BACKUP_NAME} \
    --selector ${LABEL} \
    --default-volumes-to-fs-backup \
    --wait

velero backup describe ${BACKUP_NAME}

echo "✓ Backup complete: ${BACKUP_NAME}"
```

### Exclude Resources from Backup

```bash
# Backup namespace but exclude certain resources
velero backup create production-backup \
    --include-namespaces production \
    --exclude-resources pods,replicasets \
    --default-volumes-to-fs-backup
```

## Application Backup

### Database Backup Example (PostgreSQL)

```bash
#!/bin/bash
# File: scripts/backup/backup-postgres.sh

# Backup PostgreSQL database running in Kubernetes

set -e

NAMESPACE="production"
DB_POD=$(kubectl -n ${NAMESPACE} get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}')
DB_NAME="myapp"
BACKUP_FILE="postgres-${DB_NAME}-$(date +%Y%m%d-%H%M%S).sql"
BACKUP_DIR="/backups/postgres"

mkdir -p ${BACKUP_DIR}

echo "=== Backing up PostgreSQL database: ${DB_NAME} ==="

# Create dump
kubectl -n ${NAMESPACE} exec ${DB_POD} -- \
    pg_dump -U postgres ${DB_NAME} > ${BACKUP_DIR}/${BACKUP_FILE}

# Compress
gzip ${BACKUP_DIR}/${BACKUP_FILE}

# Upload to S3/MinIO
mc cp ${BACKUP_DIR}/${BACKUP_FILE}.gz minio/backups/postgres/

echo "✓ Database backup complete: ${BACKUP_FILE}.gz"

# Cleanup old backups (keep last 30 days)
find ${BACKUP_DIR} -name "postgres-*.sql.gz" -mtime +30 -delete
```

### MongoDB Backup

```bash
#!/bin/bash
# File: scripts/backup/backup-mongodb.sh

# Backup MongoDB database

set -e

NAMESPACE="production"
MONGO_POD=$(kubectl -n ${NAMESPACE} get pods -l app=mongodb -o jsonpath='{.items[0].metadata.name}')
BACKUP_DIR="/backups/mongodb/$(date +%Y%m%d-%H%M%S)"

echo "=== Backing up MongoDB ==="

kubectl -n ${NAMESPACE} exec ${MONGO_POD} -- \
    mongodump --out=/tmp/backup --gzip

# Copy backup from pod
kubectl -n ${NAMESPACE} cp ${MONGO_POD}:/tmp/backup ${BACKUP_DIR}

# Upload to S3
mc cp -r ${BACKUP_DIR} minio/backups/mongodb/

echo "✓ MongoDB backup complete"
```

## Restore Procedures

### Restore from Velero Backup

```bash
#!/bin/bash
# File: scripts/restore/restore-namespace.sh

# Restore namespace from Velero backup

BACKUP_NAME=$1

if [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <backup-name>"
    echo "Available backups:"
    velero backup get
    exit 1
fi

echo "=== Restoring from backup: ${BACKUP_NAME} ==="

# Create restore
velero restore create --from-backup ${BACKUP_NAME} --wait

# Check restore status
velero restore describe ${BACKUP_NAME}

# Verify pods
NAMESPACE=$(velero backup describe ${BACKUP_NAME} | grep "Included Namespaces" | awk '{print $3}')
kubectl -n ${NAMESPACE} get pods

echo "✓ Restore complete"
```

### Restore to Different Namespace

```bash
# Restore to a different namespace (e.g., for testing)
velero restore create production-restore-test \
    --from-backup production-backup-20240101-120000 \
    --namespace-mappings production:production-test \
    --wait
```

### Restore Specific Resources

```bash
# Restore only specific resources
velero restore create selective-restore \
    --from-backup full-backup-20240101 \
    --include-resources deployments,services,configmaps \
    --wait
```

### Restore PostgreSQL Database

```bash
#!/bin/bash
# File: scripts/restore/restore-postgres.sh

# Restore PostgreSQL from SQL dump

set -e

BACKUP_FILE=$1
NAMESPACE="production"
DB_POD=$(kubectl -n ${NAMESPACE} get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}')

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.sql.gz>"
    exit 1
fi

echo "WARNING: This will overwrite the database!"
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Restore cancelled"
    exit 0
fi

echo "=== Restoring PostgreSQL database ==="

# Uncompress if needed
if [[ $BACKUP_FILE == *.gz ]]; then
    gunzip -c ${BACKUP_FILE} | kubectl -n ${NAMESPACE} exec -i ${DB_POD} -- \
        psql -U postgres myapp
else
    cat ${BACKUP_FILE} | kubectl -n ${NAMESPACE} exec -i ${DB_POD} -- \
        psql -U postgres myapp
fi

echo "✓ Database restore complete"
```

## Automated Backup Schedule

### Velero Backup Schedules

```yaml
# File: kubernetes/backup/backup-schedules.yaml

---
# Daily backup of production namespace
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2 AM every day
  template:
    includedNamespaces:
    - production
    defaultVolumesToFsBackup: true
    ttl: 720h  # Keep for 30 days

---
# Hourly backup of critical databases
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: databases-hourly
  namespace: velero
spec:
  schedule: "0 * * * *"  # Every hour
  template:
    labelSelector:
      matchLabels:
        backup: critical
    defaultVolumesToFsBackup: true
    ttl: 168h  # Keep for 7 days

---
# Weekly full cluster backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: cluster-weekly
  namespace: velero
spec:
  schedule: "0 3 * * 0"  # 3 AM every Sunday
  template:
    includedNamespaces:
    - "*"
    excludedNamespaces:
    - kube-system
    - kube-public
    - kube-node-lease
    defaultVolumesToFsBackup: true
    ttl: 2160h  # Keep for 90 days
```

### Apply Backup Schedules

```bash
#!/bin/bash
# File: scripts/backup/setup-schedules.sh

# Setup automated backup schedules

set -e

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

echo "=== Setting up Backup Schedules ==="

kubectl apply -f kubernetes/backup/backup-schedules.yaml

echo "✓ Backup schedules created"
echo ""
echo "View schedules:"
velero schedule get

echo ""
echo "View backups:"
velero backup get
```

## Backup Verification

### Verify Backup Integrity

```bash
#!/bin/bash
# File: scripts/backup/verify-backups.sh

# Verify backup integrity and completeness

set -e

echo "=== Verifying Backups ==="

# List all backups
echo "Recent backups:"
velero backup get

# Check backup details
LATEST_BACKUP=$(velero backup get -o json | jq -r '.items[0].metadata.name')

echo ""
echo "Checking latest backup: $LATEST_BACKUP"
velero backup describe $LATEST_BACKUP

# Check for errors
ERRORS=$(velero backup describe $LATEST_BACKUP | grep "Errors:" | awk '{print $2}')

if [ "$ERRORS" != "0" ]; then
    echo "✗ Backup has errors!"
    velero backup logs $LATEST_BACKUP
    exit 1
else
    echo "✓ Backup is clean (no errors)"
fi

# Verify etcd snapshots
echo ""
echo "etcd snapshots:"
ls -lh /var/lib/rancher/rke2/server/db/snapshots/ | tail -5
```

### Test Restore in Separate Namespace

```bash
#!/bin/bash
# File: scripts/backup/test-restore.sh

# Test restore by creating a test namespace

set -e

BACKUP_NAME=$1
TEST_NAMESPACE="restore-test-$(date +%s)"

if [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <backup-name>"
    exit 1
fi

echo "=== Testing Restore: ${BACKUP_NAME} ==="
echo "Target namespace: ${TEST_NAMESPACE}"

# Create test namespace
kubectl create namespace ${TEST_NAMESPACE}

# Restore to test namespace
ORIGINAL_NS=$(velero backup describe ${BACKUP_NAME} | grep "Included Namespaces" | awk '{print $3}')

velero restore create test-restore-$(date +%s) \
    --from-backup ${BACKUP_NAME} \
    --namespace-mappings ${ORIGINAL_NS}:${TEST_NAMESPACE} \
    --wait

# Verify
echo ""
echo "Restored resources:"
kubectl -n ${TEST_NAMESPACE} get all

# Cleanup prompt
echo ""
read -p "Delete test namespace? (yes/no): " CLEANUP
if [ "$CLEANUP" == "yes" ]; then
    kubectl delete namespace ${TEST_NAMESPACE}
    echo "✓ Test namespace deleted"
fi
```

## Backup Monitoring

### Prometheus Metrics for Velero

```yaml
# File: monitoring/prometheus/servicemonitor-velero.yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
  labels:
    app: velero
spec:
  selector:
    matchLabels:
      component: velero
  endpoints:
  - port: monitoring
    interval: 30s
```

### Alerts for Failed Backups

```yaml
# File: monitoring/prometheus/alerts-backup.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backup-alerts
  namespace: velero
spec:
  groups:
  - name: backup
    interval: 5m
    rules:
    - alert: VeleroBackupFailed
      expr: velero_backup_failure_total > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Velero backup has failed"
        description: "Velero backup {{ $labels.schedule }} has failed"
    
    - alert: VeleroBackupTooOld
      expr: time() - velero_backup_last_successful_timestamp > 86400
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Velero backup is too old"
        description: "Last successful backup was more than 24 hours ago"
```

## Backup Checklist

```yaml
backup_checklist:
  daily:
    - [ ] Verify automated backups completed
    - [ ] Check Velero backup status
    - [ ] Review etcd snapshots
    - [ ] Check backup logs for errors
  
  weekly:
    - [ ] Test restore to separate namespace
    - [ ] Verify backup storage capacity
    - [ ] Review retention policies
    - [ ] Update backup documentation
  
  monthly:
    - [ ] Full restore test (DR drill)
    - [ ] Backup encryption verification
    - [ ] Review and update RTO/RPO
    - [ ] Audit backup access logs
  
  quarterly:
    - [ ] DR failover test
    - [ ] Review backup strategy
    - [ ] Update runbooks
    - [ ] Training for team
```

## Next Steps

- ➡️ Setup automated backup schedules
- 🧪 Perform regular restore tests
- 📊 Monitor backup metrics
- 📝 Document restore procedures
- 🔄 Regular DR drills
