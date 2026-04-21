# All-in-one Monitoring + Build Server (Ansible)

This Ansible project deploys the following on one standalone Linux server as native services:

- Grafana
- Prometheus
- Prometheus Alertmanager
- OpenTelemetry Collector
- Gitea
- MySQL for Gitea
- Nginx reverse proxy for path-based access

## Directory Layout

```text
monitoring_ansible/
├── group_vars/all.yml
├── inventory/hosts.ini
├── site.yml
├── templates/
└── README.md
```

## Assumptions

- Target OS: Ubuntu 22.04 or Ubuntu 24.04
- Access: SSH with sudo privileges
- Everything runs on a single host
- Services run directly on the host via packages, binaries, and systemd

## Update inventory

Edit `inventory/hosts.ini`:

```ini
[monitoring]
monitoring-server ansible_host=192.168.1.50 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

## Update variables

Edit `group_vars/all.yml` and change all passwords, hostname values, and email settings.

Important values:

- `monitoring_server_fqdn`
- `install_grafana`
- `grafana_repo_mode`
- `grafana_internal_repo_url`
- `grafana_gpg_key_src`
- `grafana_deb_src`
- `grafana_admin_password`
- `gitea_mysql_password`
- `gitea_admin_password`

## Run

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

## Internal Grafana Repo

If Grafana is available only from an internal apt mirror:

- set `install_grafana: true`
- leave `grafana_use_external_repo: false`
- set `grafana_internal_repo_url` to your internal Grafana apt base URL
- set `grafana_gpg_key_src` to a file in this repo, for example `files/grafana.asc`

## Local Grafana Deb

If you want to install Grafana from a local `.deb` in this repo:

- set `install_grafana: true`
- set `grafana_repo_mode: "deb"`
- set `grafana_deb_src` to a file in this repo, for example `files/grafana-enterprise_11.1.0_amd64.deb`

## After deployment

### URLs

- Grafana: `http://<server>:<nginx_public_port>/grafana/`
- Prometheus: `http://<server>:<nginx_public_port>/prometheus/`
- Alertmanager: `http://<server>:<nginx_public_port>/alertmanager/`
- Loki: `http://<server>:<nginx_public_port>/loki/ready`
- Gitea: `http://<server>:<nginx_public_port>/gitea/`

### Alertmanager notifications

- Email is controlled by `alertmanager_email_enabled`.
- Slack is controlled by `alertmanager_slack_enabled`.
- To enable Slack, set `alertmanager_slack_enabled: true` and provide `alertmanager_slack_webhook_url`.
- Optional Slack settings: `alertmanager_slack_channel`, `alertmanager_slack_username`, and `alertmanager_slack_send_resolved`.

### Temporary skip

- Set `install_gitea: false` in `group_vars/all.yml` to skip Gitea and the Gitea runner for a playbook run.
- Set `gitea_runner_force_reregister: true` for one playbook run when you want Ansible to delete `/var/lib/act_runner/.runner` and register the runner again.

### Dynamic Nginx reverse proxies

- Add extra path-based proxies with `extra_nginx_reverse_proxies` in `group_vars/all.yml`.
- Each entry supports:
  - `name`: label shown on the landing page
  - `path`: external path prefix such as `/myapp`
  - `upstream`: backend URL such as `http://127.0.0.1:8080`
  - `forwarded_prefix`: optional `X-Forwarded-Prefix` header value
  - `websocket`: optional `true` for websocket apps
  - `show_on_index`: optional `false` to hide it from the landing page
- Example:

```yaml
extra_nginx_reverse_proxies:
  - name: My App
    path: /myapp
    upstream: http://127.0.0.1:8080
    forwarded_prefix: /myapp
    show_on_index: true

  - name: Code Server
    path: /code
    upstream: http://127.0.0.1:8443
    websocket: true
```

## Notes

- This is a good starting point for a lab, POC, or internal standalone server.
- For production, you should add:
  - TLS certificates
  - backups
  - firewall rules
  - node exporter / mysql exporter / cadvisor for richer metrics
  - alert rules in Prometheus
