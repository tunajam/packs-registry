# Proxmox VE Essentials

Proxmox Virtual Environment for homelab virtualization.

## When to Use

- Running multiple VMs and containers on one server
- Need KVM virtualization with web interface
- Want LXC containers for lightweight services
- Building a clustered homelab

## Post-Install Setup

### Remove Subscription Nag

```bash
# Edit the JavaScript file
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" \
    /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

### Enable No-Subscription Repository

```bash
# Disable enterprise repo
mv /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources.list.d/pve-enterprise.list.disabled

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > \
    /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

### Enable IOMMU (GPU Passthrough)

Edit `/etc/default/grub`:
```bash
# Intel
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# AMD
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

```bash
update-grub
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules
update-initramfs -u -k all
reboot
```

## LXC Containers

### Create Container

```bash
# Download template
pveam update
pveam available | grep ubuntu
pveam download local ubuntu-24.04-standard_24.04-1_amd64.tar.zst

# Create container
pct create 100 local:vztmpl/ubuntu-24.04-standard_24.04-1_amd64.tar.zst \
    --hostname mycontainer \
    --storage local-lvm \
    --rootfs local-lvm:8 \
    --memory 2048 \
    --cores 2 \
    --net0 name=eth0,bridge=vmbr0,ip=dhcp \
    --password \
    --unprivileged 1 \
    --features nesting=1

pct start 100
```

### Docker in LXC

```bash
# Enable features (in Proxmox shell)
pct set 100 --features nesting=1,keyctl=1

# Inside container
apt update && apt install -y docker.io docker-compose
systemctl enable --now docker
```

### GPU Passthrough to LXC

Edit `/etc/pve/lxc/100.conf`:
```
# Intel iGPU
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir

# For Jellyfin transcoding
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

## Virtual Machines

### Create VM from ISO

```bash
# Upload ISO to storage first, then:
qm create 200 \
    --name ubuntu-server \
    --memory 4096 \
    --cores 4 \
    --sockets 1 \
    --cpu host \
    --net0 virtio,bridge=vmbr0 \
    --scsihw virtio-scsi-pci \
    --scsi0 local-lvm:32 \
    --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
    --boot order=ide2;scsi0 \
    --ostype l26

qm start 200
```

### Cloud-Init Template

```bash
# Download cloud image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create VM
qm create 9000 --memory 2048 --cores 2 --name ubuntu-cloud --net0 virtio,bridge=vmbr0

# Import disk
qm importdisk 9000 noble-server-cloudimg-amd64.img local-lvm

# Configure VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Set cloud-init options
qm set 9000 --ciuser admin
qm set 9000 --cipassword yourpassword
qm set 9000 --sshkeys ~/.ssh/id_rsa.pub
qm set 9000 --ipconfig0 ip=dhcp

# Convert to template
qm template 9000
```

### Clone from Template

```bash
qm clone 9000 201 --name myvm --full
qm set 201 --memory 4096 --cores 4
qm set 201 --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1
qm start 201
```

## Storage

### Add NFS Storage

Datacenter > Storage > Add > NFS:
- ID: nas-media
- Server: 192.168.1.10
- Export: /mnt/pool/media
- Content: ISO image, Container template, VZDump backup file

### Add ZFS Pool

```bash
# List disks
lsblk

# Create mirror pool
zpool create -f tank mirror /dev/sda /dev/sdb

# Add to Proxmox
pvesm add zfspool tank --pool tank --content images,rootdir
```

### Thin Provisioning (LVM-Thin)

```bash
# Create thin pool
lvcreate -L 500G -T pve/data

# Add to storage config
pvesm add lvmthin data --thinpool data --vgname pve --content images,rootdir
```

## Backup

### Scheduled Backups

Datacenter > Backup > Add:
- Node: All
- Storage: your-backup-storage
- Schedule: Weekly
- Mode: Snapshot
- Compression: ZSTD

### Backup Commands

```bash
# Backup single VM
vzdump 200 --storage backup-storage --mode snapshot --compress zstd

# Backup all VMs
vzdump --all --storage backup-storage --mode snapshot

# Restore VM
qmrestore /var/lib/vz/dump/vzdump-qemu-200-2024_01_15.vma.zst 201

# Restore LXC
pct restore 101 /var/lib/vz/dump/vzdump-lxc-100-2024_01_15.tar.zst
```

## Networking

### Create VLAN-Aware Bridge

Edit `/etc/network/interfaces`:
```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.5/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

### Assign VLAN to VM

```bash
qm set 200 --net0 virtio,bridge=vmbr0,tag=100
```

## Clustering

### Create Cluster

```bash
# On first node
pvecm create my-cluster

# On additional nodes
pvecm add 192.168.1.5  # IP of first node
```

### Check Cluster Status

```bash
pvecm status
pvecm nodes
```

## Monitoring

### Enable InfluxDB Metrics

Datacenter > Metric Server > Add > InfluxDB:
- Server: influxdb.local
- Port: 8086
- Protocol: HTTP
- Organization: proxmox
- Bucket: proxmox
- Token: your-token

## Useful Commands

```bash
# List all VMs
qm list

# List all containers
pct list

# VM info
qm config 200

# Container info
pct config 100

# Live migration
qm migrate 200 other-node --online

# Resize disk
qm resize 200 scsi0 +50G

# Snapshot
qm snapshot 200 pre-upgrade --description "Before upgrade"
qm rollback 200 pre-upgrade
```

## Best Practices

1. **Use LXC for services** - lower overhead than VMs
2. **Cloud-init templates** - fast VM provisioning
3. **Separate storage pools** - ISOs, VMs, backups
4. **Regular backups** - automated, offsite
5. **Enable IOMMU early** - GPU/PCIe passthrough
6. **Use VLANs** - network segmentation
7. **Monitor resources** - InfluxDB + Grafana
