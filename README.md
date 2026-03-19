# ansible-egress-auditor

Ansible role to deploy [egress-auditor](https://github.com/devops-works/egress-auditor), a tool that monitors outbound TCP connections and generates firewall rules or logs.

## Requirements

- `community.general` collection (for the `capabilities` module)
- Target must be a Linux system with netfilter support

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `egress_auditor_version` | `v0.1.1` | Release tag to download |
| `egress_auditor_arch` | auto-detected | Binary architecture (`amd64`, `arm64`, `386`). Auto-mapped from `ansible_architecture` |
| `egress_auditor_bin_path` | `/usr/local/bin/egress-auditor` | Install path for the binary |
| `egress_auditor_input` | `nflog` | Input plugin |
| `egress_auditor_input_options` | `["nflog:group:100"]` | List of `-I` flags passed to egress-auditor |
| `egress_auditor_output` | `logfmt` | Output plugin (`logfmt`, `iptables`, `loki`) |
| `egress_auditor_output_options` | `[]` | List of `-O` flags passed to egress-auditor |
| `egress_auditor_extra_args` | `""` | Extra CLI arguments (e.g. `-R` to hide process name) |
| `egress_auditor_user` | `root` | User to run the service as |
| `egress_auditor_setcap` | `true` | Set `cap_net_admin` capability on the binary |
| `egress_auditor_nflog_group` | `100` | NFLOG group ID for the nftables rule |
| `egress_auditor_nflog_manage_rules` | `true` | Deploy nftables rules to log new outbound TCP connections |
| `egress_auditor_logrotate` | `true` when output is `logfmt` | Deploy a logrotate configuration |
| `egress_auditor_logfile` | `/var/log/egress-auditor.log` | Log file path (used by logrotate) |
| `egress_auditor_logrotate_frequency` | `daily` | Logrotate frequency |
| `egress_auditor_logrotate_rotate` | `14` | Number of rotated files to keep |

## What the role does

1. Downloads the egress-auditor binary from GitHub releases
2. Sets `cap_net_admin` capability on the binary (when `egress_auditor_setcap` is true)
3. Deploys a systemd unit file and enables/starts the service
4. Deploys nftables rules in a dedicated `inet egress_auditor` table to log new outbound TCP connections via NFLOG (when using the `nflog` input plugin and `egress_auditor_nflog_manage_rules` is true). Rules are written to `/etc/nftables.d/egress-auditor.nft` and loaded with `nft -f`.
5. Deploys a logrotate configuration that sends SIGHUP to trigger log file reopen (when using the `logfmt` output plugin)

## Example Playbook

### Log outbound connections to a file

```yaml
- hosts: servers
  roles:
    - role: ansible-egress-auditor
      egress_auditor_output: logfmt
      egress_auditor_output_options:
        - "logfmt:file:/var/log/egress-auditor.log"
```

### Generate iptables rules

```yaml
- hosts: servers
  roles:
    - role: ansible-egress-auditor
      egress_auditor_output: iptables
      egress_auditor_output_options:
        - "iptables:verbose:2"
```

### Send to Loki

```yaml
- hosts: servers
  roles:
    - role: ansible-egress-auditor
      egress_auditor_output: loki
      egress_auditor_output_options:
        - "loki:url:http://loki.example.com:3100"
        - "loki:labels:host={{ inventory_hostname }}"
```

## License

MIT
