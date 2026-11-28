# Prometheus rules + Alertmanager (Slack + Email): manual install on the monitoring VM (v1)

## Layout

The same convention as Prometheus, the exporters and Grafana:

| What | Path |
| --- | --- |
| Binaries | `/opt/alertmanager-0.34.1`, symlink `/opt/alertmanager` (`alertmanager`, `amtool`) |
| Config | `/etc/alertmanager/alertmanager.yml` |
| Templates | `/etc/alertmanager/templates/*.tmpl` |
| Secrets (Slack webhook, SMTP app password) | `/etc/alertmanager/secrets/` (0640, never committed) |
| Data (silences, notification log) | `/var/lib/alertmanager` |
| Prometheus rules | `/etc/prometheus/rules/*.yml` |
| Port | 9093 |

Everything runs on the **monitoring VM**. The exporters, Grafana and the app VMs don't change.

---

## Step 0 — Prerequisites

### 0.1 Security group

Add an **inbound** rule to the security group **attached to the monitoring instance**: TCP **9093**, source **your current public IP/32**. Find your IP on the laptop with `curl -s ifconfig.me; echo`.

If the SG is managed in Terraform, add 9093 to the same ingress block as 3000, with the same `admin_cidr`, then `terraform apply`. Otherwise the next apply wipes the manual rule.

### 0.2 Outbound access

The VM needs outbound **443** (Slack) and **587** (SMTP). AWS blocks or throttles port **25** on EC2, so never use it.

### 0.3 Slack app with an Incoming Webhook

1. If you have no workspace, create a free one at slack.com. Create the channel **`#expense-alerts`**.
2. Go to **https://api.slack.com/apps** → **Create New App**.
3. Choose one of the two options:
   - **Pick features** (the "from scratch" option) → name `Expense Alertmanager`, pick the workspace → create. Then in the left sidebar: **Incoming Webhooks** → **On** → **Add New Webhook** → choose `#expense-alerts` → **Allow**.
   - **Paste a manifest** → pick the workspace → YAML:

     ```yaml
     display_information:
       name: Expense Alertmanager
       description: Prometheus Alertmanager notifications for the expense app
     oauth_config:
       scopes:
         bot:
           - incoming-webhook
     settings:
       org_deploy_enabled: false
       socket_mode_enabled: false
       token_rotation_enabled: false
     ```

     Then **Next** → **Create** → **Install to Workspace** → choose `#expense-alerts` → **Allow**. The URL is under **Incoming Webhooks**.
4. Copy the webhook URL (`https://hooks.slack.com/services/T…/B…/…`) and test it from the VM:

   ```bash
   curl -f -X POST -H 'Content-type: application/json' \
     --data '{"text":"Hello from the monitoring VM"}' \
     'https://hooks.slack.com/services/T.../B.../xxxx'
   ```

   It should print `ok`, and the message should appear in the channel.

Notes:
- **One webhook posts to one channel.** It always posts to the channel you chose when creating it, whatever `channel:` says in the config. A separate channel needs another webhook.
- **The URL is a credential.** Anyone who has it can post. If it leaks, remove the webhook and add a new one.

### 0.4 Gmail / Google Workspace app password

1. The sending account needs **2-Step Verification** turned on.
2. Go to **myaccount.google.com → App passwords**, while signed in **as the sending account**. Create one called "alertmanager".
3. Google shows `abcd efgh ijkl mnop`. Save it **without spaces**: 16 characters.

- **The normal account password will not work.** You get `535 Username and Password not accepted`.
- **A custom domain** (e.g. `info@yourdomain.com`) only works with `smtp.gmail.com` if the domain's mail is hosted on **Google Workspace**. Check with `dig +short MX yourdomain.com`: the answers should end in `google.com`. If they don't, use that mail provider's SMTP host instead.
- **Google Workspace admins** can block app passwords: admin.google.com → Security → Authentication.

### 0.5 Test the SMTP login without Alertmanager

Do this after Step 4, once the password file exists. It reads the password from the file, so it never shows on screen:

