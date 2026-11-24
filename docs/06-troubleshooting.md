# 06. Troubleshooting log

Real problems from building the lab. Each entry: symptom, cause, fix.

## 1. Overlapping subnets on LAN and MGMT

- **Symptom:** the laptop got an address from the wrong DHCP server and could not reach Proxmox.
- **Diagnosis:** `tcpdump -e -n -i nic0` and `bridge fdb show` on Proxmox showed untagged frames where tagged ones were expected.
- **Cause:** the untagged LAN and the tagged MGMT both used 10.10.10.1/24.
- **Fix:** moved the temporary LAN to another subnet, enabled MGMT with its own gateway, put the laptop port in access VLAN 10. The LAN address and DHCP were removed at the end.
- **Lesson:** never reuse a subnet on two interfaces. Check tags with a capture before blaming the config.

## 2. Gateway not answering after DHCP worked

- **Cause:** the MGMT interface had no pass rule. DHCP is permitted automatically, so a lease was granted, but ICMP was dropped.
- **Lesson:** a lease proves only that DHCP works.

## 3. "Bad archive mirror" in the Debian installer (about 90 minutes lost)

- **Symptom:** the installer got an address but not the mirror. Packet capture showed DNS queries leaving and no replies.
- **Wrong path:** reconfiguring the DNS resolver.
- **Cause:** the INFRA block rule had destination `*` instead of the private-network alias, so it blocked everything, including the gateway.
- **Fix:** corrected the destination.
- **Lesson:** if an address is obtained but even the gateway does not answer, check firewall rules before DNS. Order: link, address, gateway ping, external IP ping, DNS name.

## 4. Wrong destination in zone exception rules

- **Cause:** *This Firewall* was used for DNS and ICMP. It matches every pfSense address, including other zones' gateways and the WAN.
- **Fix:** use the zone's own interface address.

## 5. Block rule did not block everything

- **Cause:** protocol was TCP instead of *any*, so ICMP and UDP passed.
- **Fix:** protocol *any*.

## 6. Minimal Debian quirks

- netinst left `sources.list` with only a CD-ROM entry. Fix: add the `deb.debian.org` repositories.
- `sudo` and `curl` are not installed by default. Used `su -` and `getent hosts`.
- Docker comes from Docker's own APT repository. `docker-compose-v2` is not in Debian.

## 7. SSH broken on a cloned VM

- **Cause:** host keys were deleted for the clone but not regenerated, and `/run/sshd` was missing.
- **Fix:** `ssh-keygen -A`, `mkdir -p /run/sshd`, correct ownership and mode.
- **Clone checklist:** new hostname, `/etc/hosts`, `/etc/machine-id`, SSH host keys.

## 8. pfSense target UP in Prometheus but absent in Grafana

- **Cause:** the `uname` collector fails on pfSense, so `node_uname_info` is missing.
- **Fix:** textfile metric, see [05 Monitoring](05-monitoring.md).

## 9. Unclean power-off of pfSense

- **Effect:** very long boot with filesystem checks.
- **Mitigation:** a UPS is recommended. AutoConfigBackup protects the configuration.

## 10. Switching a VM from DHCP to a static address

- **Symptom:** SSH to another VLAN failed although the static address looked right. Later `apt` failed with "Temporary failure resolving".
- **Cause (two parts):**
  1. The interface kept the old DHCP address next to the static one. Outgoing packets used the first address, which was not in the firewall alias, so the firewall dropped them.
  2. `/etc/resolv.conf` was written by the DHCP client and had no `nameserver` line. The `dns-nameservers` option in `/etc/network/interfaces` is ignored without the `resolvconf` package.
- **Fix:** remove the DHCP address (or reboot), set `inet static`, write `nameserver` into `/etc/resolv.conf`.
- **Lesson:** after changing addressing, check `ip -br a` for exactly one address and `ip route get <dst>` for the source address used.

## 11. Ansible lessons

