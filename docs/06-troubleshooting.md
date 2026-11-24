# 06. Troubleshooting

Problems I hit while building the lab: what happened, why, and how I fixed it.

## Overlapping subnets on LAN and MGMT

- **What happened:** the laptop got an address from the wrong DHCP server and could not reach Proxmox.
- **How I found it:** `tcpdump -e -n -i nic0` and `bridge fdb show` on Proxmox showed untagged frames where tagged ones were expected.
- **Why:** the untagged LAN and the tagged MGMT both used 10.10.10.1/24.
- **Fix:** moved the temporary LAN to another subnet, enabled MGMT with its own gateway, put the laptop port in VLAN 10. Removed the LAN at the end.
- **Lesson:** never use one subnet on two interfaces. Check tags with a capture before blaming the config.

## Gateway not answering, but DHCP worked

- **Why:** MGMT had no pass rule. DHCP is allowed automatically, so the lease worked, but ping was dropped.
- **Lesson:** a lease only proves that DHCP works.

## "Bad archive mirror" in the Debian installer

About 90 minutes lost.

- **What happened:** the installer got an address but could not reach the mirror. A capture showed DNS queries going out and no replies.
- **Wrong path:** I kept changing the DNS resolver settings.
- **Why:** the INFRA block rule had destination `*` instead of the private-network alias. It blocked everything, even the gateway.
- **Fix:** corrected the destination.
- **Lesson:** if a VM gets an address but even the gateway does not answer, check firewall rules before DNS. Go in order: link, address, gateway ping, internet IP ping, DNS name.

## Wrong destination in exception rules

- **Why:** I used *This Firewall* for DNS and ping. It matches every pfSense address, including other zones' gateways and the WAN.
- **Fix:** use the zone's own interface address.

## Block rule did not block everything

- **Why:** the protocol was TCP, not *any*, so ICMP and UDP got through.
- **Fix:** protocol *any*.

## Minimal Debian quirks

- netinst left only a CD-ROM line in `sources.list`. Added the `deb.debian.org` repositories.
- `sudo` and `curl` are not installed by default. Used `su -` and `getent hosts`.
- Docker comes from Docker's own APT repository. `docker-compose-v2` is not in Debian.

## SSH broken on a cloned VM

- **Why:** the host keys were deleted for the clone but not recreated, and `/run/sshd` was missing.
- **Fix:** `ssh-keygen -A`, `mkdir -p /run/sshd`, correct owner and mode.
- **Clone checklist:** new hostname, `/etc/hosts`, `/etc/machine-id`, SSH host keys.

## pfSense is UP in Prometheus but missing in Grafana

- **Why:** the `uname` collector fails on pfSense, so `node_uname_info` does not exist.
- **Fix:** a textfile metric, see [05 Monitoring](05-monitoring.md).

## Unclean power-off of pfSense

- **Result:** very long boot with filesystem checks.
- **Prevention:** a UPS would help. AutoConfigBackup keeps the config safe.

## Changing a VM from DHCP to a static address

- **What happened:** SSH to another VLAN failed even though the static address looked right. Later `apt` failed with "Temporary failure resolving".
- **Why:**
  - The interface kept the old DHCP address next to the static one. Packets left with the first address, which was not in the firewall alias, so they were dropped.
  - `/etc/resolv.conf` came from the DHCP client and had no `nameserver` line. `dns-nameservers` in `/etc/network/interfaces` is ignored without the `resolvconf` package.
- **Fix:** remove the DHCP address (or reboot), set `inet static`, write `nameserver` into `/etc/resolv.conf`.
- **Lesson:** after changing addresses, check `ip -br a` (one address only) and `ip route get <dst>` (which source address is used).

## Ansible lessons

