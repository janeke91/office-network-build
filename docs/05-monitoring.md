# 05. Monitoring

## Architecture

Prometheus (pull model) runs in Docker on the `monitoring` VM (10.10.40.10, VLAN 40) together with Grafana. Each node runs `node_exporter` on port 9100. The firewall allows exactly one flow for this: 10.10.40.10 to `NODE_EXPORTERS` on TCP/9100.

| Job | Target | Label |
|---|---|---|
| proxmox | 10.10.10.3:9100 | |
| vms | 10.10.40.10:9100 | monitoring |
| pfsense | 10.10.40.1:9100 | pfsense |

Configuration: [configs/monitoring/prometheus.yml](../configs/monitoring/prometheus.yml) and [compose.yaml](../configs/monitoring/compose.yaml).

The monitoring VM scrapes itself: `node_exporter` is installed there by `ansible/node_exporter.yml`. A security audit found this gap (the monitoring server was not monitored); see [07 Automation](07-automation.md).

## Grafana

- Data source: Prometheus at `http://prometheus:9090` (Compose service name).
- Dashboard: Node Exporter Full (ID 1860). Its Job and Host selectors are built from `node_uname_info`.

## pfSense exporter

The pfSense `node_exporter` package is bound to the MONITORING interface only. Extra flags: `--log.level=warn --no-collector.zfs`.

Problem: the `uname` collector fails with "cannot allocate memory" on pfSense, so `node_uname_info` is missing and the dashboard cannot list the host. Workaround: a textfile metric.

```sh
printf 'node_uname_info{machine="%s",nodename="%s",release="%s",sysname="%s"} 1\n' \
  "$(uname -m)" "$(uname -n)" "$(uname -r)" "$(uname -s)" \
  > /var/tmp/node_exporter/uname.prom
```

The textfile directory must match the exporter's `--collector.textfile.directory`. If `/var/tmp` is RAM-backed the file may vanish on reboot, so re-check after restarting pfSense.

## Firewall logs (Loki)

Firewall decisions are visualised from pfSense logs.

```
pfSense (remote syslog, UDP 1514, RFC 5424) -> Alloy -> Loki -> Grafana
```

- pfSense: Status > System Logs > Settings. Remote logging to `10.10.40.10:1514`, source address on the MONITORING interface, contents: firewall events only, message format RFC 5424. The traffic starts on the firewall itself in the MONITORING subnet, so no interface rule is needed.
- `monitoring` VM: Loki (7 days retention) and Grafana Alloy (syslog receiver) run in the same Compose file. Configs: [loki-config.yaml](../configs/monitoring/loki-config.yaml), [alloy-config.alloy](../configs/monitoring/alloy-config.alloy).
- Grafana: Loki data source `http://loki:3100`. Dashboard: [pfsense-firewall-logs.json](../configs/monitoring/dashboards/pfsense-firewall-logs.json) (import it and pick the Loki data source).
- Alloy keeps only the message body, so a `filterlog` line starts with the rule number. The dashboard parses the CSV fields (`rule`, `tracker`, `iface`, `action`, `src`, `dst`, `proto`, `sport`, `dport`) with a LogQL `regexp` stage.
- Interfaces map to zones: `igc1.10` MGMT, `igc1.20` INFRA, `igc1.30` DMZ, `igc1.40` MONITORING, `igc0` WAN. WAN default-deny noise (broadcasts from the home network) is shown separately so it does not hide the zone blocks.

## Homepage dashboard

The VM `homepage` (VM 108, 10.10.10.40, MGMT) runs [Homepage](https://gethomepage.dev) in Docker as a single start page for the lab: one tile per service with a status indicator, and the Proxmox host metrics taken from Prometheus (temperature, CPU, RAM, root disk). Files: [configs/homepage](../configs/homepage).

- Status indicators (`siteMonitor`) are checked by the container itself, so MGMT has three narrow rules for it (rules 1 to 3 in [03 Firewall rules](03-firewall-rules.md)). A tile link is opened by the visitor's browser, not by the VM.
- Metrics use the `prometheusmetric` widget with PromQL queries against `http://10.10.40.10:9090`. The Proxmox temperature comes from `node_hwmon_temp_celsius` (lm-sensors on the host, exposed by node_exporter).
- The pfSense box (J1900) exposes no temperature sensor, so only the Proxmox host shows temperature.
- No secrets are needed: the widgets read Prometheus without credentials. A Proxmox API token or a Tailscale API key would be kept in an untracked `.env` file (see `.gitignore`).
- The wallpaper and personal bookmarks are not stored in the repository.

## Retention and persistence

Prometheus keeps 15 days of data in the `prom-data` volume, Loki 7 days in `loki-data`. Grafana state lives in `grafana-data`. Both containers use `restart: unless-stopped`.

## Backups

- pfSense: AutoConfigBackup enabled. The encryption password is stored outside this repo.
- Proxmox: VM backups are run manually with the Ansible playbook `backup.yml` (vzdump snapshot mode, zstd, to the external disk). The playbook verifies every archive and can shut the host down afterwards. Retention is set on the backup storage (see [screenshots/proxmox](../screenshots/proxmox)).
Screenshots: [Prometheus](../screenshots/prometheus), [Grafana](../screenshots/grafana)
