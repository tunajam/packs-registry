# Network Segmentation

VLANs and firewall rules to secure your homelab.

## When to Use

- Isolate IoT devices from main network
- Separate guest network
- Protect sensitive servers
- Limit blast radius of compromises

## VLAN Design

### Typical Homelab VLANs

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 1 | Management | 192.168.1.0/24 | Network gear, Proxmox |
| 10 | Trusted | 192.168.10.0/24 | Personal devices |
| 20 | Servers | 192.168.20.0/24 | Homelab services |
| 30 | IoT | 192.168.30.0/24 | Smart home devices |
| 40 | Guest | 192.168.40.0/24 | Guest WiFi |
| 50 | Security | 192.168.50.0/24 | Cameras, NVR |

### Traffic Matrix

| Source → Dest | Trusted | Servers | IoT | Guest |
|---------------|---------|---------|-----|-------|
| Trusted | ✓ | ✓ | ✓ | ✗ |
| Servers | Response | ✓ | ✓ | ✗ |
| IoT | ✗ | Limited | ✓ | ✗ |
| Guest | ✗ | ✗ | ✗ | Internet |

## OPNsense/pfSense Setup

### Create VLANs

Interfaces → Other Types → VLAN:
- Parent: igb0 (LAN interface)
- VLAN tag: 10
- Description: Trusted

### Assign Interface

Interfaces → Assignments:
- Add VLAN to available network ports
- Enable interface
- Set static IP (192.168.10.1/24)

### DHCP Server

Services → DHCPv4 → [VLAN Interface]:
- Enable: Yes
- Range: 192.168.10.100 - 192.168.10.200
- DNS: 192.168.1.10 (Pi-hole)
- Gateway: 192.168.10.1

## Firewall Rules

### IoT VLAN Rules (Restrictive)

```
# Allow DNS to Pi-hole
Pass | IoT net | * | Pi-hole | 53 | DNS

# Allow MQTT to Home Assistant
Pass | IoT net | * | Home Assistant | 1883 | MQTT

# Allow established/related
Pass | IoT net | * | * | * | State: established/related

# Block inter-VLAN
Block | IoT net | * | RFC1918 | * | Block private

# Allow internet
Pass | IoT net | * | * | * | Internet
```

### OPNsense Rule Translation

Firewall → Rules → IoT:

1. **Allow DNS**
   - Action: Pass
   - Source: IoT net
   - Destination: Single host (192.168.1.10)
   - Port: 53

2. **Allow Home Assistant**
   - Action: Pass
   - Source: IoT net
   - Destination: Single host (192.168.20.50)
   - Port: 1883, 8123

3. **Block RFC1918**
   - Action: Block
   - Source: IoT net
   - Destination: RFC1918 networks
   
4. **Allow Internet**
   - Action: Pass
   - Source: IoT net
   - Destination: any

### Guest Network Rules

```
# Block all private networks
Block | Guest net | * | RFC1918 | *

# Allow internet
Pass | Guest net | * | !RFC1918 | *
```

## Managed Switch Configuration

### UniFi Example

1. Settings → Networks → Create New
2. Name: IoT
3. VLAN ID: 30
4. Gateway: 192.168.30.1
5. DHCP: Disable (handled by router)

### Port Profiles

- **All VLANs (Trunk)**: For router, APs, hypervisors
- **Native VLAN 10 (Access)**: For trusted devices
- **Tagged VLAN 30 (Access)**: For IoT-only ports

## WiFi VLANs

### UniFi SSIDs

| SSID | VLAN | Notes |
|------|------|-------|
| Home-WiFi | 10 | WPA3, trusted devices |
| IoT-WiFi | 30 | WPA2, hidden |
| Guest-WiFi | 40 | Captive portal |

### Assign VLAN to SSID

Settings → WiFi → Create:
- Name: IoT-WiFi
- Network: IoT (VLAN 30)
- Security: WPA2
- Hide SSID: Yes

## Docker Network Isolation

### Create Isolated Network

```yaml
networks:
  internal:
    driver: bridge
    internal: true  # No internet access
  
  frontend:
    driver: bridge

services:
  db:
    networks:
      - internal
  
  app:
    networks:
      - internal
      - frontend
  
  proxy:
    networks:
      - frontend
    ports:
      - "80:80"
```

### Macvlan (Direct VLAN Access)

```yaml
networks:
  iot:
    driver: macvlan
    driver_opts:
      parent: eth0.30  # VLAN 30
    ipam:
      config:
        - subnet: 192.168.30.0/24
          gateway: 192.168.30.1
          ip_range: 192.168.30.200/29

services:
  homeassistant:
    networks:
      iot:
        ipv4_address: 192.168.30.200
```

## Proxmox VLANs

### VLAN-Aware Bridge

`/etc/network/interfaces`:
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

### VM VLAN Assignment

```bash
qm set 100 --net0 virtio,bridge=vmbr0,tag=20
```

Or in UI: Hardware → Network Device → VLAN Tag: 20

## Inter-VLAN Routing Patterns

### Allow Specific Services

```
# Allow VLAN 10 to access Jellyfin on VLAN 20
Pass | 192.168.10.0/24 | * | 192.168.20.50 | 8096 | Jellyfin
```

### Allow One-Way Initiation

```
# VLAN 10 can reach VLAN 20, but not reverse
Pass | 192.168.10.0/24 | * | 192.168.20.0/24 | * |
# Implicit stateful allows return traffic
```

### IoT Callbacks to Home Assistant

```
# Allow IoT to reach Home Assistant API
Pass | IoT net | * | 192.168.20.50 | 8123 | HA API

# Allow mDNS for discovery
Pass | IoT net | * | 224.0.0.251 | 5353 | mDNS
```

## mDNS/Bonjour Across VLANs

### Avahi Reflector

On OPNsense: Services → Avahi → Enable

Or run avahi-reflector container:
```yaml
services:
  avahi:
    image: flungo/avahi
    network_mode: host
    volumes:
      - ./avahi-daemon.conf:/etc/avahi/avahi-daemon.conf
```

### Allow mDNS Traffic

```
# Allow mDNS multicast
Pass | * | * | 224.0.0.251 | 5353/UDP | mDNS
```

## Monitoring

### Traffic Between VLANs

```bash
# On router
tcpdump -i vlan30 -n
```

### Firewall Logs

OPNsense: Firewall → Log Files → Live View

Filter by interface to see blocked traffic.

## Troubleshooting

### Device Can't Reach Internet

1. Check VLAN tag on port
2. Verify DHCP lease
3. Check firewall rules (allow internet)
4. Test DNS resolution

### Can't Access Service Cross-VLAN

1. Verify source VLAN tag
2. Check firewall rules (allow specific service)
3. Verify service is listening on correct interface
4. Check for host firewall on destination

### mDNS/Discovery Not Working

1. Enable mDNS reflector
2. Allow multicast traffic
3. Check device supports mDNS

## Best Practices

1. **Deny by default** - whitelist allowed traffic
2. **Isolate IoT** - assume devices are compromised
3. **Guest = Internet only** - no local access
4. **Document everything** - diagram your network
5. **Log blocked traffic** - helps debugging
6. **Test changes** - before locking yourself out
7. **Backup configs** - router/switch configurations
