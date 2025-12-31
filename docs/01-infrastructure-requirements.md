# Infrastructure Requirements for Production Kubernetes Cluster

## Table of Contents
1. [Overview](#overview)
2. [Hardware Requirements](#hardware-requirements)
3. [Network Requirements](#network-requirements)
4. [Storage Requirements](#storage-requirements)
5. [Software Requirements](#software-requirements)
6. [DR Requirements](#dr-requirements)

## Overview

This document outlines the complete infrastructure requirements for deploying a production-grade on-premise Kubernetes cluster using Rancher with full High Availability (HA) and Disaster Recovery (DR) capabilities.

## Hardware Requirements

### Cluster Architecture Sizing

#### Small Production Cluster (up to 100 pods)
```yaml
cluster_size: small
total_nodes: 6
description: Suitable for small production workloads

master_nodes:
  count: 3
  role: Control plane + etcd
  specs:
    cpu: 4 vCPU
    memory: 16 GB RAM
    disk_os: 100 GB SSD
    disk_etcd: 50 GB SSD (dedicated)
    network: 1 Gbps

worker_nodes:
  count: 3
  role: Workload execution
  specs:
    cpu: 8 vCPU
    memory: 32 GB RAM
    disk_os: 100 GB SSD
    disk_data: 200 GB SSD
    network: 1 Gbps
```

#### Medium Production Cluster (100-500 pods)
```yaml
cluster_size: medium
total_nodes: 10
description: Suitable for medium production workloads

master_nodes:
  count: 3
  role: Control plane + etcd
  specs:
    cpu: 8 vCPU
    memory: 32 GB RAM
    disk_os: 200 GB SSD
    disk_etcd: 100 GB SSD (dedicated)
    network: 10 Gbps

worker_nodes:
  count: 5-7
  role: Workload execution
  specs:
    cpu: 16 vCPU
    memory: 64 GB RAM
    disk_os: 200 GB SSD
    disk_data: 500 GB SSD
    network: 10 Gbps
```

#### Large Production Cluster (500-2000 pods)
```yaml
cluster_size: large
total_nodes: 15+
description: Suitable for large production workloads

master_nodes:
  count: 3 (or 5 for extra redundancy)
  role: Control plane + etcd
  specs:
    cpu: 16 vCPU
    memory: 64 GB RAM
    disk_os: 500 GB SSD
    disk_etcd: 200 GB SSD (dedicated, NVMe preferred)
    network: 10 Gbps

worker_nodes:
  count: 10+
  role: Workload execution
  specs:
    cpu: 32 vCPU
    memory: 128 GB RAM
    disk_os: 500 GB SSD
    disk_data: 1 TB NVMe SSD
    network: 10 Gbps
```

### Load Balancer Nodes (HA Pair)
```yaml
load_balancer:
  count: 2
  purpose: HA proxy for K8s API server
  software: HAProxy or Nginx
  specs:
    cpu: 4 vCPU
    memory: 8 GB RAM
    disk: 50 GB SSD
    network: 10 Gbps
  configuration:
    - keepalived for VIP management
    - health checks for backend nodes
```

### Rancher Management Server (Optional Dedicated)
```yaml
rancher_server:
  count: 1 (or 3 for HA)
  purpose: Rancher management plane
  deployment_option: "Dedicated or on K8s cluster"
  specs:
    cpu: 8 vCPU
    memory: 16 GB RAM
    disk: 200 GB SSD
    network: 1 Gbps
```

### Bastion/Jump Host
```yaml
bastion_host:
  count: 1-2
  purpose: Secure access gateway
  specs:
    cpu: 2 vCPU
    memory: 4 GB RAM
    disk: 50 GB
    network: 1 Gbps
```

## Network Requirements

### Network Topology
```
Internet/Corporate Network
        │
        ▼
┌─────────────────┐
│  Firewall/DMZ   │
└────────┬────────┘
         │
    ┌────┴────┐
    │   VIP   │ (Floating IP for API access)
    └────┬────┘
         │
    ┌────┴────────────────────────┐
    │   Load Balancer Subnet      │
    │   192.168.1.0/28            │
    │   LB1: .10, LB2: .11        │
    │   VIP: .100                 │
    └────┬────────────────────────┘
         │
    ┌────┴────────────────────────┐
    │   Management Subnet         │
    │   192.168.10.0/24           │
    │   Masters: .10-.12          │
    │   Rancher: .20              │
    └────┬────────────────────────┘
         │
    ┌────┴────────────────────────┐
    │   Worker Subnet             │
    │   192.168.20.0/24           │
    │   Workers: .10-.50          │
    └────┬────────────────────────┘
         │
    ┌────┴────────────────────────┐
    │   Pod Network (CNI)         │
    │   10.42.0.0/16              │
    └─────────────────────────────┘
         │
    ┌────┴────────────────────────┐
    │   Service Network           │
    │   10.43.0.0/16              │
    └─────────────────────────────┘
```

### IP Address Planning

#### Management Network (192.168.10.0/24)
```yaml
management_network:
  subnet: 192.168.10.0/24
  gateway: 192.168.10.1
  dns_servers: [192.168.10.2, 192.168.10.3]
  
  assignments:
    masters:
      - master-01: 192.168.10.10
      - master-02: 192.168.10.11
      - master-03: 192.168.10.12
    
    infrastructure:
      - rancher-01: 192.168.10.20
      - bastion-01: 192.168.10.30
      - ntp-server: 192.168.10.40
    
    load_balancers:
      - lb-01: 192.168.1.10
      - lb-02: 192.168.1.11
      - api-vip: 192.168.1.100
```

#### Worker Network (192.168.20.0/24)
```yaml
worker_network:
  subnet: 192.168.20.0/24
  gateway: 192.168.20.1
  
  assignments:
    workers:
      - worker-01: 192.168.20.10
      - worker-02: 192.168.20.11
      - worker-03: 192.168.20.12
      # ... up to worker-N
```

#### Kubernetes Internal Networks
```yaml
kubernetes_networks:
  pod_network:
    cidr: 10.42.0.0/16
    cni: "Canal (Calico + Flannel) or Cilium"
    
  service_network:
    cidr: 10.43.0.0/16
    cluster_dns: 10.43.0.10
```

### Port Requirements

#### Master Nodes (Control Plane)
```yaml
master_node_ports:
  inbound:
    - protocol: TCP
      port: 6443
      source: ALL
      description: "Kubernetes API Server"
    
    - protocol: TCP
      port: 2379-2380
      source: MASTER_NODES
      description: "etcd server client API"
    
    - protocol: TCP
      port: 10250
      source: MASTER_NODES, WORKER_NODES
      description: "Kubelet API"
    
    - protocol: TCP
      port: 10251
      source: LOCALHOST
      description: "kube-scheduler"
    
    - protocol: TCP
      port: 10252
      source: LOCALHOST
      description: "kube-controller-manager"
    
    - protocol: TCP
      port: 9099
      source: RANCHER
      description: "Rancher agent"
    
    - protocol: TCP
      port: 22
      source: BASTION
      description: "SSH"
```

#### Worker Nodes
```yaml
worker_node_ports:
  inbound:
    - protocol: TCP
      port: 10250
      source: MASTER_NODES
      description: "Kubelet API"
    
    - protocol: TCP
      port: 30000-32767
      source: ALL
      description: "NodePort Services"
    
    - protocol: TCP
      port: 9099
      source: RANCHER
      description: "Rancher agent"
    
    - protocol: TCP
      port: 22
      source: BASTION
      description: "SSH"
    
    - protocol: TCP/UDP
      port: 8472
      source: ALL_NODES
      description: "Flannel VXLAN"
    
    - protocol: UDP
      port: 4789
      source: ALL_NODES
      description: "Flannel VXLAN (alternative)"
```

#### Rancher Server
```yaml
rancher_ports:
  inbound:
    - protocol: TCP
      port: 80
      source: USERS
      description: "HTTP (redirect to HTTPS)"
    
    - protocol: TCP
      port: 443
      source: USERS
      description: "HTTPS UI/API"
    
    - protocol: TCP
      port: 22
      source: BASTION
      description: "SSH"
```

### Network Performance Requirements
```yaml
network_requirements:
  bandwidth:
    inter_node: "10 Gbps (minimum 1 Gbps)"
    internet: "100 Mbps (minimum)"
  
  latency:
    master_to_master: "< 2ms (same datacenter)"
    master_to_worker: "< 5ms"
    cross_datacenter_dr: "< 100ms acceptable"
  
  features_required:
    - "VLAN support for network segmentation"
    - "Jumbo frames support (MTU 9000)"
    - "Multicast support (for some CNI plugins)"
    - "QoS capabilities"
```

## Storage Requirements

### etcd Storage
```yaml
etcd_storage:
  type: "Dedicated SSD/NVMe"
  size_minimum: 50 GB
  size_recommended: 100 GB
  iops: "> 3000 IOPS"
  notes:
    - "Must be low latency storage"
    - "Separate disk from OS for performance"
    - "Critical for cluster performance"
```

### Operating System Storage
```yaml
os_storage:
  type: SSD
  size_master: "100-500 GB"
  size_worker: "100-500 GB"
  filesystem: "ext4 or xfs"
  mount_points:
    - "/: 50 GB minimum"
    - "/var: 100 GB minimum (container images, logs)"
    - "/var/lib/rancher: 50 GB"
```

### Persistent Storage Solutions

#### Option 1: Longhorn (Recommended for K8s)
```yaml
longhorn:
  description: "Cloud-native distributed block storage"
  requirements:
    worker_nodes: "Minimum 3 nodes"
    disk_per_node: "200 GB - 2 TB SSD"
    replicas: 3
  features:
    - "Automatic replication"
    - "Backup to S3/NFS"
    - "Snapshot support"
    - "DR capabilities"
```

#### Option 2: Rook-Ceph
```yaml
rook_ceph:
  description: "Enterprise-grade distributed storage"
  requirements:
    dedicated_nodes: "3-5 nodes (or use worker nodes)"
    disk_per_node: "500 GB - 10 TB"
    network: "10 Gbps dedicated storage network"
  features:
    - "Block, file, and object storage"
    - "High availability"
    - "Replication and erasure coding"
```

#### Option 3: External Storage (NFS/iSCSI)
```yaml
external_storage:
  nfs:
    server: "nas.example.com"
    path: "/k8s-storage"
    size: "5 TB minimum"
  
  iscsi:
    target: "san.example.com"
    luns: "Multiple LUNs"
    multipath: true
```

### Backup Storage
```yaml
backup_storage:
  location: "Separate physical location from production"
  type: "NFS, S3-compatible, or tape"
  size: "2x production data size"
  retention:
    daily: "7 days"
    weekly: "4 weeks"
    monthly: "12 months"
```

## Software Requirements

### Operating System

#### Supported Linux Distributions
```yaml
supported_os:
  recommended:
    - name: "Rocky Linux"
      versions: ["8.x", "9.x"]
      notes: "RHEL clone, enterprise-grade"
    
    - name: "Ubuntu Server"
      versions: ["20.04 LTS", "22.04 LTS"]
      notes: "Wide Rancher support"
    
    - name: "RHEL"
      versions: ["8.x", "9.x"]
      notes: "Enterprise support available"
  
  also_supported:
    - "CentOS Stream 8/9"
    - "SLES 15 SP3+"
    - "Oracle Linux 8+"
```

#### OS Configuration Requirements
```bash
# Kernel version
minimum_kernel_version: "4.18+"
recommended_kernel_version: "5.4+"

# Required kernel modules
required_modules:
  - br_netfilter
  - ip_vs
  - ip_vs_rr
  - ip_vs_wrr
  - ip_vs_sh
  - overlay
  - nf_conntrack

# System settings
swap: "disabled"
selinux: "permissive or disabled (production should use permissive)"
firewall: "configured (firewalld or iptables)"
```

### Container Runtime

#### Recommended: containerd
```yaml
containerd:
  version: "1.6.x or 1.7.x"
  configuration:
    - "systemd cgroup driver"
    - "proper storage configuration"
  
  installation_method:
    - "Package manager (yum/apt)"
    - "Official binaries"
```

#### Alternative: Docker (via cri-dockerd)
```yaml
docker:
  version: "20.10.x or newer"
  note: "Requires cri-dockerd adapter for K8s 1.24+"
  not_recommended: "Use containerd instead"
```

### Kubernetes Version
```yaml
kubernetes:
  rancher_supported_versions:
    - "1.25.x"
    - "1.26.x"
    - "1.27.x"
    - "1.28.x"
  
  installation_method: "Rancher RKE2 (recommended)"
  
  alternatives:
    - "RKE (Rancher Kubernetes Engine)"
    - "K3s (lightweight clusters)"
```

### Rancher Version
```yaml
rancher:
  version: "2.7.x or 2.8.x (latest stable)"
  deployment_options:
    option_1:
      name: "Rancher on Kubernetes"
      description: "HA setup on K8s cluster"
      recommended: true
    
    option_2:
      name: "Docker installation"
      description: "Single node testing only"
      recommended: false
```

### Additional Software Components
```yaml
additional_software:
  load_balancer:
    - name: "HAProxy"
      version: "2.4+"
    - name: "Nginx"
      version: "1.20+"
  
  monitoring:
    - name: "Prometheus"
      version: "2.40+"
    - name: "Grafana"
      version: "9.x+"
  
  logging:
    - name: "Elasticsearch/OpenSearch"
      version: "8.x / 2.x"
    - name: "Loki"
      version: "2.8+"
  
  backup:
    - name: "Velero"
      version: "1.11+"
    - name: "Kasten K10"
      version: "5.x+"
  
  service_mesh_optional:
    - name: "Istio"
      version: "1.18+"
    - name: "Linkerd"
      version: "2.13+"
```

## DR Requirements

### DR Site Infrastructure
```yaml
dr_site:
  location: "Separate geographical location"
  distance: "> 50 km from primary"
  
  infrastructure:
    mirror_primary: true
    minimum_specs:
      - "Same number of master nodes (3)"
      - "Same or N-1 worker nodes"
      - "Equal or better storage capacity"
  
  connectivity:
    primary_to_dr:
      - "Dedicated WAN link or VPN"
      - "Minimum 100 Mbps"
      - "Latency < 100ms"
```

### Replication Requirements
```yaml
replication:
  storage_replication:
    method: "Longhorn cross-cluster replication"
    frequency: "Continuous or hourly"
  
  etcd_backup:
    frequency: "Every 6 hours"
    destination: "DR site storage"
    retention: "30 days"
  
  application_data:
    method: "Velero backups"
    frequency: "Daily"
    destination: "S3-compatible storage in DR site"
```

### RTO/RPO Targets
```yaml
business_continuity:
  rto: "< 4 hours (Recovery Time Objective)"
  rpo: "< 1 hour (Recovery Point Objective)"
  
  automation:
    - "Automated failover scripts"
    - "Health monitoring"
    - "Automated testing (monthly)"
```

## Summary Checklist

### Pre-Installation Checklist
```markdown
Hardware:
- [ ] Master nodes provisioned (3 minimum)
- [ ] Worker nodes provisioned (3+ minimum)
- [ ] Load balancer nodes provisioned (2)
- [ ] Storage capacity allocated
- [ ] DR site infrastructure ready

Network:
- [ ] IP addresses assigned
- [ ] DNS records configured
- [ ] Firewall rules configured
- [ ] Load balancer configured
- [ ] Network speed verified (10 Gbps recommended)

Software:
- [ ] OS installed and updated
- [ ] Container runtime installed
- [ ] Required kernel modules loaded
- [ ] Swap disabled
- [ ] Time synchronization configured (NTP)
- [ ] SSH keys configured

Access:
- [ ] Root or sudo access verified
- [ ] Bastion host configured
- [ ] VPN access (if required)
- [ ] Certificate authorities prepared

Documentation:
- [ ] Network diagram created
- [ ] IP address spreadsheet maintained
- [ ] Credentials securely stored
- [ ] Runbooks prepared
```

## Next Steps

After verifying all infrastructure requirements:
1. Proceed to [Network Design](02-network-design.md)
2. Review [Installation Guide](03-installation-guide.md)
3. Plan [Disaster Recovery](05-disaster-recovery.md) implementation
