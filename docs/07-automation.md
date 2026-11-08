# 07. Automation (Ansible)

## Design

- **Control node:** `ansible` (10.10.10.20), a small Debian VM in MGMT (VLAN 10). It holds the SSH key that opens every managed host, so it is treated like the hypervisor: it lives only in the management zone.
- **Firewall:** MGMT rules 5 and 6 use the alias `ADMIN_HOSTS` (admin laptop and `ansible`) as source. Everything else stays unchanged: Ansible reaches the VMs only on TCP/22, like an administrator.
- **Authentication:** SSH key (ed25519) protected by a passphrase, loaded into `ssh-agent` per session. Privilege escalation through `sudo` with the password entered at run time (`--ask-become-pass`). No passwords or keys are stored in this repository.
- **Inventory:** `ansible/inventory.ini`. Groups are named `*_hosts` to avoid clashing with host names.

## Playbooks

| Playbook | What it does |
|---|---|
| `ansible/update.yml` | Refreshes the apt cache, upgrades packages one host at a time (`serial: 1`), compares the running kernel with the newest installed kernel, and prints a one-line report per host (packages upgraded, reboot required or not). Hosts that fail or are unreachable show up as `NO DATA` in the report. |
| `ansible/harden_ssh.yml` | SSH hardening: drop-in `sshd_config.d/01-hardening.conf` (password and keyboard-interactive login off, root login off). The file is validated with `sshd -t` before it is written, sshd is reloaded (not restarted), hosts are changed one at a time and key-based reachability is checked after each host. |
| `ansible/audit.yml` | Read-only security audit per host: effective sshd settings (root login, password authentication), listening ports compared with a per-host baseline (unexpected and missing ports), pending security updates. Prints `PASS` or `FAIL` with reasons. |
| `ansible/node_exporter.yml` | Installs and starts `prometheus-node-exporter` on the monitoring VM (idempotent). |
| `ansible/backup.yml` | Backs up every VM on the Proxmox host with `vzdump` (snapshot mode, zstd) to the external disk. Refuses to run if the disk is not mounted, verifies that every VM has a fresh non-empty archive, and shuts the host down afterwards only when asked (`-e shutdown_after=true`) and only if all backups passed. |

Run:

```bash
cd ansible
eval $(ssh-agent) && ssh-add ~/.ssh/ansible_ed25519
ansible all -m ping
ansible-playbook update.yml --check --ask-become-pass   # dry run
ansible-playbook update.yml --ask-become-pass
```

## Design decisions

- **No automatic reboots.** A reboot of the Proxmox host stops every VM, so playbooks only report that a reboot is needed.
- **One host at a time** (`serial: 1`): a failed upgrade affects a single host.
- **Dry run first** (`--check`).
- **Not managed yet:** pfSense (not Debian), the Proxmox host and the Windows domain controller (needs WinRM and its own firewall rules).

## Audit findings and what was done

| Finding | Hosts | Decision |
|---|---|---|
| `PasswordAuthentication yes` | all | Fixed with `harden_ssh.yml`: password login is off, key-only login verified from the admin laptop. Audit shows `ssh: ok` on all three VMs |
| UDP 68 and 546 listening | all | DHCP client sockets, accepted in the baseline (documented in the playbook) |
| Port 9100 missing | monitoring | Real gap: the monitoring server was not monitored. Fixed with `node_exporter.yml` and a new Prometheus target |
| A random high port seen once | monitoring | Transient socket, gone on re-check |
| UDP 1514 listening | monitoring | Expected: Alloy syslog listener for pfSense firewall logs. Added to the audit baseline after the log pipeline was built |

An audit baseline is a statement of what is expected. Fixing the baseline is only valid when the reason is written down.

## Planned

- Lynis score before and after hardening
- Run on a schedule after boot, once a restricted `sudo` policy is defined for unattended use
- Manage the Windows domain controller (WinRM and its own firewall rules)

Screenshots: [Ansible](../screenshots/ansible), [Proxmox backups](../screenshots/proxmox)
