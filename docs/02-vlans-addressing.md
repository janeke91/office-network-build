# 02. VLANs and addressing

## VLAN plan

| VLAN | Name | Subnet | pfSense address | DHCP pool |
|---|---|---|---|---|
| 10 | MGMT | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.100 to .199, known clients only |
| 20 | INFRA | 10.10.20.0/24 | 10.10.20.1 | 10.10.20.100 to .254 |
| 30 | DMZ | 10.10.30.0/24 | 10.10.30.1 | none (static) |
| 40 | MONITORING | 10.10.40.0/24 | 10.10.40.1 | none (static) |

## Static hosts

| Host | Zone | Address |
|---|---|---|
| pfSense | MGMT | 10.10.10.1 |
| Tenda switch | VLAN 1 | 10.10.10.2 |
| Proxmox host | MGMT | 10.10.10.3 |
| Admin laptop | MGMT | 10.10.10.10 (static mapping) |
| ansible (VM 103) | MGMT | 10.10.10.20 (static mapping) |
| tailscale-vpn (VM 106) | MGMT | 10.10.10.30 (static, set in the VM) |
| test-infra (VM 100) | INFRA | 10.10.20.10 |
| dmz (VM 101) | DMZ | 10.10.30.10 |
| exit-node-vpn (VM 107) | DMZ | 10.10.30.20 (static, set in the VM) |
| monitoring (VM 102) | MONITORING | 10.10.40.10 |

The MGMT DHCP server uses "allow known clients from only this interface", so unknown devices do not receive a lease.

## Proxmox network configuration

See [configs/proxmox/interfaces](../configs/proxmox/interfaces).

Key points:

- `vmbr0` is VLAN-aware and carries VLANs 10, 20, 30, 40 on the physical port `nic0`.
- The host address is on **`vmbr0.10`**, not on `vmbr0`. An address on `vmbr0` itself would send untagged frames, which the switch trunk places into VLAN 1.
- VM NICs use VirtIO on `vmbr0` with the VLAN tag set per VM (20, 30, 40).
- All VMs are set to start at boot.

## Design note: why the switch management stays in VLAN 1

The switch is managed through port 4 (PVID 1) only. Moving its management address into a tagged VLAN risks locking the administrator out when a port profile is changed, so it was kept simple and isolated to one physical port.
