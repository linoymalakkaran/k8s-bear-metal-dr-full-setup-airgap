# Network Design and Configuration

## Table of Contents
1. [Network Architecture](#network-architecture)
2. [Network Segmentation](#network-segmentation)
3. [Load Balancer Configuration](#load-balancer-configuration)
4. [CNI Plugin Selection](#cni-plugin-selection)
5. [Network Policies](#network-policies)
6. [DNS Configuration](#dns-configuration)
7. [Firewall Rules](#firewall-rules)

## Network Architecture

### Multi-Tier Network Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Internet / Corporate Network                  │
│                                 ↕                                    │
│                        ┌──────────────────┐                         │
│                        │   Firewall/NAT   │                         │
│                        └────────┬─────────┘                         │
└─────────────────────────────────┼──────────────────────────────────┘
                                  │
                        ┌─────────▼──────────┐
                        │   DMZ Zone (Opt)   │
                        │   Ingress/Egress   │
                        └─────────┬──────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │  Load Balancer VIP Layer   │
                    │  192.168.1.100 (Floating)  │
                    │  ┌──────┐      ┌──────┐   │
                    │  │ LB-1 │      │ LB-2 │   │
                    │  └──────┘      └──────┘   │
                    └─────────────┬──────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────▼─────────┐ ┌──────▼────────┐ ┌───────▼────────┐
    │ Management Network│ │Worker Network  │ │Storage Network │
    │  192.168.10.0/24  │ │192.168.20.0/24 │ │192.168.30.0/24 │
    │                   │ │                │ │                │
    │ Masters (3)       │ │Workers (N)     │ │Storage Nodes   │
    │ Rancher           │ │                │ │                │
    │ Bastion           │ │                │ │                │
    └───────────────────┘ └────────────────┘ └────────────────┘
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │   Kubernetes Overlay       │
                    │   Pod Network: 10.42.0.0/16│
                    │   Svc Network: 10.43.0.0/16│
                    └────────────────────────────┘
```

## Network Segmentation

### VLAN Configuration

#### Production Environment VLANs
```yaml
vlans:
  # Management VLAN - Control Plane and Administrative Access
  vlan_10:
    id: 10
    name: "management"
    subnet: "192.168.10.0/24"
    gateway: "192.168.10.1"
    purpose: "Master nodes, Rancher, Bastion"
    security_level: "high"
    
  # Worker VLAN - Application Workloads
  vlan_20:
    id: 20
    name: "workers"
    subnet: "192.168.20.0/24"
    gateway: "192.168.20.1"
    purpose: "Worker nodes"
    security_level: "medium"
    
  # Storage VLAN - Persistent Storage Traffic
  vlan_30:
    id: 30
    name: "storage"
    subnet: "192.168.30.0/24"
    gateway: "192.168.30.1"
    purpose: "Storage backend (Longhorn, Ceph)"
    security_level: "high"
    jumbo_frames: true  # MTU 9000
    
  # Load Balancer VLAN
  vlan_5:
    id: 5
    name: "loadbalancer"
    subnet: "192.168.1.0/28"
    gateway: "192.168.1.1"
    purpose: "Load balancers and VIP"
    security_level: "high"
```

### IP Address Allocation Strategy

#### Management Network (VLAN 10 - 192.168.10.0/24)
```yaml
management_network:
  vlan: 10
  subnet: "192.168.10.0/24"
  gateway: "192.168.10.1"
  dns_servers: ["192.168.10.2", "192.168.10.3"]
  ntp_server: "192.168.10.4"
  
  # IP Range Allocation
  ranges:
    infrastructure: "192.168.10.1-192.168.10.9"
    masters: "192.168.10.10-192.168.10.19"
    management_services: "192.168.10.20-192.168.10.49"
    reserved: "192.168.10.50-192.168.10.99"
    dhcp_reserved: "192.168.10.100-192.168.10.200"
  
  # Static Assignments
  static_ips:
    # Infrastructure
    gateway: 192.168.10.1
    dns_primary: 192.168.10.2
    dns_secondary: 192.168.10.3
    ntp_server: 192.168.10.4
    
    # Master Nodes
    master_01: 192.168.10.10
    master_02: 192.168.10.11
    master_03: 192.168.10.12
    
    # Management Services
    rancher_server: 192.168.10.20
    harbor_registry: 192.168.10.21
    gitlab: 192.168.10.22
    bastion_host: 192.168.10.30
    ansible_controller: 192.168.10.31
```

#### Worker Network (VLAN 20 - 192.168.20.0/24)
```yaml
worker_network:
  vlan: 20
  subnet: "192.168.20.0/24"
  gateway: "192.168.20.1"
  
  # IP Range Allocation
  ranges:
    infrastructure: "192.168.20.1-192.168.20.9"
    workers: "192.168.20.10-192.168.20.99"
    reserved_expansion: "192.168.20.100-192.168.20.200"
  
  # Static Assignments for Workers
  static_ips:
    worker_01: 192.168.20.10
    worker_02: 192.168.20.11
    worker_03: 192.168.20.12
    worker_04: 192.168.20.13
    worker_05: 192.168.20.14
    # ... scale as needed
```

#### Storage Network (VLAN 30 - 192.168.30.0/24)
```yaml
storage_network:
  vlan: 30
  subnet: "192.168.30.0/24"
  gateway: "192.168.30.1"
  mtu: 9000  # Jumbo frames for storage performance
  
  # Dedicated storage interfaces on all nodes
  static_ips:
    # Masters with storage interface
    master_01_storage: 192.168.30.10
    master_02_storage: 192.168.30.11
    master_03_storage: 192.168.30.12
    
    # Workers with storage interface
    worker_01_storage: 192.168.30.20
    worker_02_storage: 192.168.30.21
    worker_03_storage: 192.168.30.22
    # ... continue
    
    # Dedicated storage nodes (if using Ceph)
    storage_01: 192.168.30.100
    storage_02: 192.168.30.101
    storage_03: 192.168.30.102
```

#### Load Balancer Network (VLAN 5 - 192.168.1.0/28)
```yaml
loadbalancer_network:
  vlan: 5
  subnet: "192.168.1.0/28"
  gateway: "192.168.1.1"
  
  static_ips:
    lb_01: 192.168.1.10
    lb_02: 192.168.1.11
    api_vip: 192.168.1.100        # Floating VIP for K8s API
    rancher_vip: 192.168.1.101    # Floating VIP for Rancher UI
    ingress_vip: 192.168.1.102    # Floating VIP for Ingress
```

## Load Balancer Configuration

### HAProxy Configuration (Recommended)

#### HAProxy Installation and Setup
```bash
#!/bin/bash
# File: scripts/setup/install-haproxy.sh

# Install HAProxy on both LB nodes
yum install -y haproxy keepalived

# Enable and configure services
systemctl enable haproxy
systemctl enable keepalived
```

#### HAProxy Configuration File
```haproxy
# File: infrastructure/loadbalancer/haproxy.cfg

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    
    # SSL/TLS settings
    tune.ssl.default-dh-param 2048
    ssl-default-bind-ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# Kubernetes API Server (6443)
#---------------------------------------------------------------------
frontend k8s_api_frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    
    # Master nodes
    server master-01 192.168.10.10:6443 check fall 3 rise 2
    server master-02 192.168.10.11:6443 check fall 3 rise 2
    server master-03 192.168.10.12:6443 check fall 3 rise 2

#---------------------------------------------------------------------
# Rancher HTTPS (443)
#---------------------------------------------------------------------
frontend rancher_https_frontend
    bind *:443
    mode tcp
    option tcplog
    default_backend rancher_https_backend

backend rancher_https_backend
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    
    # Rancher servers (if HA Rancher deployment)
    server rancher-01 192.168.10.20:443 check fall 3 rise 2
    # Add more Rancher nodes if deployed in HA
    # server rancher-02 192.168.10.21:443 check fall 3 rise 2
    # server rancher-03 192.168.10.22:443 check fall 3 rise 2

#---------------------------------------------------------------------
# Rancher HTTP (80) - Redirect to HTTPS
#---------------------------------------------------------------------
frontend rancher_http_frontend
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

#---------------------------------------------------------------------
# Ingress Controller HTTP (80)
#---------------------------------------------------------------------
frontend ingress_http_frontend
    bind *:8080
    mode tcp
    option tcplog
    default_backend ingress_http_backend

backend ingress_http_backend
    mode tcp
    balance roundrobin
    
    # Worker nodes (NodePort or direct Ingress)
    server worker-01 192.168.20.10:80 check fall 3 rise 2
    server worker-02 192.168.20.11:80 check fall 3 rise 2
    server worker-03 192.168.20.12:80 check fall 3 rise 2

#---------------------------------------------------------------------
# Ingress Controller HTTPS (443) - Alternative port
#---------------------------------------------------------------------
frontend ingress_https_frontend
    bind *:8443
    mode tcp
    option tcplog
    default_backend ingress_https_backend

backend ingress_https_backend
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    
    # Worker nodes
    server worker-01 192.168.20.10:443 check fall 3 rise 2
    server worker-02 192.168.20.11:443 check fall 3 rise 2
    server worker-03 192.168.20.12:443 check fall 3 rise 2

#---------------------------------------------------------------------
# Statistics Page
#---------------------------------------------------------------------
listen stats
    bind *:9000
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:your-secure-password
    stats refresh 30s
```

#### Keepalived Configuration (VIP Management)

##### Master LB (lb-01) - keepalived.conf
```bash
# File: infrastructure/loadbalancer/keepalived-master.conf

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# VIP for Kubernetes API
vrrp_instance VI_K8S_API {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass k8s_secret_2024
    }
    
    virtual_ipaddress {
        192.168.1.100/28
    }
    
    track_script {
        check_haproxy
    }
}

# VIP for Rancher UI
vrrp_instance VI_RANCHER {
    state MASTER
    interface eth0
    virtual_router_id 52
    priority 101
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass rancher_secret_2024
    }
    
    virtual_ipaddress {
        192.168.1.101/28
    }
    
    track_script {
        check_haproxy
    }
}
```

##### Backup LB (lb-02) - keepalived.conf
```bash
# File: infrastructure/loadbalancer/keepalived-backup.conf

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# VIP for Kubernetes API
vrrp_instance VI_K8S_API {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100  # Lower than master
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass k8s_secret_2024
    }
    
    virtual_ipaddress {
        192.168.1.100/28
    }
    
    track_script {
        check_haproxy
    }
}

# VIP for Rancher UI
vrrp_instance VI_RANCHER {
    state BACKUP
    interface eth0
    virtual_router_id 52
    priority 100  # Lower than master
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass rancher_secret_2024
    }
    
    virtual_ipaddress {
        192.168.1.101/28
    }
    
    track_script {
        check_haproxy
    }
}
```

## CNI Plugin Selection

### Recommended: Canal (Calico + Flannel)

```yaml
# File: kubernetes/cluster-config/cni-canal-config.yaml

cni:
  plugin: canal
  description: "Combination of Calico for policy and Flannel for overlay"
  
  advantages:
    - "Network policy support"
    - "Good performance"
    - "Well tested with Rancher"
    - "Easy troubleshooting"
  
  configuration:
    pod_cidr: "10.42.0.0/16"
    interface_name: "eth0"  # Adjust to your interface
    backend_type: "vxlan"
    mtu: 1450  # Adjust based on your network (1500 - 50 for VXLAN)
```

### Alternative: Cilium (Advanced)

```yaml
# File: kubernetes/cluster-config/cni-cilium-config.yaml

cni:
  plugin: cilium
  description: "eBPF-based networking and security"
  
  advantages:
    - "High performance"
    - "Advanced network policies"
    - "Service mesh capabilities"
    - "Better observability"
  
  configuration:
    pod_cidr: "10.42.0.0/16"
    enable_ipv4: true
    enable_ipv6: false
    tunnel_mode: "vxlan"  # or "geneve"
    
  features:
    hubble: true  # Enable observability
    encryption: false  # Enable for sensitive workloads
```

### CNI Installation Script

```bash
#!/bin/bash
# File: scripts/setup/configure-cni.sh

# This is handled by Rancher/RKE2, but documenting for reference

# For manual configuration (if needed):
# 1. Pod CIDR: 10.42.0.0/16
# 2. Service CIDR: 10.43.0.0/16
# 3. CNI Plugin: canal (default in RKE2)

# Network requirements for CNI
echo "Configuring network requirements for CNI..."

# Load required kernel modules
cat > /etc/modules-load.d/k8s-cni.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Set sysctl parameters
cat > /etc/sysctl.d/k8s-cni.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.forwarding        = 1
net.ipv6.conf.all.forwarding        = 1
EOF

sysctl --system

echo "CNI prerequisites configured successfully"
```

## Network Policies

### Default Deny Policy

```yaml
# File: kubernetes/network-policies/00-default-deny.yaml

# Deny all ingress traffic by default in production namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production  # Apply to each namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Deny all egress traffic by default (optional, more restrictive)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Allow DNS Policy

```yaml
# File: kubernetes/network-policies/01-allow-dns.yaml

# Allow all pods to access DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Application-Specific Policy Example

```yaml
# File: kubernetes/network-policies/app-web-to-db.yaml

# Allow web tier to communicate with database tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - protocol: TCP
      port: 3306  # MySQL
    - protocol: TCP
      port: 5432  # PostgreSQL
```

## DNS Configuration

### Internal DNS Setup

```yaml
# File: infrastructure/dns/internal-dns-zones.yaml

# Internal DNS zones for cluster
dns_zones:
  # Cluster domain
  cluster_domain:
    name: "cluster.local"
    type: "Kubernetes internal"
    nameserver: "10.43.0.10"  # CoreDNS service IP
  
  # Management domain
  management_domain:
    name: "k8s.example.internal"
    type: "Internal infrastructure"
    records:
      # Master nodes
      - name: "master-01.k8s.example.internal"
        type: "A"
        value: "192.168.10.10"
      - name: "master-02.k8s.example.internal"
        type: "A"
        value: "192.168.10.11"
      - name: "master-03.k8s.example.internal"
        type: "A"
        value: "192.168.10.12"
      
      # API VIP
      - name: "api.k8s.example.internal"
        type: "A"
        value: "192.168.1.100"
      
      # Rancher
      - name: "rancher.k8s.example.internal"
        type: "A"
        value: "192.168.1.101"
      
      # Wildcard for ingress
      - name: "*.apps.k8s.example.internal"
        type: "A"
        value: "192.168.1.102"
```

### CoreDNS Customization

```yaml
# File: kubernetes/cluster-config/coredns-custom.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    # Forward internal domain queries to corporate DNS
    example.internal:53 {
        errors
        cache 30
        forward . 192.168.10.2 192.168.10.3
    }
    
    # Custom hosts entries
    hosts {
        192.168.10.20 rancher.k8s.example.internal
        192.168.1.100 api.k8s.example.internal
        fallthrough
    }
```

## Firewall Rules

### Firewalld Rules Script

```bash
#!/bin/bash
# File: scripts/setup/configure-firewall.sh

# Firewall configuration for Kubernetes nodes

configure_master_firewall() {
    echo "Configuring firewall for Master node..."
    
    # Kubernetes API Server
    firewall-cmd --permanent --add-port=6443/tcp
    
    # etcd
    firewall-cmd --permanent --add-port=2379-2380/tcp
    
    # Kubelet API
    firewall-cmd --permanent --add-port=10250/tcp
    
    # kube-scheduler
    firewall-cmd --permanent --add-port=10251/tcp
    
    # kube-controller-manager
    firewall-cmd --permanent --add-port=10252/tcp
    
    # Rancher agent
    firewall-cmd --permanent --add-port=9099/tcp
    
    # NodePort range
    firewall-cmd --permanent --add-port=30000-32767/tcp
    
    # CNI - Flannel VXLAN
    firewall-cmd --permanent --add-port=8472/udp
    firewall-cmd --permanent --add-port=4789/udp
    
    # Allow from specific sources
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" accept'
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.20.0/24" accept'
    
    # Reload firewall
    firewall-cmd --reload
    
    echo "Master node firewall configured successfully"
}

configure_worker_firewall() {
    echo "Configuring firewall for Worker node..."
    
    # Kubelet API
    firewall-cmd --permanent --add-port=10250/tcp
    
    # NodePort range
    firewall-cmd --permanent --add-port=30000-32767/tcp
    
    # Rancher agent
    firewall-cmd --permanent --add-port=9099/tcp
    
    # CNI - Flannel VXLAN
    firewall-cmd --permanent --add-port=8472/udp
    firewall-cmd --permanent --add-port=4789/udp
    
    # Allow from specific sources
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" accept'
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.20.0/24" accept'
    
    # Reload firewall
    firewall-cmd --reload
    
    echo "Worker node firewall configured successfully"
}

# Detect node type and configure accordingly
if [ "$NODE_TYPE" == "master" ]; then
    configure_master_firewall
elif [ "$NODE_TYPE" == "worker" ]; then
    configure_worker_firewall
else
    echo "Error: NODE_TYPE not set. Use: export NODE_TYPE=master or export NODE_TYPE=worker"
    exit 1
fi
```

### IPTables Rules (Alternative to firewalld)

```bash
#!/bin/bash
# File: scripts/setup/configure-iptables.sh

# For systems using iptables directly

# Flush existing rules (be careful in production!)
# iptables -F
# iptables -X

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow from cluster networks
iptables -A INPUT -s 192.168.10.0/24 -j ACCEPT
iptables -A INPUT -s 192.168.20.0/24 -j ACCEPT
iptables -A INPUT -s 10.42.0.0/16 -j ACCEPT
iptables -A INPUT -s 10.43.0.0/16 -j ACCEPT

# Save rules
iptables-save > /etc/sysconfig/iptables
```

## Network Validation Scripts

```bash
#!/bin/bash
# File: scripts/setup/validate-network.sh

# Network validation script

validate_connectivity() {
    local target_ip=$1
    local target_name=$2
    
    echo "Testing connectivity to $target_name ($target_ip)..."
    if ping -c 3 -W 2 $target_ip > /dev/null 2>&1; then
        echo "✓ Connectivity to $target_name successful"
        return 0
    else
        echo "✗ Failed to reach $target_name"
        return 1
    fi
}

validate_port() {
    local target_ip=$1
    local target_port=$2
    local service_name=$3
    
    echo "Testing port $target_port on $service_name..."
    if timeout 5 bash -c "cat < /dev/null > /dev/tcp/$target_ip/$target_port" 2>/dev/null; then
        echo "✓ Port $target_port on $service_name is open"
        return 0
    else
        echo "✗ Port $target_port on $service_name is not accessible"
        return 1
    fi
}

echo "=== Network Validation ==="

# Test connectivity to all master nodes
validate_connectivity 192.168.10.10 "master-01"
validate_connectivity 192.168.10.11 "master-02"
validate_connectivity 192.168.10.12 "master-03"

# Test API server port
validate_port 192.168.1.100 6443 "Kubernetes API"

# Test DNS resolution
echo "Testing DNS resolution..."
if nslookup google.com > /dev/null 2>&1; then
    echo "✓ External DNS resolution works"
else
    echo "✗ External DNS resolution failed"
fi

# Check MTU settings
echo "Checking MTU settings..."
ip link show | grep mtu

echo "=== Validation Complete ==="
```

## Next Steps

After completing network configuration:
1. Verify all firewall rules are applied
2. Test connectivity between all nodes
3. Proceed to [Installation Guide](03-installation-guide.md)
4. Configure [Airgap Installation](04-airgap-installation.md) if required
