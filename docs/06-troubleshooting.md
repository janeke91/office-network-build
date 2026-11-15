# 06. Troubleshooting log

Real problems met while building the lab. Each entry: symptom, diagnosis, root cause, fix.

## 1. Overlapping subnets on LAN and MGMT

- **Symptom:** the laptop received an address from the wrong DHCP server and could not reach Proxmox.
- **Diagnosis:** `tcpdump -e -n -i nic0` and `bridge fdb show` on Proxmox showed untagged frames where tagged ones were expected.
- **Root cause:** the untagged LAN and tagged MGMT both used 10.10.10.1/24.
- **Fix:** moved the temporary LAN to a different subnet, enabled MGMT with its own gateway, and put the laptop port in access VLAN 10. The LAN address and DHCP were removed at the end.
- **Lesson:** never reuse a subnet on two interfaces, and check tags with a capture before blaming the config.

## 2. Gateway not answering after DHCP worked

- **Cause:** the MGMT interface had no pass rule. DHCP is auto-permitted, so a lease was granted, but ICMP was dropped.
- **Lesson:** a lease proves only that DHCP works.

## 3. "Bad archive mirror" in the Debian installer (about 90 minutes lost)

- **Symptom:** the installer got an address but could not reach the mirror. Packet capture showed DNS queries leaving but no replies.
- **Wrong path:** reconfiguring the DNS resolver (listen interfaces, localhost).
- **Root cause:** the INFRA block rule had destination `*` instead of the private-network alias, so it blocked everything, including the gateway.
- **Fix:** corrected the destination.
- **Lesson:** if an address is obtained but even the gateway does not answer, look at firewall rules first, not DNS. Diagnose in order: link, address, gateway ping, external IP ping, DNS name.

## 4. Wrong destination in zone exception rules

- **Cause:** *This Firewall* was used for DNS and ICMP. It matches every pfSense address, including other zones' gateways and the WAN.
- **Fix:** use the zone's own interface address.

## 5. Block rule did not block everything

- **Cause:** protocol was TCP instead of *any*, so ICMP and UDP passed.
- **Fix:** protocol *any*.

## 6. Minimal Debian quirks

- netinst left `sources.list` with only a CD-ROM entry. Fixed by adding the `deb.debian.org` repositories.
- `sudo` and `curl` are not installed by default. Used `su -` and `getent hosts`.
- Docker comes from Docker's own APT repository; `docker-compose-v2` is not in Debian.

## 7. SSH broken on a cloned VM

- **Cause:** host keys were deleted for the clone but not regenerated, and `/run/sshd` was missing.
- **Fix:** `ssh-keygen -A`, `mkdir -p /run/sshd`, correct ownership and mode.
- **Checklist for clones:** new hostname, `/etc/hosts`, `/etc/machine-id`, SSH host keys.

## 8. pfSense target UP in Prometheus but absent in Grafana

- **Cause:** the `uname` collector fails on pfSense, so `node_uname_info` is missing.
- **Fix:** textfile metric (see [05 Monitoring](05-monitoring.md)).

## 9. Unclean power-off of pfSense

- **Effect:** very long boot with filesystem checks.
- **Mitigation:** UPS recommended; AutoConfigBackup protects the configuration.

## 10. Switching a VM from DHCP to a static address

- **Symptom:** SSH to another VLAN failed although the new static address looked correct; later `apt` failed with "Temporary failure resolving".
- **Root cause (two parts):**
  1. The interface kept the old DHCP address next to the static one. Outgoing packets used the first address, which was not in the firewall alias, so the firewall dropped them.
  2. `/etc/resolv.conf` had been written by the DHCP client and contained no `nameserver` line. The `dns-nameservers` option in `/etc/network/interfaces` is ignored without the `resolvconf` package.
- **Fix:** remove the DHCP address (or reboot), set `inet static`, write `nameserver` into `/etc/resolv.conf`.
- **Lesson:** after changing addressing, check `ip -br a` for exactly one address and `ip route get <dst>` for the source address used.

## 11. Ansible lessons

- A group and a host must not share a name (`monitoring` as both): rename the groups.
- `ERROR: unsupported locale setting`: set `LC_ALL=C.UTF-8` on the control node.
- `Incorrect sudo password` on every host: the hidden prompt was typed in the wrong keyboard layout.
- `Could not find the requested service` in `--check` mode: the package was not really installed in a dry run, so a service task cannot find its unit.
- `sudo: unable to resolve host`: the hostname is missing in `/etc/hosts` (clone checklist).
- `PLAY RECAP` hides the useful part; print an explicit per-host report as the last play.
- Disabling password login is safe only after key login is verified from every admin machine; the playbook runs one host at a time and checks reachability after each change.