- A group and a host must not share a name (`monitoring` as both). Rename the groups.
- `ERROR: unsupported locale setting`: set `LC_ALL=C.UTF-8` on the control node.
- `Incorrect sudo password` on every host: the hidden prompt was typed in the wrong keyboard layout.
- `Could not find the requested service` in `--check` mode: the package is not really installed in a dry run, so a service task cannot find its unit.
- `sudo: unable to resolve host`: the hostname is missing in `/etc/hosts` (clone checklist).
- `PLAY RECAP` hides the useful part. Print an explicit per-host report as the last play.
- Disable password login only after key login works from every admin machine. The playbook runs one host at a time and checks reachability after each change.

## 12. Firewall logs not visible in Grafana

- **Symptom:** Alloy logged `expecting a version value in the range 1-999`. Later Explore showed "No data" for a parsing query.
- **Causes (three):**
  1. pfSense sent the old BSD syslog format (RFC 3164), Alloy expects RFC 5424. Fix: *Status > System Logs > Settings > Log Message Format = RFC 5424*.
  2. Alloy removes the syslog header, so the stored line starts with the rule number. A query for the word `filterlog` matched nothing.
  3. A LogQL `regexp` pattern must be in quotes or backticks.
- **Method:** test the chain link by link: `tcpdump` on the VM (packets arrive), Alloy logs (parse errors), plain `{job="pfsense"}` query (data stored), then add parsing.
- **Lesson:** stored data does not mean the query is right. Check what is actually stored before writing filters.

## 13. Proxmox backup storage unavailable after unplugging the USB disk

- **Symptom:** storage `back-up-HDD` showed a question mark: `failed to create content directory '/mnt/backup/dump' - Operation not permitted`.
- **Cause:** the USB disk was removed while mounted. The mount point is write-protected when nothing is mounted, so Proxmox could not create the directory.
- **Fix:** reconnect the disk, check `lsblk -f` and `dmesg`, run `mount /mnt/backup`, then check `ls /mnt/backup/dump` and `pvesm status`.
- **Lesson:** run `umount /mnt/backup` or shut the host down before removing the disk. Do not run `backup.yml` until `findmnt /mnt/backup` shows the disk.

## 14. Grafana could not send Telegram messages (DNS "server misbehaving")

- **Symptom:** the contact point test failed with `lookup api.telegram.org on 127.0.0.11:53: server misbehaving`.
- **Cause:** containers on a Compose network resolve names through Docker's embedded DNS (127.0.0.11), which forwards to the host's DNS. That forwarding failed on the monitoring VM, although the VM itself resolved names (`getent hosts api.telegram.org` worked) and the firewall allowed DNS to the gateway.
- **Fix:** give the Grafana service its own DNS in `compose.yaml` (`dns: - 10.10.40.1`, the zone's pfSense address) and recreate it with `docker compose up -d`. The test message arrived.
- **Lesson:** test name resolution inside the container, not only on the host.

## 15. Pi-hole did not filter DNS for remote devices

- **Symptom:** with the exit node selected, `nslookup doubleclick.net` returned real addresses instead of `0.0.0.0`. Direct queries to the node timed out or were refused.
- **Causes (five, found one by one):**
  1. No exit node was selected on the client (`tailscale status` showed `offers exit-node`, not `exit node`), so DNS still went through the Wi-Fi resolver.
  2. Pi-hole listened in the default `LOCAL` mode and silently ignored queries from Tailscale addresses (the client saw a timeout). Fix: `sudo pihole-FTL --config dns.listeningMode ALL`, then restart `pihole-FTL`.
  3. The nameserver in the Tailscale admin console pointed to a stale address of another node, so Pi-hole was never asked. Tests against it gave "connection refused".
  4. Tailscale commands on the client need `sudo` (or `tailscale set --operator=$USER` once).
  5. The web interface at `10.10.30.20` timed out: the DMZ is not routed over the VPN. Fix: open it by node name, `http://exit-node-vpn/admin`. No tunnel or firewall rule is needed.
- **Method:** test layer by layer. `dig @127.0.0.1` on the VM (Pi-hole itself), then a query to the node's Tailscale address from the client (network and access policy), then the client's resolver (`resolvectl status`, `tailscale dns status`).
- **Lessons:** get the node's current address from `tailscale status` or `tailscale ping`, not from an old setting. An unreachable global nameserver with *Override local DNS* breaks name resolution on the client. `tailscale down` restores the local resolver while the console is fixed.