```bash
printf 'Subject: SMTP test\n\nHello from monitoring VM\n' > /tmp/mail.txt

curl -v --ssl-reqd --url 'smtp://smtp.gmail.com:587' \
  --user "sender@yourdomain.com:$(sudo cat /etc/alertmanager/secrets/smtp_password)" \
  --mail-from 'sender@yourdomain.com' \
  --mail-rcpt 'you@example.com' \
  -T /tmp/mail.txt 2>&1 | grep -E '^[<>] (2|3|5)|AUTH|curl:'
```

| Reply | Meaning |
| --- | --- |
| `235 2.7.0 Accepted` then `250 2.0.0 OK` | Credentials are good. The mail was sent |
| `535 5.7.8 … BadCredentials` | Wrong password, a non-app password, the wrong account, or app passwords are blocked |
| `534 5.7.9 … WebLoginRequired` | Usually a wrong or placeholder username, or Google wants a browser login. Check that you replaced the example values |

---

## Step 1 — Rules files

```bash
sudo mkdir -p /etc/prometheus/rules
```

### 1a. Recording rules

```bash
sudo tee /etc/prometheus/rules/recording.yml >/dev/null <<'EOF'
groups:
  - name: node_recording_rules
    interval: 30s
    rules:
      - record: instance:node_cpu_utilisation:ratio
        expr: 1 - avg by (instance, tier) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
      - record: instance:node_memory_utilisation:ratio
        expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
      - record: instance:node_filesystem_utilisation:ratio
        expr: 1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"})

  - name: backend_recording_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_request_duration_seconds_count{job="backend", route!="unmatched"}[5m]))
      - record: job:http_errors_5xx:rate5m
        expr: sum by (job) (rate(http_request_duration_seconds_count{job="backend", route!="unmatched", status_code=~"5.."}[5m]))
      - record: job:http_errors_5xx:ratio5m
        expr: job:http_errors_5xx:rate5m / job:http_requests:rate5m
      - record: job:http_request_duration_seconds:p99_5m
        expr: histogram_quantile(0.99, sum by (job, le) (rate(http_request_duration_seconds_bucket{job="backend", route!="unmatched"}[5m])))
EOF
```

- **Naming is `level:metric:operation`.** For example, `job:http_errors_5xx:ratio5m` tells you the aggregation level, the metric and the window.
- **`rate()` comes before `sum by()`,** always.
- **When there are no 5xx errors,** the error series doesn't exist, so the ratio has no value. For an alert that is correct: no series means no alert.

### 1b. Alerting rules

```bash
sudo tee /etc/prometheus/rules/alerts.yml >/dev/null <<'EOF'
groups:
  - name: availability_alerts
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} target {{ $labels.instance }} is down"
          description: "Prometheus could not scrape {{ $labels.instance }} for more than 1 minute."

      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL is down on {{ $labels.instance }}"
          description: "mysqld_exporter is up but cannot reach MySQL. UP is not healthy."

      - alert: NginxDown
        expr: nginx_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "nginx is down on {{ $labels.instance }}"
          description: "nginx-prometheus-exporter cannot read stub_status."

      - alert: SiteDown
        expr: probe_success{job=~"blackbox.*"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Public probe failing: {{ $labels.instance }}"
          description: "Blackbox probe has failed for 2 minutes. Users are affected."

      - alert: TLSCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry{job=~"blackbox.*"} - time()) / 86400 < 14
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "TLS cert for {{ $labels.instance }} expires in {{ $value | humanize }} days"
          description: "Renew the certificate."

  - name: host_alerts
    rules:
      - alert: HostHighCPU
        expr: instance:node_cpu_utilisation:ratio > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }} ({{ $labels.tier }})"
          description: "CPU is {{ $value | humanizePercentage }} for 5 minutes."

      - alert: HostHighMemory
        expr: instance:node_memory_utilisation:ratio > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }} ({{ $labels.tier }})"
          description: "Memory used is {{ $value | humanizePercentage }}."

      - alert: HostDiskAlmostFull
        expr: instance:node_filesystem_utilisation:ratio > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} is {{ $value | humanizePercentage }} full"
          description: "Free up space or grow the LV (lvextend -r)."

  - name: backend_alerts
    rules:
      - alert: BackendHighErrorRate
        expr: job:http_errors_5xx:ratio5m > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Backend 5xx ratio is {{ $value | humanizePercentage }}"
          description: "More than 5% of API requests failed for 5 minutes."

      - alert: BackendHighLatencyP99
        expr: job:http_request_duration_seconds:p99_5m > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Backend p99 latency is {{ $value | humanizeDuration }}"
          description: "p99 is above the 500ms threshold for 10 minutes."
EOF

sudo chown -R root:prometheus /etc/prometheus/rules
sudo chmod -R g+rX /etc/prometheus/rules
/opt/prometheus/promtool check rules /etc/prometheus/rules/*.yml
```

