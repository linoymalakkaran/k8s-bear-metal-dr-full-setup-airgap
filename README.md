# Production On-Premise Kubernetes Cluster with Rancher

## Overview
This project provides a comprehensive guide and automation scripts for deploying a production-grade on-premise Kubernetes cluster using Rancher, with full Disaster Recovery (DR) capabilities.

## Project Structure
```
dr-k8s/
├── README.md                          # This file
├── QUICKSTART.md                      # 30-minute quick start guide
├── PROJECT_SUMMARY.md                 # Complete feature overview
├── docs/                              # Comprehensive documentation
│   ├── 00-bare-metal-setup.md         # NEW: Complete bare metal server setup
│   ├── 01-infrastructure-requirements.md
│   ├── 02-network-design.md
│   ├── 03-installation-guide.md
│   ├── 04-airgap-installation.md
│   ├── airgap-images-manifest.md      # NEW: Complete airgap image inventory
│   ├── master-worker-setup.md         # NEW: Master/worker/etcd setup guide
│   ├── 05-disaster-recovery.md
│   ├── 06-monitoring-observability.md
│   ├── 07-backup-restore.md
│   └── 08-maintenance-operations.md
├── infrastructure/                    # Infrastructure as Code
│   ├── terraform/                     # Terraform configurations (if using)
│   ├── ansible/                       # Ansible playbooks for provisioning
│   └── requirements.yaml              # Infrastructure requirements spec
├── rancher/                           # Rancher setup
│   ├── installation/                  # Installation scripts
│   ├── config/                        # Configuration files
│   └── backup/                        # Backup configurations
├── kubernetes/                        # Kubernetes configurations
│   ├── cluster-config/                # Cluster configuration
│   ├── namespaces/                    # Namespace definitions
│   ├── rbac/                          # RBAC policies
│   └── network-policies/              # Network policies
├── helm-charts/                       # Helm charts
│   ├── applications/                  # Application charts
│   └── infrastructure/                # Infrastructure charts
├── scripts/                           # Automation scripts
│   ├── setup/                         # Setup scripts
│   ├── backup/                        # Backup scripts
│   ├── restore/                       # Restore scripts
│   └── maintenance/                   # Maintenance scripts
├── pipelines/                         # CI/CD pipelines
│   ├── jenkins/                       # Jenkins pipelines
│   ├── gitlab-ci/                     # GitLab CI configurations
│   └── github-actions/                # GitHub Actions workflows
├── monitoring/                        # Monitoring setup
│   ├── prometheus/                    # Prometheus configurations
│   ├── grafana/                       # Grafana dashboards
│   └── alertmanager/                  # Alert configurations
└── dr/                                # Disaster Recovery
    ├── runbooks/                      # DR runbooks
    ├── scripts/                       # DR automation scripts
    └── configs/                       # DR configurations
```

## Quick Start

See [QUICKSTART.md](QUICKSTART.md) for a complete 30-minute deployment guide.

### Prerequisites
- **Bare metal servers** or VMs (3 masters + 3-5 workers minimum)
- **OS**: Rocky Linux 8.8+, RHEL 8+, or Ubuntu 22.04 LTS
- **Hardware**: See [Infrastructure Requirements](docs/01-infrastructure-requirements.md)
- Root or sudo access on all nodes
- Network connectivity between nodes
- (Optional) Air-gapped environment setup

### Installation Path

**For Bare Metal Deployment:**
1. 🖥️ **[Bare Metal Setup](docs/00-bare-metal-setup.md)** - Complete hardware and OS installation
   - BIOS/UEFI configuration
   - RAID setup
   - OS installation (Rocky Linux 8)
   - Disk partitioning
   - Network configuration
   
2. 🏗️ **[Infrastructure Requirements](docs/01-infrastructure-requirements.md)** - Hardware specifications
   
3. 🌐 **[Network Design](docs/02-network-design.md)** - VLAN design, HAProxy, firewall rules

4. 🎯 **[Master & Worker Setup](docs/master-worker-setup.md)** - Complete cluster bootstrapping
   - Bootstrap first master node
   - Setup 3-node etcd cluster
   - Join additional masters
   - Add worker nodes
   
5. 📦 **[Installation Guide](docs/03-installation-guide.md)** - RKE2 and Rancher installation

**For Airgap Deployment:**
- 📥 **[Airgap Images Manifest](docs/airgap-images-manifest.md)** - Complete image list with sizes
- 🔒 **[Airgap Installation](docs/04-airgap-installation.md)** - Offline installation procedures

