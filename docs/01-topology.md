# 01. Topology

## Hardware

| Device | Role | Notes |
|---|---|---|
| Mini PC, Celeron J1900 | pfSense CE 2.7.2 firewall | 8 GB DDR3L, 32 GB SSD, 4x Intel I226-V |
| Tenda TEG2205D | Managed 5-port gigabit switch, 802.1Q | Management IP 10.10.10.2 (stays in VLAN 1) |
| Proxmox VE host | Hypervisor | 16 GB RAM, management IP 10.10.10.3 |
| Laptop | Admin workstation | Static mapping 10.10.10.10 |
| ISP router | Upstream | double NAT in front of pfSense |

## pfSense interfaces

| Interface | Role |
|---|---|
| igc0 | WAN, DHCP from the ISP router |
| igc1 | Trunk to the switch. No IP of its own, parent for VLAN 10/20/30/40 |
| igc2, igc3 | Spare |

`igc1` is the VLAN parent: it has no IP address and no allow rules (the default LAN allow rules were deleted), but it must stay enabled, otherwise the physical link and all VLANs on it go down.

## Switch port map

| Port | Mode | VLANs | Connected to |
|---|---|---|---|
| 1 | Trunk | tagged 10, 20, 30, 40; PVID 1 | pfSense igc1 |
| 2 | Trunk | tagged 10, 20, 30, 40; PVID 1 | Proxmox host |
| 3 | Access | PVID 10 | Admin laptop |
| 4 | Access | PVID 1 | Switch management only (manual 10.10.10.50/24) |
| 5 | Unused | | |

## Design: router-on-a-stick

One physical link (igc1) carries all VLANs as 802.1Q tags. pfSense terminates every VLAN as a routed interface, so all inter-VLAN traffic passes through the firewall policy. The switch only forwards tagged or untagged frames; it makes no policy decisions.

Proxmox uses a VLAN-aware bridge. Each VM NIC gets a VLAN tag in its Proxmox hardware settings, and the bridge tags the frames on the way out.

## Upstream network

pfSense sits behind the ISP router (double NAT). Outbound access is unaffected. Inbound publishing is not planned through port forwarding: external access to DMZ services is intended to use an outbound-only Cloudflare Tunnel.
