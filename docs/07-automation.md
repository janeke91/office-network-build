# 07. Automation (Ansible)

## Design

- **Control node:** `ansible` (10.10.10.20), a small Debian VM in MGMT. It holds the SSH key that opens every managed host, so it is treated like the hypervisor and lives only in the management zone.
- **Firewall:** MGMT rules 6, 10 and 11 use the alias `ADMIN_HOSTS` (admin laptop and `ansible`) as source. Ansible reaches VMs on TCP/22 only, like an administrator.
- **Authentication:** SSH key (ed25519) with a passphrase, loaded into `ssh-agent` per session. Privilege escalation through `sudo` without a password for the `janeke` user on lab hosts (`/etc/sudoers.d/janeke`). Fine for a lab; in production it would be limited to the commands the playbooks need. No passwords or keys are stored in this repository.
- **Inventory:** `ansible/inventory.ini`. Groups are named `*_hosts` to avoid clashing with host names.

## Playbooks

| Playbook | What it does |
|---|---|
| `ansible/update.yml` | Refreshes the apt cache, upgrades packages one host at a time (`serial: 1`), compares the running kernel with the newest installed one, prints one line per host (packages upgraded, reboot needed or not). Failed or unreachable hosts show as `NO DATA`. |
| `ansible/harden_ssh.yml` | SSH hardening through the drop-in `sshd_config.d/01-hardening.conf` (password and keyboard-interactive login off, root login off). The file is checked with `sshd -t` before it is written, sshd is reloaded (not restarted), hosts change one at a time, and key login is checked after each host. |
| `ansible/audit.yml` | Read-only audit per host: effective sshd settings, listening ports against a per-host baseline (unexpected and missing), pending security updates. Prints `PASS` or `FAIL` with reasons. |
| `ansible/node_exporter.yml` | Installs and starts `prometheus-node-exporter` on the monitoring VM (idempotent). |
| `ansible/backup.yml` | Backs up every VM on the Proxmox host with `vzdump` (snapshot mode, zstd) to the external disk. Refuses to run if the disk is not mounted, checks that every VM has a fresh non-empty archive, and shuts the host down only with `-e shutdown_after=true` and only if all backups passed. |

Run:

```bash
cd ansible
eval $(ssh-agent) && ssh-add ~/.ssh/ansible_ed25519
ansible all -m ping
ansible-playbook update.yml --check   # dry run
ansible-playbook update.yml
```

## Design decisions

- **No automatic reboots.** Rebooting the Proxmox host stops every VM, so playbooks only report that a reboot is needed.
- **One host at a time** (`serial: 1`), so a failed upgrade affects one host.
- **Dry run first** (`--check`).
- **Not managed yet:** pfSense (not Debian), the Proxmox host, and the Windows domain controller (needs WinRM and its own firewall rules).

## Audit findings

| Finding | Hosts | Decision |
|---|---|---|
| `PasswordAuthentication yes` | all | Fixed with `harden_ssh.yml`. Key-only login verified from the admin laptop. Audit shows `ssh: ok` on all three VMs |
| UDP 68 and 546 listening | all | DHCP client sockets, accepted in the baseline |
| Port 9100 missing | monitoring | Real gap: the monitoring server was not monitored. Fixed with `node_exporter.yml` and a new Prometheus target |
| A random high port seen once | monitoring | Transient socket, gone on re-check |
| UDP 1514 listening | monitoring | Expected: Alloy syslog listener for pfSense logs. Added to the baseline after the log pipeline was built |

A baseline states what is expected. Changing it is valid only when the reason is written down.

## Planned

- Lynis score before and after hardening
- Scheduled run after boot, once a restricted `sudo` policy exists for unattended use
- Manage the Windows domain controller (WinRM and its own firewall rules)

Screenshots: [Ansible](../screenshots/ansible), [Proxmox backups](../screenshots/proxmox)
