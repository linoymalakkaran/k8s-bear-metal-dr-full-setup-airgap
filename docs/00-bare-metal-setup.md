# Bare Metal Server Setup Guide

## Table of Contents
1. [Hardware Preparation](#hardware-preparation)
2. [BIOS/UEFI Configuration](#biosuefi-configuration)
3. [Operating System Installation](#operating-system-installation)
4. [Disk Partitioning Strategy](#disk-partitioning-strategy)
5. [Network Configuration](#network-configuration)
6. [Post-Install OS Configuration](#post-install-os-configuration)
7. [Validation](#validation)

## Hardware Preparation

### Physical Server Checklist

```yaml
bare_metal_checklist:
  hardware_verification:
    - [ ] Servers physically racked and powered
    - [ ] All power supplies connected (redundant PSUs)
    - [ ] Network cables connected to all NICs
    - [ ] IPMI/iLO/iDRAC configured for remote management
    - [ ] RAID controllers configured (if applicable)
    - [ ] All drives detected in BIOS
    - [ ] Memory modules installed correctly
    - [ ] CPU cooling verified
  
  network_connectivity:
    - [ ] Management network cabled
    - [ ] Data network cabled (10Gbps recommended)
    - [ ] Storage network cabled (if separate)
    - [ ] Network switches configured
    - [ ] VLANs created on switches
  
  remote_management:
    - [ ] IPMI/iLO accessible via network
    - [ ] Remote console access tested
    - [ ] Virtual media support verified
```

### Server Inventory Template

```bash
# File: infrastructure/bare-metal/server-inventory.csv

Hostname,Role,SerialNumber,IPMIAddress,ManagementIP,DataIP,StorageIP,MACAddress,CPUs,RAM,Disks
master-01,master,SN001,192.168.0.10,192.168.10.10,,,AA:BB:CC:DD:01:01,8,32GB,2x200GB SSD
master-02,master,SN002,192.168.0.11,192.168.10.11,,,AA:BB:CC:DD:01:02,8,32GB,2x200GB SSD
master-03,master,SN003,192.168.0.12,192.168.10.12,,,AA:BB:CC:DD:01:03,8,32GB,2x200GB SSD
worker-01,worker,SN004,192.168.0.20,192.168.20.10,,,AA:BB:CC:DD:02:01,16,64GB,2x500GB SSD
worker-02,worker,SN005,192.168.0.21,192.168.20.11,,,AA:BB:CC:DD:02:02,16,64GB,2x500GB SSD
worker-03,worker,SN006,192.168.0.22,192.168.20.12,,,AA:BB:CC:DD:02:03,16,64GB,2x500GB SSD
```

## BIOS/UEFI Configuration

### Accessing BIOS/UEFI

```bash
# Most common methods:
# - Dell: Press F2 during boot
# - HP: Press F9 during boot
# - Supermicro: Press Delete during boot
# - Lenovo: Press F1 during boot

# Or use IPMI for remote access:
# Dell iDRAC example
ipmitool -I lanplus -H 192.168.0.10 -U root -P password chassis bootdev bios
ipmitool -I lanplus -H 192.168.0.10 -U root -P password chassis power reset
```

### Required BIOS Settings

```yaml
bios_configuration:
  boot_settings:
    boot_mode: "UEFI"  # Modern systems use UEFI
    secure_boot: "Disabled"  # Required for most Linux distributions
    boot_order:
      1: "Hard Drive"
      2: "Network (PXE)" # For automated deployments
      3: "USB/Virtual Media"
    
  cpu_settings:
    virtualization_technology: "Enabled"  # Intel VT-x or AMD-V
    hyper_threading: "Enabled"
    turbo_boost: "Enabled"
    c_states: "Disabled"  # For consistent performance
    
  memory_settings:
    memory_mode: "Performance Mode"
    node_interleaving: "Disabled"  # For NUMA awareness
    
  power_management:
    power_profile: "Maximum Performance"
    c1e_support: "Disabled"
    
  storage_controller:
    raid_mode: "RAID" or "HBA/JBOD"  # Depends on your storage strategy
    write_cache: "Enabled with BBU"  # If RAID controller
    
  network_settings:
    pxe_boot: "Enabled" # On at least one NIC
    wake_on_lan: "Enabled"
```

### BIOS Configuration Script (IPMI)

```bash
#!/bin/bash
# File: scripts/setup/configure-bios-ipmi.sh

# Configure BIOS settings via IPMI (Dell iDRAC example)

SERVER_IPMI=$1
IPMI_USER="root"
IPMI_PASS="password"

if [ -z "$SERVER_IPMI" ]; then
    echo "Usage: $0 <ipmi-address>"
    exit 1
fi

echo "Configuring BIOS for $SERVER_IPMI..."

# Set boot mode to UEFI
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS set BIOS.BiosBootSettings.BootMode Uefi

# Disable Secure Boot
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS set BIOS.SecureBoot.SecureBootMode Disabled

# Enable Virtualization
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS set BIOS.ProcSettings.ProcVirtualization Enabled

# Set performance profile
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS set BIOS.SysProfileSettings.SysProfile PerfPerWattOptimizedDasr

# Create BIOS config job and reboot
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS jobqueue create BIOS.Setup.1-1
racadm -r $SERVER_IPMI -u $IPMI_USER -p $IPMI_PASS serveraction powercycle

echo "BIOS configuration job created. Server will reboot."
```

## Operating System Installation

### Option 1: Manual Installation (Rocky Linux 8)

#### Step 1: Download ISO

```bash
# On your workstation or jumpbox

# Rocky Linux 8.8 (RHEL clone)
wget https://download.rockylinux.org/pub/rocky/8/isos/x86_64/Rocky-8.8-x86_64-minimal.iso

# Verify checksum
wget https://download.rockylinux.org/pub/rocky/8/isos/x86_64/CHECKSUM
sha256sum -c CHECKSUM --ignore-missing

# For Ubuntu 22.04 LTS
# wget https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso
```

#### Step 2: Mount ISO via IPMI

```bash
#!/bin/bash
# File: scripts/setup/mount-iso-ipmi.sh

# Mount ISO via IPMI virtual media (Dell iDRAC example)

SERVER_IPMI=$1
ISO_PATH=$2  # Can be HTTP/NFS path or local

# Dell iDRAC
racadm -r $SERVER_IPMI -u root -p password remoteimage -c
racadm -r $SERVER_IPMI -u root -p password remoteimage -t cdimage -l $ISO_PATH

# HP iLO
# hponcfg -i mount_iso.xml

# Supermicro IPMI
# ipmitool -I lanplus -H $SERVER_IPMI -U ADMIN -P ADMIN sol activate

echo "ISO mounted. Boot the server from virtual media."
```

#### Step 3: Installation Parameters

```yaml
# File: infrastructure/bare-metal/kickstart-template.cfg
# Kickstart file for automated Rocky Linux installation

# Use network installation
url --url="http://download.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/"

# Language and keyboard
lang en_US.UTF-8
keyboard us

# Network configuration (adjust for each server)
network --bootproto=static --ip=192.168.10.10 --netmask=255.255.255.0 --gateway=192.168.10.1 --nameserver=8.8.8.8,8.8.4.4 --hostname=master-01.k8s.internal --device=eno1 --onboot=yes
network --bootproto=static --ip=192.168.30.10 --netmask=255.255.255.0 --device=eno2 --onboot=yes --nodefroute

# Root password (change this!)
rootpw --iscrypted $6$rounds=4096$saltsaltsa$hashhashhash

# Create admin user
user --name=admin --groups=wheel --password=$6$rounds=4096$saltsalt$hash --iscrypted

# Timezone
timezone America/New_York --utc

# Bootloader
bootloader --location=mbr --boot-drive=sda

# Clear and partition disks
clearpart --all --initlabel --drives=sda,sdb
ignoredisk --only-use=sda,sdb

# Disk partitioning (see detailed section below)
part /boot/efi --fstype=efi --size=512 --ondisk=sda
part /boot --fstype=xfs --size=1024 --ondisk=sda
part pv.01 --size=1 --grow --ondisk=sda

# LVM configuration
volgroup vg_system pv.01
logvol / --fstype=xfs --name=lv_root --vgname=vg_system --size=50000
logvol /var --fstype=xfs --name=lv_var --vgname=vg_system --size=100000
logvol swap --fstype=swap --name=lv_swap --vgname=vg_system --size=16384

# Second disk for etcd/data (master nodes) or container storage (worker nodes)
part pv.02 --size=1 --grow --ondisk=sdb
volgroup vg_data pv.02
logvol /var/lib/rancher --fstype=xfs --name=lv_rancher --vgname=vg_data --size=50000
logvol /var/lib/etcd --fstype=xfs --name=lv_etcd --vgname=vg_data --size=100000  # Master only

# Install minimal packages
%packages
@^minimal-environment
@standard
vim
wget
curl
net-tools
bind-utils
%end

# Post-installation script
%post --log=/root/kickstart-post.log
#!/bin/bash

# Update system
yum update -y

# Disable SELinux (will be set to permissive later for K8s)
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Disable firewall (will configure later)
systemctl disable firewalld

# Enable SSH
systemctl enable sshd

# Set up SSH key for admin user
mkdir -p /home/admin/.ssh
cat >> /home/admin/.ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ... your-public-key-here
EOF
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
chown -R admin:admin /home/admin/.ssh

%end

# Reboot after installation
reboot
```

### Option 2: Automated PXE Installation

```bash
#!/bin/bash
# File: scripts/setup/setup-pxe-server.sh

# Setup PXE server for automated installations

TFTP_ROOT="/var/lib/tftpboot"
HTTP_ROOT="/var/www/html"

echo "Setting up PXE boot server..."

# Install required packages
yum install -y dhcp-server tftp-server httpd syslinux xinetd

# Configure DHCP
cat > /etc/dhcp/dhcpd.conf <<EOF
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    next-server 192.168.10.5;  # PXE server IP
    filename "pxelinux.0";
}

# Static entries for each server
host master-01 {
    hardware ethernet AA:BB:CC:DD:01:01;
    fixed-address 192.168.10.10;
    filename "pxelinux.0";
}
EOF

# Configure TFTP
cat > /etc/xinetd.d/tftp <<EOF
service tftp
{
    socket_type = dgram
    protocol = udp
    wait = yes
    user = root
    server = /usr/sbin/in.tftpd
    server_args = -s $TFTP_ROOT
    disable = no
    per_source = 11
    cps = 100 2
    flags = IPv4
}
EOF

# Setup PXE boot files
mkdir -p $TFTP_ROOT/pxelinux.cfg
cp /usr/share/syslinux/pxelinux.0 $TFTP_ROOT/
cp /usr/share/syslinux/menu.c32 $TFTP_ROOT/
cp /usr/share/syslinux/memdisk $TFTP_ROOT/
cp /usr/share/syslinux/mboot.c32 $TFTP_ROOT/
cp /usr/share/syslinux/chain.c32 $TFTP_ROOT/

# Download Rocky Linux
mkdir -p $HTTP_ROOT/rocky8
wget -r -np -nH --cut-dirs=5 -P $HTTP_ROOT/rocky8/ \
  http://download.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/

# PXE menu configuration
cat > $TFTP_ROOT/pxelinux.cfg/default <<EOF
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300

MENU TITLE K8s Cluster Installation

LABEL rocky8-auto
    MENU LABEL Rocky Linux 8 - Automated Install
    KERNEL rocky8/vmlinuz
    APPEND initrd=rocky8/initrd.img inst.ks=http://192.168.10.5/kickstart/master.cfg
EOF

# Create kickstart files directory
mkdir -p $HTTP_ROOT/kickstart
# Copy your kickstart files here

# Start services
systemctl enable --now dhcpd
systemctl enable --now xinetd
systemctl enable --now httpd

echo "PXE server setup complete"
```

## Disk Partitioning Strategy

### Master Nodes Partitioning

```yaml
master_node_partitioning:
  disk_1_sda_200gb:  # OS disk
    partitions:
      - mount: "/boot/efi"
        size: "512 MB"
        filesystem: "vfat"
        flags: ["boot", "esp"]
      
      - mount: "/boot"
        size: "1 GB"
        filesystem: "xfs"
      
      - type: "LVM PV"
        size: "remaining"
        volume_group: "vg_system"
        logical_volumes:
          - name: "lv_root"
            mount: "/"
            size: "50 GB"
            filesystem: "xfs"
          
          - name: "lv_var"
            mount: "/var"
            size: "100 GB"
            filesystem: "xfs"
          
          - name: "lv_swap"
            mount: "swap"
            size: "16 GB"
            filesystem: "swap"
          
          - name: "lv_tmp"
            mount: "/tmp"
            size: "10 GB"
            filesystem: "xfs"
            options: "noexec,nosuid,nodev"
  
  disk_2_sdb_200gb:  # etcd and K8s data
    partitions:
      - type: "LVM PV"
        size: "all"
        volume_group: "vg_data"
        logical_volumes:
          - name: "lv_etcd"
            mount: "/var/lib/etcd"
            size: "100 GB"
            filesystem: "xfs"
            notes: "Dedicated disk for etcd performance"
          
          - name: "lv_rancher"
            mount: "/var/lib/rancher"
            size: "80 GB"
            filesystem: "xfs"
```

### Worker Nodes Partitioning

```yaml
worker_node_partitioning:
  disk_1_sda_200gb:  # OS disk
    # Same as master node disk 1
  
  disk_2_sdb_500gb:  # Container storage and data
    partitions:
      - type: "LVM PV"
        size: "all"
        volume_group: "vg_data"
        logical_volumes:
          - name: "lv_rancher"
            mount: "/var/lib/rancher"
            size: "100 GB"
            filesystem: "xfs"
          
          - name: "lv_containers"
            mount: "/var/lib/containers"
            size: "100 GB"
            filesystem: "xfs"
          
          - name: "lv_longhorn"
            mount: "/var/lib/longhorn"
            size: "remaining (~280 GB)"
            filesystem: "xfs"
            notes: "Used by Longhorn for persistent storage"
```

### Manual Partitioning Commands

```bash
#!/bin/bash
# File: scripts/setup/partition-disks.sh

# Manual disk partitioning script (if not using Kickstart)

set -e

echo "WARNING: This will destroy all data on /dev/sda and /dev/sdb!"
read -p "Continue? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then exit 1; fi

# Partition disk 1 (OS)
parted /dev/sda --script mklabel gpt
parted /dev/sda --script mkpart primary fat32 1MiB 513MiB
parted /dev/sda --script set 1 esp on
parted /dev/sda --script mkpart primary xfs 513MiB 1537MiB
parted /dev/sda --script mkpart primary 1537MiB 100%
parted /dev/sda --script set 3 lvm on

# Create filesystems
mkfs.vfat -F32 /dev/sda1
mkfs.xfs -f /dev/sda2

# Create LVM
pvcreate /dev/sda3
vgcreate vg_system /dev/sda3
lvcreate -L 50G -n lv_root vg_system
lvcreate -L 100G -n lv_var vg_system
lvcreate -L 16G -n lv_swap vg_system
lvcreate -L 10G -n lv_tmp vg_system

# Format LVM volumes
mkfs.xfs /dev/vg_system/lv_root
mkfs.xfs /dev/vg_system/lv_var
mkswap /dev/vg_system/lv_swap
mkfs.xfs /dev/vg_system/lv_tmp

# Partition disk 2 (data)
parted /dev/sdb --script mklabel gpt
parted /dev/sdb --script mkpart primary 1MiB 100%
parted /dev/sdb --script set 1 lvm on

pvcreate /dev/sdb1
vgcreate vg_data /dev/sdb1

# For master nodes
if [ "$NODE_TYPE" == "master" ]; then
    lvcreate -L 100G -n lv_etcd vg_data
    lvcreate -L 80G -n lv_rancher vg_data
    mkfs.xfs /dev/vg_data/lv_etcd
    mkfs.xfs /dev/vg_data/lv_rancher
fi

# For worker nodes
if [ "$NODE_TYPE" == "worker" ]; then
    lvcreate -L 100G -n lv_rancher vg_data
    lvcreate -L 100G -n lv_containers vg_data
    lvcreate -l 100%FREE -n lv_longhorn vg_data
    mkfs.xfs /dev/vg_data/lv_rancher
    mkfs.xfs /dev/vg_data/lv_containers
    mkfs.xfs /dev/vg_data/lv_longhorn
fi

echo "Disk partitioning complete"
```

### /etc/fstab Configuration

```bash
# File: /etc/fstab (Master Node Example)

# OS Disk
UUID=xxxx-xxxx    /boot/efi    vfat    defaults    0 2
UUID=xxxx-xxxx    /boot        xfs     defaults    0 2

# LVM volumes
/dev/mapper/vg_system-lv_root    /              xfs    defaults        0 1
/dev/mapper/vg_system-lv_var     /var           xfs    defaults        0 2
/dev/mapper/vg_system-lv_tmp     /tmp           xfs    noexec,nosuid,nodev  0 2
/dev/mapper/vg_system-lv_swap    swap           swap   defaults        0 0

# Data disk (etcd on dedicated disk)
/dev/mapper/vg_data-lv_etcd      /var/lib/etcd      xfs    defaults,noatime    0 2
/dev/mapper/vg_data-lv_rancher   /var/lib/rancher   xfs    defaults            0 2
```

## Network Configuration

### Network Interface Configuration

#### Using NetworkManager (Rocky/RHEL 8)

```bash
#!/bin/bash
# File: scripts/setup/configure-network.sh

# Configure network interfaces using nmcli

NODE_HOSTNAME="master-01"
MGMT_IP="192.168.10.10"
MGMT_GW="192.168.10.1"
STORAGE_IP="192.168.30.10"

# Set hostname
hostnamectl set-hostname ${NODE_HOSTNAME}.k8s.internal

# Configure management interface (eno1)
nmcli connection modify eno1 \
    ipv4.method manual \
    ipv4.addresses ${MGMT_IP}/24 \
    ipv4.gateway ${MGMT_GW} \
    ipv4.dns "8.8.8.8 8.8.4.4" \
    ipv4.dns-search "k8s.internal" \
    connection.autoconnect yes

# Configure storage interface (eno2) - no default route
nmcli connection modify eno2 \
    ipv4.method manual \
    ipv4.addresses ${STORAGE_IP}/24 \
    ipv4.never-default yes \
    connection.autoconnect yes

# Apply changes
nmcli connection up eno1
nmcli connection up eno2

# Verify
nmcli device status
ip addr show
ip route show
```

#### Static Network Config Files (Alternative)

```bash
# File: /etc/sysconfig/network-scripts/ifcfg-eno1 (Rocky/RHEL)

TYPE=Ethernet
BOOTPROTO=none
NAME=eno1
DEVICE=eno1
ONBOOT=yes
IPADDR=192.168.10.10
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
DNS1=8.8.8.8
DNS2=8.8.4.4
DOMAIN=k8s.internal
```

```bash
# File: /etc/sysconfig/network-scripts/ifcfg-eno2 (Storage Network)

TYPE=Ethernet
BOOTPROTO=none
NAME=eno2
DEVICE=eno2
ONBOOT=yes
IPADDR=192.168.30.10
NETMASK=255.255.255.0
DEFROUTE=no  # Don't use as default route
```

#### Ubuntu/Netplan Configuration

```yaml
# File: /etc/netplan/01-network-config.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      addresses:
        - 192.168.10.10/24
      gateway4: 192.168.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - k8s.internal
      dhcp4: no
    
    eno2:
      addresses:
        - 192.168.30.10/24
      dhcp4: no
      dhcp4-overrides:
        use-routes: false  # Don't use as default route
```

```bash
# Apply netplan configuration
netplan apply
```

### DNS and Hostname Configuration

```bash
# File: /etc/hosts (on all nodes)

127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain

# Cluster nodes
192.168.10.10   master-01.k8s.internal master-01
192.168.10.11   master-02.k8s.internal master-02
192.168.10.12   master-03.k8s.internal master-03

192.168.20.10   worker-01.k8s.internal worker-01
192.168.20.11   worker-02.k8s.internal worker-02
192.168.20.12   worker-03.k8s.internal worker-03

# Load balancers
192.168.1.10    lb-01.k8s.internal lb-01
192.168.1.11    lb-02.k8s.internal lb-02
192.168.1.100   api.k8s.internal

# Infrastructure
192.168.10.20   rancher.k8s.internal rancher
```

## Post-Install OS Configuration

### System Hardening

```bash
#!/bin/bash
# File: scripts/setup/harden-os.sh

# System hardening for Kubernetes nodes

echo "Hardening OS for Kubernetes..."

# 1. Update all packages
yum update -y

# 2. Install security updates automatically (optional)
yum install -y yum-cron
systemctl enable --now yum-cron

# 3. Configure SSH hardening
cat >> /etc/ssh/sshd_config <<EOF
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
MaxSessions 10
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

systemctl restart sshd

# 4. Configure firewall (will be customized per node type later)
systemctl enable firewalld
systemctl start firewalld

# 5. Disable unnecessary services
systemctl disable postfix
systemctl stop postfix

# 6. Configure auditd
yum install -y audit
systemctl enable --now auditd

# 7. Set timezone
timedatectl set-timezone America/New_York

# 8. Configure NTP
yum install -y chrony
cat > /etc/chrony.conf <<EOF
server time.google.com iburst
server time.cloudflare.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
EOF

systemctl enable --now chronyd

# 9. Configure log rotation
cat > /etc/logrotate.d/kubernetes <<EOF
/var/log/containers/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
EOF

echo "OS hardening complete"
```

## Validation

### Pre-Kubernetes Validation Script

```bash
#!/bin/bash
# File: scripts/setup/validate-bare-metal.sh

# Comprehensive validation before K8s installation

echo "=== Bare Metal Server Validation ==="

ERRORS=0

# Check hostname
echo -n "Checking hostname... "
HOSTNAME=$(hostname -f)
if [[ $HOSTNAME == *.k8s.internal ]]; then
    echo "✓ $HOSTNAME"
else
    echo "✗ Hostname not properly set"
    ((ERRORS++))
fi

# Check network interfaces
echo -n "Checking network interfaces... "
if ip addr show eno1 | grep -q "192.168.10"; then
    echo "✓ Management network configured"
else
    echo "✗ Management network not configured"
    ((ERRORS++))
fi

# Check disk space
echo "Checking disk space..."
for mount in / /var /var/lib/rancher; do
    AVAIL=$(df -h $mount 2>/dev/null | awk 'NR==2 {print $4}')
    echo "  $mount: $AVAIL available"
done

# Check memory
echo -n "Checking memory... "
MEM_GB=$(free -g | awk '/^Mem:/ {print $2}')
if [ $MEM_GB -ge 16 ]; then
    echo "✓ ${MEM_GB}GB RAM"
else
    echo "✗ Insufficient RAM: ${MEM_GB}GB (minimum 16GB)"
    ((ERRORS++))
fi

# Check CPU
echo -n "Checking CPU... "
CPU_CORES=$(nproc)
if [ $CPU_CORES -ge 4 ]; then
    echo "✓ ${CPU_CORES} cores"
else
    echo "✗ Insufficient CPU: ${CPU_CORES} cores (minimum 4)"
    ((ERRORS++))
fi

# Check time sync
echo -n "Checking time synchronization... "
if timedatectl | grep -q "synchronized: yes"; then
    echo "✓ Time synchronized"
else
    echo "✗ Time not synchronized"
    ((ERRORS++))
fi

# Check swap
echo -n "Checking swap... "
if [ $(free | grep Swap | awk '{print $2}') -eq 0 ]; then
    echo "✓ Swap disabled"
else
    echo "⚠ Swap is enabled (will be disabled during K8s install)"
fi

# Check SELinux
echo -n "Checking SELinux... "
SELINUX=$(getenforce 2>/dev/null || echo "Not installed")
echo "$SELINUX"

# Check connectivity
echo "Checking connectivity..."
for host in master-01 master-02 master-03; do
    if ping -c 1 -W 2 $host &>/dev/null; then
        echo "  ✓ $host reachable"
    else
        echo "  ✗ $host not reachable"
        ((ERRORS++))
    fi
done

# Check DNS
echo -n "Checking DNS resolution... "
if nslookup google.com &>/dev/null; then
    echo "✓ DNS working"
else
    echo "✗ DNS not working"
    ((ERRORS++))
fi

# Summary
echo ""
echo "=== Validation Summary ==="
if [ $ERRORS -eq 0 ]; then
    echo "✓ All checks passed! Server ready for Kubernetes installation."
    exit 0
else
    echo "✗ Found $ERRORS error(s). Please fix before proceeding."
    exit 1
fi
```

## Next Steps

After bare metal setup is complete:

1. ✅ Verify all validations pass
2. ➡️ Proceed to [Installation Guide](03-installation-guide.md)
3. 📝 Document all server details in inventory
4. 💾 Take snapshots/backups before installing Kubernetes

## Troubleshooting

### Common Issues

```bash
# Network interface not coming up
nmcli connection reload
nmcli connection up eno1

# Check network interface names
ip link show

# IPMI not accessible
ipmitool -I lanplus -H 192.168.0.10 -U root -P password chassis status

# Disk not detected
lsblk
cat /proc/partitions

# Time sync issues
chronyc sources
chronyc tracking

# Hostname resolution
getent hosts master-01
```
