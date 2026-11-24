# 03. Firewall rules (pfSense)

## Access summary (as built)

| From \ To | MGMT | INFRA | DMZ | MONITORING | Internet |
|---|---|---|---|---|---|
| MGMT | n/a | SSH from admin PC and ansible; RDP to dc-01 from the same hosts | SSH only, from admin PC and ansible | SSH, Grafana, Prometheus (admin PC) | web (80/443), NTP |
| INFRA | blocked | n/a | blocked | blocked | web (80/443), NTP |
| DMZ | blocked | blocked | n/a | blocked | web (80/443), NTP |
| MONITORING | blocked | 9100 only | 9100 only | n/a | web (80/443), NTP |

Admin access is limited to two hosts: the admin PC (10.10.10.10) and the Ansible control node (10.10.10.20). Proxmox accepts only ports 22 and 8006. Every zone can also use DNS and ping on its own gateway address.

## How rules are evaluated

- Rules are checked top to bottom on the interface where the packet enters. The first match wins.
- No match means drop (default deny).
- Rules are stateful: replies to allowed connections pass automatically.
- A rule matches only if protocol, source, destination and port all match. ICMP has no ports.
- Destination *INFRA address* means only the interface address (10.10.20.1). *This Firewall* covers every pfSense address and would break isolation, so it is not used.

## Aliases

| Alias | Type | Values |
|---|---|---|
| RFC1918 | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| web_ports | Port | 80, 443 |
| ADMIN_HOSTS | Host | 10.10.10.10 (admin laptop), 10.10.10.20 (ansible) |
| NODE_EXPORTERS | Host | 10.10.10.3, 10.10.40.1 |
| SSH_TARGETS | Host | 10.10.10.30, 10.10.30.20, 10.10.40.10 |
| proxmox_ports | Port | 22, 8006 |
| monitoring_ports | Port | 3000, 9090 |

In a pfSense rule, `22 - 8006` is a range of about 8000 ports. Use a port alias for a list.

## Pattern

Each restricted zone follows the same order: narrow pass exceptions, then a logged block to RFC1918, then a pass to the internet. Private addresses not explicitly allowed are blocked. Everything else is the public internet.

## MGMT (VLAN 10)

Hosts: admin laptop 10.10.10.10, ansible 10.10.10.20, `tailscale-vpn` 10.10.10.30, `homepage` 10.10.10.40, Proxmox 10.10.10.3.

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | 10.10.10.40 | 10.10.10.1 | 443 | Homepage check of the pfSense GUI |
| 2 | Pass | TCP | 10.10.10.40 | 10.10.30.20 | 80 | Homepage check of Pi-hole |
| 3 | Pass | TCP | 10.10.10.40 | 10.10.40.10 | monitoring_ports | Homepage checks and Prometheus metrics |
| 4 | Pass | TCP | 10.10.10.30 | 10.10.40.10 | monitoring_ports | Grafana and Prometheus from the VPN node |
| 5 | Pass | TCP | 10.10.10.30 | MGMT address | 443 | pfSense GUI from the VPN node |
| 6 | Pass | TCP | ADMIN_HOSTS | 10.10.20.5 | 3389 | RDP to dc-01 |
| 7 | Pass | TCP | 10.10.10.10 | MGMT address | 443 | pfSense GUI |
| 8 | Pass | TCP/UDP | MGMT subnet | MGMT address | 53 | DNS |
| 9 | Pass | ICMP | MGMT subnet | MGMT address | n/a | Gateway ping |
| 10 | Pass | TCP | ADMIN_HOSTS | 10.10.10.3 | proxmox_ports | Proxmox GUI and SSH |
| 11 | Pass | TCP | ADMIN_HOSTS | SSH_TARGETS | 22 | SSH to VMs |
| 12 | Pass | TCP | 10.10.10.10 | 10.10.40.10 | monitoring_ports | Grafana, Prometheus |
| 13 | Block (log) | any | MGMT subnet | RFC1918 | any | Block internal networks |
| 14 | Pass | TCP | MGMT subnet | any | web_ports | Updates, Tailscale, Homepage weather widget |
| 15 | Pass | UDP | MGMT subnet | any | 123 | NTP |

Notes:

- Every exception sits above the block rule (13). A pass rule for a private address below it never matches.
- Ansible uses `ADMIN_HOSTS` and reaches VMs on TCP/22 only, like an administrator.
- The old temporary *pass any* rule is gone. The anti-lockout rule only covers LAN, which has no address now, so rule 7 keeps the GUI reachable from the laptop. Before editing MGMT rules, keep a config backup and console access.
- Rules 4 and 5: remote traffic leaves the VPN node with its own address as source. Hosts in 10.10.10.0/24 share its subnet, so that traffic does not pass pfSense. The pfSense GUI and the monitoring host (another zone) do.
- Rules 1 to 3 give the Homepage VM only the destinations it needs. It is not in `ADMIN_HOSTS`, because that alias also opens SSH and the Proxmox GUI. Dashboard links are opened by the browser of whoever clicks them, so these rules cover only status checks and Prometheus queries.

## INFRA (VLAN 20)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | INFRA subnet | INFRA address | 53 | DNS |
| 2 | Pass | ICMP | INFRA subnet | INFRA address | n/a | Gateway ping |
| 3 | Block (log) | any | INFRA subnet | RFC1918 | any | Block internal networks |
| 4 | Pass | TCP | INFRA subnet | any | web_ports | Updates and downloads |
| 5 | Pass | UDP | INFRA subnet | any | 123 | NTP |

Rule 4 used to be *pass any*. It was narrowed so VMs reach only web and time servers. ICMP to the internet is no longer allowed.

## DMZ (VLAN 30)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ subnet | DMZ address | 53 | DNS |
| 2 | Pass | ICMP | DMZ subnet | DMZ address | n/a | Gateway ping |
| 3 | Block (log) | any | DMZ subnet | RFC1918 | any | |
| 4 | Pass | TCP | DMZ subnet | any | web_ports | HTTP/HTTPS only |
| 5 | Pass | UDP | DMZ subnet | any | 123 | NTP |

`exit-node-vpn` (10.10.30.20) uses these rules without exceptions. Tailscale connects out over TCP 443. Pi-hole on the same VM uses pfSense (10.10.30.1) as upstream, because DMZ allows DNS only to the gateway. Remote users' traffic leaving through the node falls under rule 3, so it cannot reach any private network.

Planned: TCP/UDP 7844 outbound for `cloudflared`.

## MONITORING (VLAN 40)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | MONITORING subnet | MONITORING address | 53 | DNS |
| 2 | Pass | ICMP | MONITORING subnet | MONITORING address | n/a | Gateway ping |
| 3 | Pass | TCP | 10.10.40.10 | NODE_EXPORTERS | 9100 | Scrape node_exporters |
| 4 | Block (log) | any | MONITORING subnet | RFC1918 | any | |
| 5 | Pass | TCP | MONITORING subnet | any | web_ports | Image pulls and updates |
| 6 | Pass | UDP | MONITORING subnet | any | 123 | NTP |

## Lessons

- Exceptions go above the block. Rule 3 in MONITORING would never fire below rule 4.
- A block rule with protocol TCP does not stop ICMP or UDP. Use *any*.
- Describe every logged rule, or the log shows only `USER_RULE (id)`.

Screenshots: [screenshots/firewall-rules](../screenshots/firewall-rules)
