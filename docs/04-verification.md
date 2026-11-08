# 04. Verification

Every rule is tested from the zone it applies to. Results are from the built lab. Screenshots: [screenshots/verification](../screenshots/verification).

| # | From | Test | Expected | Result |
|---|---|---|---|---|
| 1 | INFRA VM | `ping 10.10.20.1` | Reply | Pass |
| 2 | INFRA VM | `getent hosts debian.org` | Resolves via pfSense | Pass |
| 3 | INFRA VM | `curl -sI https://debian.org` | HTTP response (web ports allowed) | Pass |
| 4 | INFRA VM | `nc -zv 8.8.8.8 22` | Timeout (only web ports and NTP leave INFRA) | Pass |
| 5 | INFRA VM | `ping 10.10.30.10` (DMZ) | No reply, firewall log "Block internal networks" | Pass |
| 6 | INFRA VM | `ping <ISP router address>` | No reply | Pass |
| 7 | INFRA VM | `ping 10.10.10.3` (Proxmox) | No reply | Pass |
| 8 | DMZ VM | gateway, DNS, internet | Work | Pass |
| 9 | DMZ VM | ping INFRA, ISP router, Proxmox | No reply, logged | Pass |
| 10 | MONITORING VM | `nc -zv 10.10.20.10 9100` | Open | Pass |
| 11 | MONITORING VM | `nc -zv 10.10.20.10 22` | Blocked | Pass |
| 12 | Laptop (MGMT) | SSH to each VM | Works | Pass |
| 13 | Laptop (MGMT) | pfSense and Proxmox web UI | Open | Pass |
| 14 | Laptop (MGMT) | `ping 10.10.10.1` | Reply | Pass |
| 15 | Laptop (MGMT) | `ping <ISP router address>` | No reply, logged | Pass |
| 16 | Laptop (MGMT) | `ping 10.10.20.10` while SSH works | No reply, SSH ok | Pass |
| 17 | Laptop (MGMT) | `ping 1.1.1.1` | No reply (no ICMP rule to internet) | Pass |
| 18 | All | Reboot pfSense, then Proxmox | MGMT, VMs and Grafana return without manual action | Pass |

## Useful commands

```bash
# Is a port open? (from a VM)
nc -zv 10.10.20.10 9100

# Which services listen locally?
ss -tulpn

# Frames and VLAN tags on the Proxmox uplink
tcpdump -e -n -i nic0 vlan

# Bridge state on Proxmox
bridge vlan show
bridge fdb show
ip neigh
```

On pfSense: *Status > System Logs > Firewall* shows blocked traffic with the rule description. *Diagnostics > Packet Capture* sees packets before the firewall decides, so a packet visible in capture and absent in logs points to a rule problem, not a link problem.
