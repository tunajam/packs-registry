# VLAN & Firewall Quick Reference

Quick patterns for network segmentation.

## Typical VLAN Layout

| VLAN | Subnet | Purpose |
|------|--------|---------|
| 1 | 192.168.1.0/24 | Management |
| 10 | 192.168.10.0/24 | Trusted |
| 20 | 192.168.20.0/24 | Servers |
| 30 | 192.168.30.0/24 | IoT |
| 40 | 192.168.40.0/24 | Guest |

## OPNsense/pfSense Rule Patterns

### IoT Isolation

```
# Allow DNS to Pi-hole only
Pass | IoT → 192.168.1.10 | 53 | DNS

# Allow MQTT to Home Assistant
Pass | IoT → 192.168.20.50 | 1883 | MQTT

# Block all private IPs
Block | IoT → RFC1918 | * | Block LAN

# Allow internet
Pass | IoT → any | * | Internet
```

### Guest Network

```
# Block everything local
Block | Guest → RFC1918 | * | No LAN

# Allow internet only
Pass | Guest → !RFC1918 | * | Internet
```

### Server Access from Trusted

```
# Full access to server VLAN
Pass | Trusted → Servers | * | All

# Specific services only
Pass | Trusted → 192.168.20.50 | 8096 | Jellyfin
Pass | Trusted → 192.168.20.50 | 32400 | Plex
```

## Switch Port Types

| Type | Purpose | Config |
|------|---------|--------|
| Access | Single VLAN device | Untagged VLAN X |
| Trunk | Router, hypervisor | Tagged all VLANs |
| Hybrid | AP with multiple SSIDs | Native + Tagged |

## mDNS Across VLANs

Enable Avahi/mDNS reflector:
- OPNsense: Services → Avahi
- pfSense: Services → Avahi (package)

Allow multicast:
```
Pass | any → 224.0.0.251 | 5353/UDP | mDNS
```

## Quick Diagnostics

```bash
# Check VLAN tag
tcpdump -i eth0 -e vlan

# Test cross-VLAN
ping -I eth0.30 192.168.20.1

# View firewall logs
tail -f /var/log/filter.log  # pfSense
clog /var/log/filter/latest.log  # OPNsense
```