- **`for:`** is how long the condition must stay true before the alert moves from Pending to Firing. It is the main guard against flapping alerts.
- **Template trap:** `{{ $labels.tier }}` only renders if the expression keeps the `tier` label. That is why the CPU rule uses `by (instance, tier)`.
- **Check the label names against your setup:**
  - The blackbox job name must match `job=~"blackbox.*"`. Look it up in Status → Targets.
  - The backend `/metrics` must have the `route` and `status_code` labels.

---

## Step 2 — Prometheus config

Edit `/etc/prometheus/prometheus.yml`, the file the systemd unit uses. Confirm the path with `systemctl cat prometheus`. Add or merge these top-level blocks:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

Under `scrape_configs`, add a job so Alertmanager itself is monitored:

```yaml
  - job_name: alertmanager
    static_configs:
      - targets: ['localhost:9093']
```

Validate and reload. No restart is needed, because `--web.enable-lifecycle` is on:

```bash
/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml
curl -f -X POST http://localhost:9090/-/reload
sudo journalctl -u prometheus -n 30 --no-pager | grep -i -E 'rule|error|alertmanager'
```

Open `http://<monitoring-ip>:9090/alerts`: every rule should be listed as **Inactive**. Prometheus logs errors about `localhost:9093`, and the `alertmanager` target shows DOWN, until Step 7 is done. That is expected.

---

## Step 3 — Install Alertmanager

```bash
VERSION=0.34.1
cd /opt
sudo curl -fLO https://github.com/prometheus/alertmanager/releases/download/v${VERSION}/alertmanager-${VERSION}.linux-amd64.tar.gz
file alertmanager-${VERSION}.linux-amd64.tar.gz           # expect: gzip compressed data

sudo mkdir -p /opt/alertmanager-${VERSION}
sudo tar -xzf alertmanager-${VERSION}.linux-amd64.tar.gz -C /opt/alertmanager-${VERSION} --strip-components=1
sudo ln -sfn /opt/alertmanager-${VERSION} /opt/alertmanager

file /opt/alertmanager/alertmanager                        # expect: ELF 64-bit ... x86-64
/opt/alertmanager/alertmanager --version
/opt/alertmanager/amtool --version

sudo useradd --system --no-create-home --shell /sbin/nologin alertmanager
sudo mkdir -p /etc/alertmanager/{templates,secrets} /var/lib/alertmanager
sudo chown -R alertmanager:alertmanager /var/lib/alertmanager
```

---

## Step 4 — Secrets as files

```bash
# leading space = not saved in bash history; -n = no trailing newline
 echo -n 'https://hooks.slack.com/services/XXX/YYY/ZZZ' | sudo tee /etc/alertmanager/secrets/slack_webhook >/dev/null
 echo -n 'abcdefghijklmnop' | sudo tee /etc/alertmanager/secrets/smtp_password >/dev/null

sudo chown -R root:alertmanager /etc/alertmanager
sudo chmod 0750 /etc/alertmanager/secrets
sudo chmod 0640 /etc/alertmanager/secrets/*
```

Check the password file without printing it:

```bash
sudo wc -c /etc/alertmanager/secrets/smtp_password      # must be 16
```

- **Don't edit secret files with vim.** It adds a trailing newline.
- **Don't `cat` them where others can see the screen, or in screenshots.**
- **Restart Alertmanager after changing a secret file:** `sudo systemctl restart alertmanager`.

`alertmanager.yml` only references the file paths, so it is safe to commit later.

---

## Step 5 — Notification templates