## 12. Firewall logs not visible in Grafana

- **Symptom:** Alloy logged `expecting a version value in the range 1-999`, later Explore showed "No data" for a parsing query.
- **Root causes (three):**
  1. pfSense sent the old BSD syslog format (RFC 3164) while Alloy expects RFC 5424. Fix: Status > System Logs > Settings > Log Message Format = RFC 5424.
  2. Alloy removes the syslog header, so the stored line starts with the rule number. A query that searched for the word `filterlog` matched nothing.
  3. A LogQL `regexp` pattern must be inside quotes or backticks; without them the editor shows a red underline.
- **Method:** test the chain link by link: `tcpdump` on the VM (packets arrive), Alloy logs (parse errors), plain `{job="pfsense"}` query (data stored), then add parsing.
- **Lesson:** a part of the chain working (data stored) does not mean the query is right. Check what is actually stored before writing filters.

## 13. Proxmox backup storage unavailable after unplugging the USB disk

- **Symptom:** the storage `back-up-HDD` showed a question mark; the error was `failed to create content directory '/mnt/backup/dump' - Operation not permitted`.
- **Cause:** the USB disk was disconnected while mounted. The mount point is protected against writes when nothing is mounted, so Proxmox could not create the directory.
- **Fix:** reconnect the disk, check `lsblk -f` and `dmesg`, mount it (`mount /mnt/backup`), verify the old backups with `ls /mnt/backup/dump` and the storage state with `pvesm status`.
- **Lesson:** unmount (`umount /mnt/backup`) or shut the host down before removing a disk, and do not run `backup.yml` until `findmnt /mnt/backup` shows the disk.

## 14. Grafana could not send Telegram messages (DNS "server misbehaving")

- **Symptom:** the contact point test failed with `lookup api.telegram.org on 127.0.0.11:53: server misbehaving`.
- **Cause:** a container on a Compose network resolves names through Docker's embedded DNS (127.0.0.11), which forwards to the DNS of the host. This forwarding did not work from the monitoring VM, although the VM itself resolved names fine (`getent hosts api.telegram.org` worked). The MONITORING firewall rules already allowed DNS to the gateway.
- **Fix:** give the Grafana service its own DNS server in `compose.yaml` (`dns: - 10.10.40.1`, the pfSense address of the zone) and recreate the container (`docker compose up -d`). The test message arrived.
- **Lesson:** test name resolution from inside the container (not only from the host) and check which resolver the container actually uses.

## 15. Pi-hole did not filter DNS for remote devices

- **Symptom:** with the exit node selected, `nslookup doubleclick.net` returned real addresses instead of `0.0.0.0`; direct queries to the node timed out or were refused.
- **Causes (five, found one by one):**
  1. The Tailscale client had no exit node selected (`tailscale status` showed `offers exit-node`, not `exit node`), so DNS still went through the Wi-Fi resolver.
  2. Pi-hole listened in the default `LOCAL` mode and silently ignored queries from Tailscale addresses (client saw a timeout). Fix: `sudo pihole-FTL --config dns.listeningMode ALL`, then restart `pihole-FTL`.
  3. The nameserver in the Tailscale admin console pointed to a wrong, stale address (another node), so even a working Pi-hole was never asked. Tests against that address gave "connection refused".
  4. Tailscale commands on the client need `sudo` (or `tailscale set --operator=$USER` once).
  5. Opening the web interface: `10.10.30.20` is in the DMZ, which the VPN does not route to, so the page timed out. Fix: open it by the node name over Tailscale (`http://exit-node-vpn/admin`) instead of the address; no tunnel or firewall rule is needed.
- **Method:** test layer by layer. First `dig @127.0.0.1` on the VM (Pi-hole itself), then a query to the node's Tailscale address from the client (network and access policy), then the client's own resolver (`resolvectl status`, `tailscale dns status`).
- **Lesson:** check the exact address of the node with `tailscale status` or `tailscale ping` instead of trusting an old setting, and an unreachable global nameserver with *Override local DNS* breaks name resolution on the client; `tailscale down` restores the local resolver while fixing the console.
