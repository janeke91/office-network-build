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

## Remote access

The VM `tailscale-vpn` (10.10.10.30, MGMT) is a Tailscale subnet router. It advertises only `10.10.10.0/24`, so a remote laptop or phone reaches MGMT hosts as if it were on the cable. No port is forwarded: the VM connects out to Tailscale, and the double NAT in front of pfSense does not matter.

- Remote traffic leaves the VM with its own address as source (10.10.10.30), so rule 1 in MGMT is needed to open the pfSense GUI. Other zones are not advertised and stay unreachable from the VPN.
- Login uses an external identity account with two-factor authentication. Key expiry is disabled for the node so the route does not drop silently.
- Client devices use `tailscale up --accept-routes` (Linux) or the app on a phone.

### Exit node and DNS filter for remote users

A second Tailscale node, `exit-node-vpn` (10.10.30.20, DMZ), is an exit node: remote devices can send all internet traffic through the home connection. It runs AdGuard Home (blocklist: AdGuard DNS filter), and the Tailscale admin console points the tailnet DNS at it, so remote devices get ad and tracker blocking.

- The node sits in DMZ on purpose: the DMZ rules block every private network, so traffic of other people cannot reach MGMT or INFRA even if a device is compromised.
- The node has the tag `tag:exit`. The Tailscale access policy gives a restricted user group only DNS (port 53) on that tag and the internet; the admin group has full access. The policy contains `tests` that fail the save if a restricted user could reach Proxmox, the pfSense GUI or the VPN node.
- Limits: DNS filtering does not remove ads served from the same domain as the content (for example YouTube video ads), and all traffic leaves with the home IP address.