```bash
sudo tee /etc/alertmanager/templates/expense.tmpl >/dev/null <<'EOF'
{{ define "slack.expense.title" -}}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
{{- end }}

{{ define "slack.expense.text" -}}
{{ range .Alerts -}}
*{{ .Annotations.summary }}*
• severity: `{{ .Labels.severity }}`  job: `{{ .Labels.job }}`  instance: `{{ .Labels.instance }}`
• {{ .Annotations.description }}
• started: {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}

{{ end -}}
{{- end }}

{{ define "email.expense.subject" -}}
[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }} ({{ .Alerts | len }} alerts)
{{- end }}
EOF
```

The email body uses Alertmanager's built-in HTML template. Only the subject is customised.

---

## Step 6 — alertmanager.yml (grouping, routing, email, Slack)

Replace the sender address, the `to:` address and the channel with your own.

```bash
sudo tee /etc/alertmanager/alertmanager.yml >/dev/null <<'EOF'
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'Expense Alerts <sender@yourdomain.com>'
  smtp_auth_username: 'sender@yourdomain.com'
  smtp_auth_password_file: '/etc/alertmanager/secrets/smtp_password'
  smtp_require_tls: true

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: slack-only
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers:
        - severity="critical"
      receiver: slack-and-email
    - matchers:
        - severity="warning"
      receiver: slack-only

inhibit_rules:
  - source_matchers:
      - alertname="MySQLDown"
    target_matchers:
      - alertname="BackendHighErrorRate"

receivers:
  - name: slack-only
    slack_configs:
      - api_url_file: '/etc/alertmanager/secrets/slack_webhook'
        channel: '#expense-alerts'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ template "slack.expense.title" . }}'
        text: '{{ template "slack.expense.text" . }}'

  - name: slack-and-email
    slack_configs:
      - api_url_file: '/etc/alertmanager/secrets/slack_webhook'
        channel: '#expense-alerts'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ template "slack.expense.title" . }}'
        text: '{{ template "slack.expense.text" . }}'
    email_configs:
      - to: 'you@example.com'
        send_resolved: true
        headers:
          Subject: '{{ template "email.expense.subject" . }}'
EOF

sudo chown root:alertmanager /etc/alertmanager/alertmanager.yml
sudo chmod 0640 /etc/alertmanager/alertmanager.yml
/opt/alertmanager/amtool check-config /etc/alertmanager/alertmanager.yml
```

`smtp_from` **must** wrap the address in angle brackets: `'Name <addr>'`.

What each setting teaches:

| Setting | Meaning | Analogy |
| --- | --- | --- |
| `group_by: [alertname, job]` | 3 targets down give **one** message listing 3, not 3 messages | One WhatsApp message with all the absentees, not one message per student |
| `group_wait: 30s` | Wait before the first message of a new group, so related alerts arrive together | |
| `group_interval: 5m` | Wait before notifying that a group **changed** (an alert was added or resolved) | |
| `repeat_interval: 4h` | Re-send a group that is still firing and unchanged. Firing is a **state**, so this is the only thing that repeats notifications | |
| `routes` + `matchers` | The first matching child route wins. Unmatched alerts go to the root `receiver` | A post office sorting by PIN code |
| `inhibit_rules` | MySQL down means backend 5xx is a **symptom**, so mute it and page on the cause | Don't complain "no water in the tap" when the main tank is empty |
| `send_resolved: true` | Also send a message when the alert clears | |

`matchers:` is the current syntax. `match:` / `match_re:` are deprecated.

---

## Step 7 — systemd unit

Use the monitoring VM's **real public IP** in `--web.external-url`. Slack and email messages link to this URL.

```bash
PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
echo $PUBLIC_IP

sudo tee /etc/systemd/system/alertmanager.service >/dev/null <<EOF
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/alerting/latest/alertmanager/
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/opt/alertmanager/alertmanager \\
  --config.file=/etc/alertmanager/alertmanager.yml \\
  --storage.path=/var/lib/alertmanager \\
  --web.listen-address=:9093 \\
  --web.external-url=http://${PUBLIC_IP}:9093 \\
  --cluster.listen-address=
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
systemctl cat alertmanager | grep external-url          # must show the real IP, not a placeholder
```

This heredoc is **unquoted** so that `${PUBLIC_IP}` gets filled in. That is why the line continuations are written `\\`.

