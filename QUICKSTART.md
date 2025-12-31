# Quick Start Guide - Production Kubernetes with Rancher

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Linux machines provisioned (RHEL 8+, Ubuntu 20.04+, or Rocky Linux 8+)
- [ ] Minimum 3 master nodes and 3 worker nodes
- [ ] All nodes have network connectivity
- [ ] Root or sudo access on all nodes
- [ ] IP addresses assigned and documented
- [ ] DNS records configured
- [ ] SSL/TLS certificates (optional, can use self-signed)

## Quick Installation (30 minutes)

### Step 1: Prepare All Nodes (10 min)

Run on each node:

```bash
# Download preparation script
curl -sfL https://raw.githubusercontent.com/your-repo/dr-k8s/main/scripts/setup/prepare-node.sh -o prepare-node.sh

# Run preparation
chmod +x prepare-node.sh
sudo ./prepare-node.sh

# Reboot
sudo reboot
```

### Step 2: Setup Load Balancer (5 min)

On two dedicated load balancer nodes:

```bash
# Install HAProxy and Keepalived
sudo yum install -y haproxy keepalived

# Copy configurations
sudo cp infrastructure/loadbalancer/haproxy.cfg /etc/haproxy/
sudo cp infrastructure/loadbalancer/keepalived-master.conf /etc/keepalived/keepalived.conf  # On LB1
sudo cp infrastructure/loadbalancer/keepalived-backup.conf /etc/keepalived/keepalived.conf  # On LB2

# Start services
sudo systemctl enable --now haproxy keepalived
```

### Step 3: Install First Master (5 min)

On the first master node:

```bash
# Run installation script
curl -sfL https://get.rke2.io | sudo sh -

# Create config
sudo mkdir -p /etc/rancher/rke2
sudo cat > /etc/rancher/rke2/config.yaml <<EOF
write-kubeconfig-mode: "0640"
tls-san:
  - "api.k8s.internal"
  - "192.168.1.100"
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
EOF

# Start RKE2
sudo systemctl enable --now rke2-server

# Get node token
sudo cat /var/lib/rancher/rke2/server/node-token
```

### Step 4: Join Additional Masters (3 min each)

On each additional master node:

```bash
sudo mkdir -p /etc/rancher/rke2
sudo cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://192.168.10.10:9345
token: <TOKEN_FROM_STEP_3>
tls-san:
  - "api.k8s.internal"
  - "192.168.1.100"
EOF

curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable --now rke2-server
```

### Step 5: Join Worker Nodes (3 min each)

On each worker node:

```bash
sudo mkdir -p /etc/rancher/rke2
sudo cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://192.168.1.100:9345
token: <TOKEN_FROM_STEP_3>
EOF

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh -
sudo systemctl enable --now rke2-agent
```

### Step 6: Install Rancher (5 min)

On the first master node:

```bash
# Setup kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Install Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.k8s.internal \
  --set bootstrapPassword=admin
```

## Access Your Cluster

### Via Rancher UI
```
URL: https://rancher.k8s.internal
Username: admin
Password: admin (change on first login)
```

### Via kubectl
```bash
# Copy kubeconfig from master
scp master-01:/etc/rancher/rke2/rke2.yaml ~/.kube/config

# Edit server address
sed -i 's/127.0.0.1/192.168.1.100/g' ~/.kube/config

# Test
kubectl get nodes
```

## Next Steps

1. **Install Storage**: [Longhorn](docs/03-installation-guide.md#install-longhorn-storage)
2. **Setup Monitoring**: [Monitoring Guide](docs/06-monitoring-observability.md)
3. **Configure Backups**: [Backup & Restore](docs/07-backup-restore.md)
4. **Plan DR**: [Disaster Recovery](docs/05-disaster-recovery.md)
5. **Deploy Apps**: Use Helm charts from `helm-charts/applications/`

## Troubleshooting

### Cluster not forming
```bash
# Check RKE2 logs
sudo journalctl -u rke2-server -f

# Check connectivity
ping 192.168.1.100
telnet 192.168.10.10 6443
```

### Nodes not joining
```bash
# Verify token
sudo cat /var/lib/rancher/rke2/server/node-token

# Check firewall
sudo firewall-cmd --list-all
```

### Rancher not accessible
```bash
# Check pods
kubectl get pods -n cattle-system

# Check ingress
kubectl get ingress -A
```

## Support

For detailed documentation, see:
- [Full Installation Guide](docs/03-installation-guide.md)
- [Infrastructure Requirements](docs/01-infrastructure-requirements.md)
- [Network Design](docs/02-network-design.md)
