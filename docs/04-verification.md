# 04. Verification

Each rule is tested from the zone it applies to. Tests 1 to 17 used two small test VMs (`test-infra` in INFRA, `dmz` in DMZ). They were removed later to free resources; the rules they tested did not change. Screenshots: [screenshots/verification](../screenshots/verification).

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
| 19 | Phone on mobile data | Tailscale on, open `https://10.10.10.3:8006` and `https://10.10.10.1` | Both open | Pass |
| 20 | Phone on mobile data | Tailscale on, `exit-node-vpn` selected, open an "IP address" website | Home ISP address, not the mobile operator's | Pass |
| 21 | exit-node-vpn | `dig @127.0.0.1 doubleclick.net +short` | `0.0.0.0`; query log shows "Blocked" | Pass |
| 22 | Tailscale access policy | Policy `tests`: restricted user to Proxmox, pfSense GUI, VPN node, Pi-hole panel | Denied; DNS (53) on the exit node allowed | Pass |
| 23 | Phone or laptop on Tailscale | Open `http://10.10.40.10:3000` (Grafana) and `:9090` (Prometheus) | Both open | Pass |
| 24 | homepage (MGMT) | Status indicators for pfSense GUI, Pi-hole, Grafana, Prometheus; Proxmox widget values from Prometheus | All green, metrics shown | Pass |
| 25 | Laptop on Tailscale | `exit-node-vpn` selected, `nslookup doubleclick.net` | `0.0.0.0`; the laptop appears in the Pi-hole query log | Pass |

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

On pfSense, *Status > System Logs > Firewall* shows blocked traffic with the rule description. *Diagnostics > Packet Capture* sees packets before the firewall decides: a packet in the capture but not in the logs points to a rule problem, not a link problem.