- **`--cluster.listen-address=` (empty):** a single node needs no gossip cluster. This turns off port 9094 and its warnings.
- **The public IP changes when the instance is stopped and started.** Update the unit when that happens, or attach an Elastic IP.

Verify:

```bash
systemctl status alertmanager --no-pager
sudo ss -tlnp | grep 9093                           # want *:9093, not 127.0.0.1:9093
curl -f http://localhost:9093/-/healthy; echo       # OK
curl -s http://localhost:9090/api/v1/alertmanagers  # activeAlertmanagers lists localhost:9093
```

Open `http://<monitoring-public-ip>:9093`. Use **http**, not https. If it doesn't open, see **Troubleshooting: can't reach 9093**.

For later config changes: edit, then `amtool check-config`, then `curl -f -X POST http://localhost:9093/-/reload`. After changing **secret files** or the **systemd unit**, restart instead.

---

## Step 8 — Test the pipeline with a fake alert

This tests **only** Alertmanager routing and delivery, with no Prometheus rules involved.

```bash
# wait until it's up (right after a restart it refuses connections for a second)
curl -f http://localhost:9093/-/healthy && echo

/opt/alertmanager/amtool alert add PipelineTest severity=critical job=manual instance=test \
  --annotation='summary="Alert pipeline test"' \
  --annotation='description="Slack + email delivery works"' \
  --alertmanager.url=http://localhost:9093
```

- **`severity=critical`:** after about 30 s (`group_wait`) it arrives in Slack **and** email. Check spam too. It resolves by itself after about 5 min (`resolve_timeout`).
- **Again with `severity=warning`:** it arrives in **Slack only**.
- **Use a new alert name for each retry** (`PipelineTest2`, `PipelineTest3`…). Otherwise it joins the old group and waits behind it.

Dry-run the routing, without sending anything:

```bash
/opt/alertmanager/amtool config routes test --config.file=/etc/alertmanager/alertmanager.yml severity=critical
/opt/alertmanager/amtool config routes show --config.file=/etc/alertmanager/alertmanager.yml
```

If nothing arrives:

```bash
sudo journalctl -u alertmanager --since "5 min ago" --no-pager | grep -i -E 'notify|error|smtp|slack|email'
curl -s http://localhost:9093/metrics | grep -E 'alertmanager_notifications(_failed)?_total\{.*integration="(email|slack)"'
```

Each integration fails and retries **independently**. For example, Slack can succeed while email fails. Check the `integration=` field in the log line. Also check the timestamp, because an old process may still be retrying an old alert.

---

## Step 9 — Break drills (the real lesson)

Keep three things open side by side: `:9090/alerts`, `:9093`, and the Slack channel.

| # | Break | Expect |
| --- | --- | --- |
| 1 | `sudo systemctl stop node_exporter` on the backend VM | `TargetDown`: Inactive → **Pending** (yellow, 1m) → **Firing** (red) → Slack + email. Start it again → **RESOLVED** |
| 2 | Stop node_exporter on **two** VMs within 30 s | **One** Slack message listing 2 alerts (grouping) |
| 3 | `sudo systemctl stop mysqld` on the MySQL VM, with load running | `MySQLDown` fires. `BackendHighErrorRate` fires in Prometheus but is **suppressed** in Alertmanager (inhibition) |
| 4 | Stop `mysqld_exporter` instead of MySQL | `TargetDown` for job `mysql`, **not** `MySQLDown`: a dead exporter means `mysql_up` doesn't exist at all |
| 5 | Maintenance silence | `amtool silence add alertname=TargetDown instance=~"backend.*" --duration=30m --comment="patching" --alertmanager.url=http://localhost:9093`, then repeat drill 1: no message. `amtool silence query`, `amtool silence expire <id>` |
| 6 | Burn CPU: run `yes > /dev/null &` twice on the t3.micro | `HostHighCPU` (warning) after 5 min → Slack only, no email. Stop it with `kill %1 %2` |

Drill 3 versus drill 4 is the "UP ≠ healthy" lesson again, this time applied to alerts.

---

## Troubleshooting: can't reach 9093 in the browser