- A group and a host cannot have the same name (`monitoring` for both). Rename the groups.
- `unsupported locale setting`: set `LC_ALL=C.UTF-8` on the control node.
- `Incorrect sudo password` on every host: I typed the hidden password in the wrong keyboard layout.
- `Could not find the requested service` with `--check`: a dry run does not install the package, so the service does not exist yet.
- `sudo: unable to resolve host`: the hostname is missing in `/etc/hosts`.
- `PLAY RECAP` hides the useful part. Print a short report per host in the last play.
- Turn off password login only after key login works from every admin machine. The playbook changes one host at a time and checks access after each.

## Firewall logs not visible in Grafana

- **What happened:** Alloy logged `expecting a version value in the range 1-999`. Later Explore showed "No data" for a parsing query.
- **Why:**
  - pfSense sent the old syslog format (RFC 3164), but Alloy expects RFC 5424. Fix: *Status > System Logs > Settings > Log Message Format = RFC 5424*.
  - Alloy removes the syslog header, so the stored line starts with the rule number. A query for the word `filterlog` found nothing.
  - A LogQL `regexp` pattern must be in quotes or backticks.
- **How I debugged:** checked the chain step by step: `tcpdump` on the VM (packets arrive), Alloy logs (parse errors), plain `{job="pfsense"}` query (data is stored), then added parsing.
- **Lesson:** stored data does not mean the query is right. Look at what is actually stored first.

## Proxmox backup storage gone after unplugging the USB disk

- **What happened:** storage `back-up-HDD` showed a question mark: `failed to create content directory '/mnt/backup/dump' - Operation not permitted`.
- **Why:** I pulled the disk while it was mounted. The mount point is write-protected when nothing is mounted, so Proxmox could not create the folder.
- **Fix:** reconnect the disk, check `lsblk -f` and `dmesg`, run `mount /mnt/backup`, then check `ls /mnt/backup/dump` and `pvesm status`.
- **Lesson:** run `umount /mnt/backup` or shut the host down before removing the disk. Do not run `backup.yml` until `findmnt /mnt/backup` shows the disk.

## Grafana could not send Telegram messages

- **What happened:** the contact point test failed with `lookup api.telegram.org on 127.0.0.11:53: server misbehaving`.
- **Why:** containers on a Compose network use Docker's built-in DNS (127.0.0.11), which passes queries on to the host's DNS. That did not work on the monitoring VM, although the VM itself resolved names fine and the firewall allowed DNS to the gateway.
- **Fix:** gave the Grafana service its own DNS in `compose.yaml` (`dns: - 10.10.40.1`, the zone's pfSense address) and recreated it with `docker compose up -d`. The test message arrived.
- **Lesson:** test name resolution inside the container, not only on the host.

## Pi-hole did not filter DNS for remote devices

- **What happened:** with the exit node selected, `nslookup doubleclick.net` returned real addresses instead of `0.0.0.0`. Direct queries to the node timed out or were refused.
- **Why (five separate causes):**
  - The exit node was not selected on the client (`tailscale status` showed `offers exit-node`, not `exit node`), so DNS still went through Wi-Fi.
  - Pi-hole was in the default `LOCAL` listening mode and silently ignored queries from Tailscale addresses. Fix: `sudo pihole-FTL --config dns.listeningMode ALL`, then restart `pihole-FTL`.
  - The nameserver in the Tailscale admin console was an old address of another node, so Pi-hole was never asked.
  - Tailscale commands on the client need `sudo` (or run `tailscale set --operator=$USER` once).
  - The web page at `10.10.30.20` timed out because the VPN does not route to the DMZ. Fix: open it by node name, `http://exit-node-vpn/admin`. No tunnel or firewall rule needed.
- **How I debugged:** layer by layer. First `dig @127.0.0.1` on the VM (is Pi-hole itself fine?), then a query to the node's Tailscale address from the client (network and access policy), then the client's own resolver (`resolvectl status`, `tailscale dns status`).
- **Lessons:** take the node's current address from `tailscale status` or `tailscale ping`, not from an old setting. A dead global nameserver with *Override local DNS* breaks DNS on the client. `tailscale down` restores the local resolver while the console is fixed.