**Post-Installation:**
- 🔄 **[Disaster Recovery](docs/05-disaster-recovery.md)** - DR setup and failover
- 📊 **[Monitoring & Observability](docs/06-monitoring-observability.md)** - Complete monitoring stack

## Key Features

### Production-Ready Setup
- High Availability (HA) configuration
- Multi-master setup for control plane
- Load balancing for API server
- etcd backup and restore capabilities

### Disaster Recovery
- Automated backup solutions
- Multi-region DR strategy
- RTO/RPO optimization
- Failover automation

### Security
- Network policies and segmentation
- RBAC implementation
- Pod Security Policies/Standards
- Secrets management
- Certificate management

### Monitoring & Observability
- Prometheus metrics collection
- Grafana dashboards
- Centralized logging (ELK/Loki)
- Alerting and notifications

### Airgap Support
- Complete offline installation guide
- Private container registry setup
- Helm chart repository hosting
- Image synchronization strategies

## Architecture

### Production Cluster Design
```
┌─────────────────────────────────────────────────────────────┐
│                      Load Balancer (HAProxy/Nginx)          │
│                    (VIP: 192.168.1.100)                      │
└───────────────┬─────────────────┬─────────────────┬─────────┘
                │                 │                 │
        ┌───────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
        │ Master Node 1│  │ Master Node 2│  │ Master Node 3│
        │  (Control)   │  │  (Control)   │  │  (Control)   │
        │  + etcd      │  │  + etcd      │  │  + etcd      │
        └──────────────┘  └──────────────┘  └──────────────┘
                │                 │                 │
        ┌───────┴─────────────────┴─────────────────┴────────┐
        │                                                     │
┌───────▼──────┐  ┌──────────────┐  ┌──────────────┐ ┌──────▼──────┐
│ Worker Node 1│  │ Worker Node 2│  │ Worker Node 3│ │ Worker Node N│
│ (Workloads)  │  │ (Workloads)  │  │ (Workloads)  │ │ (Workloads)  │
└──────────────┘  └──────────────┘  └──────────────┘ └──────────────┘
```

### DR Architecture
```
┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐
│      Primary Data Center (DC1)       │     │    DR Data Center (DC2)             │
│                                      │     │                                     │
│  ┌────────────────────────────────┐ │     │  ┌────────────────────────────────┐│
│  │   Production K8s Cluster        │ │     │  │   DR K8s Cluster               ││
│  │   - 3 Master Nodes              │ │     │  │   - 3 Master Nodes             ││
│  │   - N Worker Nodes              │ │────►│  │   - N Worker Nodes             ││
│  │   - Rancher Management          │ │     │  │   - Rancher Management         ││
│  └────────────────────────────────┘ │     │  └────────────────────────────────┘│
│                                      │     │                                     │
│  ┌────────────────────────────────┐ │     │  ┌────────────────────────────────┐│
│  │   Storage (Persistent)          │ │     │  │   Storage (Replicated)         ││
│  │   - Longhorn/Rook-Ceph          │◄├────►│  │   - Longhorn/Rook-Ceph         ││
│  └────────────────────────────────┘ │     │  └────────────────────────────────┘│
└─────────────────────────────────────┘     └─────────────────────────────────────┘
              │                                             ▲
              │                                             │
              └─────────── etcd Backup / Replication ───────┘
```

## Infrastructure Requirements Summary

### Minimum Production Setup
- **Master Nodes**: 3 nodes (HA)
  - 4 vCPU, 16GB RAM, 100GB SSD each
- **Worker Nodes**: 3+ nodes (scalable)
  - 8 vCPU, 32GB RAM, 200GB SSD each
- **Load Balancer**: 2 nodes (HA)
  - 2 vCPU, 4GB RAM, 50GB disk each
- **Storage**: Distributed storage solution (Longhorn/Rook-Ceph)
  - Additional disks for persistent storage

See detailed requirements in [Infrastructure Requirements](docs/01-infrastructure-requirements.md)

## Support & Maintenance

### Regular Maintenance Tasks
- etcd backup verification
- Certificate renewal
- Security updates
- Cluster upgrades
- Capacity planning

### Monitoring Endpoints
- Rancher UI: `https://rancher.example.com`
- Grafana: `https://grafana.example.com`
- Prometheus: `https://prometheus.example.com`

## Contributing
This is an internal production infrastructure project. Please follow the change management process for any modifications.

## License
Internal Use Only - [Your Organization]

## Authors
- Infrastructure Team
- DevOps Team

## Last Updated
January 2026
# k8s-bear-metal-dr-full-setup-airgap