1. **On the VM:**

   ```bash
   sudo ss -tlnp | grep 9093                   # *:9093
   curl -s http://localhost:9093/-/healthy; echo
   curl -s http://<private-ip>:9093/-/healthy; echo
   sudo systemctl is-active firewalld          # if active: firewall-cmd --permanent --add-port=9093/tcp && firewall-cmd --reload
   ```

2. **From the laptop:**

   ```bash
   curl -s ifconfig.me; echo
   curl -v --max-time 10 http://<monitoring-public-ip>:9093/-/healthy
   ```

| Laptop result | Cause | Fix |
| --- | --- | --- |
| Hangs, then times out | The security group or NACL drops it | Add an inbound 9093 rule on the SG **attached to the monitoring instance**, with source = your **current** IP/32 |
| `Connection refused` | Reaches the VM, but nothing accepts it | firewalld, or Alertmanager is not running |
| `OK` | The network is fine | Type the full `http://…:9093` in the browser, because some browsers force https |

**Seen on 2026-11-27:** the laptop's public IP had changed after moving to a different network, so the SG rule's source no longer matched. The `curl` from the laptop timed out.

---

## Gotchas (all hit during the first install)

| Symptom | Cause | Fix |
| --- | --- | --- |
| `535 5.7.8 Username and Password not accepted` | The normal password was used, spaces were left in the app password, there is a trailing newline, the app password was made in a different account, or Alertmanager still has the old password loaded | Use a 16-character app password with no spaces. Check with `wc -c` (16). Write it with `echo -n`. **Restart** Alertmanager |
| `534 5.7.9 WebLoginRequired` in the curl SMTP test | The example `yourname@gmail.com` / password placeholders were still in the command | Use the real username and read the password from the file (Step 0.5) |
| Custom-domain sender rejected by `smtp.gmail.com` | The domain's mail is not on Google Workspace | Check `dig +short MX <domain>`. Use the real provider's SMTP host |
| Login works but the send fails, or there is no From name | `smtp_from` has no `<…>` around the address | `'Expense Alerts <sender@domain>'` |
| `dial tcp …:25: i/o timeout` | Port 25 is blocked on EC2 | Use `:587` with `smtp_require_tls: true` |
| `amtool: … connect: connection refused` right after a restart | Alertmanager is not listening yet | Wait for `curl -f localhost:9093/-/healthy`, then send |
| amtool warning "moving to a new parser … incompatible" | An annotation value contains spaces and isn't quoted | `--annotation='summary="text with spaces"'` |
| Old `Notify attempt failed` lines keep appearing | An old alert group is still retrying, or the line is older than the restart | Look at the timestamp. Restart. Use a new alert name for the test |
| Slack messages and emails link to `http://<monitoring-public-ip>:9093` | A placeholder was left in `--web.external-url` | `sed` in the real IP, then `daemon-reload` + `restart` |
| Slack `invalid_token` / `no_service` / 404 | The webhook was revoked, or the app was removed | Create a new webhook and update the secret file, then restart |
| `permission denied` on secrets | The file is not readable by group `alertmanager` | `chown root:alertmanager`, `chmod 0640` |
| Alert stuck in Pending | `for:` has not elapsed yet, or the condition is flapping | Wait, or check the expression in the Graph tab |
| Alert Firing in Prometheus but absent in Alertmanager | No `alerting:` block, or Prometheus not reloaded | Step 2. Check `/api/v1/alertmanagers` |
| In Alertmanager but no message | `group_wait`, an inhibition, a silence, or the wrong route | Run `amtool config routes test`. Check the Silences tab |
| The same alert repeats every few hours | `repeat_interval` is working as designed | Raise it, or fix the problem |
| No RESOLVED message | `send_resolved` is not set | Set `send_resolved: true` on each receiver config |
| `{{ $labels.tier }}` is empty | The expression aggregated the label away | Keep it in `by (...)` |
| Rules don't appear | YAML error, or the wrong `prometheus.yml` was edited | Run `promtool check config`. Check the path with `systemctl cat prometheus` |

## Security checklist

- [ ] Secret files are 0640 `root:alertmanager` and not in git.
- [ ] The app password was rotated if it was ever shown on screen, pasted in chat or put in a screenshot.
- [ ] The SG allows 9093 from your IP only, never `0.0.0.0/0`. Alertmanager has no login, and anyone who reaches it can create silences.