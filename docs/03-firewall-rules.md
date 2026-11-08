# 03. Firewall rules (pfSense)

## Evaluation model

- Rules are evaluated **top to bottom on the interface where the packet enters**. The first match wins.
- If nothing matches, the packet is dropped (default deny).
- Rules are stateful: replies to allowed connections are permitted automatically.
- A rule matches only if protocol, source, destination and port all match.
- ICMP is its own protocol and has no ports.
- Destination *INFRA address* means only the interface address (10.10.20.1). *This Firewall* would mean every pfSense address and would break isolation, so it is not used.

## Aliases

| Alias | Type | Values |
|---|---|---|
| RFC1918 | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| web_ports | Port | 80, 443 |
| ADMIN_HOSTS | Host | 10.10.10.10 (admin laptop), 10.10.10.20 (ansible) |
| NODE_EXPORTERS | Host | 10.10.10.3, 10.10.20.10, 10.10.30.10, 10.10.40.1 |

## Pattern

For each restricted zone the order is: narrow *pass* exceptions, then a logged *block to RFC1918*, then the *pass* for the internet. Anything private that is not explicitly allowed is blocked by the block rule; anything else is the public internet.

## MGMT (VLAN 10)

Admin workstation: 10.10.10.10 (static DHCP mapping). Ansible control node `ansible`: 10.10.10.20. Proxmox: 10.10.10.3. Rules 1, 5 and 6 use the alias `ADMIN_HOSTS`, so Ansible reaches the VMs on TCP/22 only, like an administrator. Every exception sits above the block rule (8): a pass rule for a private address placed below it would never match.

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | ADMIN_HOSTS | 10.10.20.5 | 3389 | RDP to dc-01 |
| 2 | Pass | TCP | 10.10.10.10 | MGMT address | 443 | pfSense admin GUI |
| 3 | Pass | TCP/UDP | MGMT subnet | MGMT address | 53 | DNS |
| 4 | Pass | ICMP | MGMT subnet | MGMT address | n/a | Gateway ping |
| 5 | Pass | TCP | ADMIN_HOSTS | 10.10.10.3 | proxmox_ports (22, 8006) | Proxmox GUI and SSH |
| 6 | Pass | TCP | ADMIN_HOSTS | SSH_TARGETS | 22 | SSH to VMs |
| 7 | Pass | TCP | 10.10.10.10 | 10.10.40.10 | monitoring_ports (3000, 9090) | Grafana, Prometheus |
| 8 | Block (log) | any | MGMT subnet | RFC1918 | any | "block internal networks" |
| 9 | Pass | TCP | MGMT subnet | any | web_ports | Updates |
| 10 | Pass | UDP | MGMT subnet | any | 123 | NTP |

The former temporary *pass any* rule was removed. pfSense's built-in anti-lockout rule only covers the LAN interface, which no longer has an address, so rule 2 is what keeps the GUI reachable: keep it above the block rule and keep a config backup and console access before editing MGMT rules.

Additional aliases: `SSH_TARGETS` (10.10.20.10, 10.10.30.10, 10.10.40.10), `proxmox_ports` (22, 8006), `monitoring_ports` (3000, 9090).

Note: in a pfSense rule, `22 - 8006` is a *range* of about 8000 ports. Use a port alias for a list.

## INFRA (VLAN 20)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | INFRA subnet | INFRA address | 53 | DNS |
| 2 | Pass | ICMP | INFRA subnet | INFRA address | n/a | Gateway ping |
| 3 | Block (log) | any | INFRA subnet | RFC1918 | any | "Block internal networks" |
| 4 | Pass | TCP | INFRA subnet | any | web_ports | Updates and downloads |
| 5 | Pass | UDP | INFRA subnet | any | 123 | NTP |

Earlier versions had *pass any* as rule 4. It was narrowed so a VM can only reach the web and time servers; ICMP to the internet is no longer allowed.

## DMZ (VLAN 30)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ subnet | DMZ address | 53 | DNS |
| 2 | Pass | ICMP | DMZ subnet | DMZ address | n/a | Gateway ping |
| 3 | Block (log) | any | DMZ subnet | RFC1918 | any | |
| 4 | Pass | TCP | DMZ subnet | any | web_ports | HTTP/HTTPS only |
| 5 | Pass | UDP | DMZ subnet | any | 123 | NTP |

Planned: TCP/UDP 7844 outbound for `cloudflared`.

## MONITORING (VLAN 40)

| # | Action | Proto | Source | Destination | Port | Note |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | MONITORING subnet | MONITORING address | 53 | DNS |
| 2 | Pass | ICMP | MONITORING subnet | MONITORING address | n/a | Gateway ping |
| 3 | Pass | TCP | 10.10.40.10 | NODE_EXPORTERS | 9100 | "scrape node_exporters": one source, one destination alias, one port |
| 4 | Block (log) | any | MONITORING subnet | RFC1918 | any | |
| 5 | Pass | TCP | MONITORING subnet | any | web_ports | Image pulls and updates |
| 6 | Pass | UDP | MONITORING subnet | any | 123 | NTP |

## Lessons embedded in the rules

- Exception rules must sit **above** the block. Rule 3 in MONITORING would never fire if placed below rule 4.
- A block rule with protocol *TCP* does not stop ICMP or UDP. Use *any*.
- Give every logged rule a description. Otherwise the log shows `USER_RULE (id)`.

Screenshots: [screenshots/firewall-rules](../screenshots/firewall-rules)
