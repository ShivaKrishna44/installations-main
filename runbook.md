# Runbook: Deploy the Expense App and Monitor It with Prometheus

End-to-end steps to set up the domain, deploy the infrastructure with Terraform, install the exporters, and configure Prometheus to monitor a 3-tier app (frontend, backend, MySQL).

## Contents

1. [Phase 1: Domain and DNS setup](#phase-1-domain-and-dns-setup-hostinger--aws-route-53)
2. [Phase 2: AWS authentication and SCP check](#phase-2-aws-authentication-and-organizations-scp-check)
3. [Phase 3: Configure and apply Terraform](#phase-3-configure-and-apply-terraform-expense-infra-vm-v1-main)
4. [Phase 4: Install and start exporters on the application VMs](#phase-4-install-and-start-exporters-on-the-application-vms)
5. [Phase 5: Configure and start the Prometheus server](#phase-5-configure-and-start-the-prometheus-server)
6. [Troubleshooting cheat sheet](#troubleshooting-cheat-sheet)

## Architecture

| Tier | Exporters | Ports |
|---|---|---|
| Frontend (Nginx) | `node_exporter`, `nginx-prometheus-exporter` | 9100, 9113 |
| Backend (Node.js) | `node_exporter`, app `/metrics` | 9100, 9101 |
| MySQL VM | `node_exporter`, `mysqld_exporter` | 9100, 9104 |
| Prometheus server | Prometheus UI | 9090 |

Prometheus pulls (scrapes) every exporter every 15 seconds. The exporters do not push data.

---

## Phase 1: Domain and DNS setup (Hostinger + AWS Route 53)

### Step 1. Create the public hosted zone in AWS

1. Log into the AWS account where you will deploy the infrastructure (for example `111122223333`).
2. Go to **Route 53 > Hosted zones > Create hosted zone**.
3. Enter your domain (`vosukula.online`) and select **Public hosted zone**.
4. Copy the 4 AWS nameservers listed under the auto-generated `NS` record.

> **Rule:** Never edit or overwrite the `NS` or `SOA` records inside Route 53.

### Step 2. Update nameservers at your registrar (Hostinger)

1. Open **Hostinger > Domains > vosukula.online > DNS / Nameservers > Change Nameservers**.
2. Paste the 4 AWS Route 53 nameservers, without trailing dots.
3. Save.

### Step 3. Verify DNS delegation

```bash
nslookup -type=ns vosukula.online 8.8.8.8
```

The output should list the 4 AWS nameservers.

---

## Phase 2: AWS authentication and Organizations SCP check

### Step 1. Authenticate Terraform with an IAM user (never root keys)

1. In the target AWS account (`111122223333`), go to **IAM > Users > Create user** (`terraform-admin`).
2. Attach `AdministratorAccess`.
3. Go to **Security credentials > Create access key (CLI)**.
4. In Git Bash on your laptop, clear old sessions and configure the CLI:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE
aws configure
aws sts get-caller-identity   # Verify "Account" matches your Route 53 account
```

> Scope the permissions down once the setup works, never commit access keys, and delete the key when you no longer need it.

### Step 2. Check AWS Organizations Service Control Policies (SCPs)

If you see `UnauthorizedOperation ... explicit deny in a service control policy`:

1. Log into your Management Account and open **AWS Organizations > Policies > Service control policies**.
2. Open the blocking policy (for example `AdvancedModeRegionRestrictionSecurityControlPolicy`).
3. Do one of the following:
   - Deploy to an unrestricted region (for example `ap-southeast-2`).
   - Remove the region deny statement (for example `UsEast1Partitional`) if your AMI requires `us-east-1`.

---

## Phase 3: Configure and apply Terraform (`expense-infra-vm-v1-main`)

### Step 1. Get your laptop IP and the Prometheus server IP

On your laptop (Git Bash):

```bash
curl -s https://checkip.amazonaws.com
```

On your Prometheus server (over SSH):

```bash
curl -s https://checkip.amazonaws.com
```

> The original notes had the bare URL `https://amazonaws.com`, which looks cut off during copying. I used `https://checkip.amazonaws.com`, which returns your public IP. Confirm this is the URL you meant.

### Step 2. Update `terraform.tfvars` (watch the data types)

- `admin_cidr` must be a **single string** ending in `/32`.
- `monitoring_cidrs` must be a **list** `[ ... ]` with `/32` entries.

```hcl
aws_region       = "us-east-1"
domain_name      = "vosukula.online"
admin_cidr       = "<YOUR_LAPTOP_IP>/32"
monitoring_cidrs = ["<PROMETHEUS_PUBLIC_IP>/32"]
```

### Step 3. Deploy the infrastructure

```bash
terraform init
terraform apply -auto-approve
```

Save the outputs: `public_ips`, `private_ips`, `ssh_commands`, and the passwords.

> Run `terraform plan` and read it before using `-auto-approve`. Terraform outputs and passwords sit in state, so keep state private and never commit it.

---

## Phase 4: Install and start exporters on the application VMs

Before configuring Prometheus, make sure every exporter service is running and listening on each VM:

```bash
netstat -lntp      # or: ss -tulpn
```

### Step 1. Install `node_exporter` (port 9100) on frontend, backend and mysql

```bash
cd /opt
curl -fLO https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.12.1.linux-amd64.tar.gz
mv node_exporter-1.12.1.linux-amd64 /opt/node_exporter
rm -f node_exporter-1.12.1.linux-amd64.tar.gz

useradd --system --no-create-home --shell /sbin/nologin node_exporter 2>/dev/null || true
chown -R node_exporter:node_exporter /opt/node_exporter

cat << 'EOF' > /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/opt/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
```

> The original download URL was cut off to `https://github.com`. I rebuilt it from the standard GitHub release path. Check that release `v1.12.1` exists at https://github.com/prometheus/node_exporter/releases and use the current version if it does not.

### Step 2. Verify the tier-specific ports are listening

| VM | Run | Expected ports |
|---|---|---|
| Frontend | `netstat -lntp` | 9100 (`node_exporter`), 9113 (`nginx-prometheus-exporter`) |
| Backend | `netstat -lntp` | 9100 (`node_exporter`), 9101 (node app `/metrics`) |
| MySQL | `netstat -lntp` | 9100 (`node_exporter`), 9104 (`mysqld_exporter`) |

For MySQL, make sure the `mysqld_exporter` user exists and owns `/etc/mysqld_exporter/.my.cnf`:

```bash
useradd --system --no-create-home --shell /sbin/nologin mysqld_exporter
chown -R mysqld_exporter:mysqld_exporter /etc/mysqld_exporter
```

---

## Phase 5: Configure and start the Prometheus server

### Step 1. Create the `prometheus` system user and directories

```bash
useradd --no-create-home --shell /bin/false prometheus 2>/dev/null || true
mkdir -p /var/lib/prometheus /etc/prometheus
chown -R prometheus:prometheus /opt/prometheus /etc/prometheus /var/lib/prometheus
```

### Step 2. Configure `/etc/prometheus/prometheus.yml`

Use the `.yml` extension, not `.yaml`.

> **Critical rule:** If Prometheus is in a different VPC (for example `172.31.x.x`) than your app VMs (`10.0.x.x`), use the **public IPs** of frontend, backend and mysql. Do not use `*.vosukula.online` hostnames if Route 53 resolves them to the `10.0.x.x` private IPs.

```bash
cat << 'EOF' > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["<FRONTEND_PUBLIC_IP>:9100"]
        labels: { node: "frontend" }
      - targets: ["<BACKEND_PUBLIC_IP>:9100"]
        labels: { node: "backend" }
      - targets: ["<MYSQL_PUBLIC_IP>:9100"]
        labels: { node: "mysql" }

  - job_name: "nginx"
    static_configs:
      - targets: ["<FRONTEND_PUBLIC_IP>:9113"]

  - job_name: "backend"
    static_configs:
      - targets: ["<BACKEND_PUBLIC_IP>:9101"]

  - job_name: "mysql"
    static_configs:
      - targets: ["<MYSQL_PUBLIC_IP>:9104"]
EOF

chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### Step 3. Validate the config and start Prometheus

```bash
/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml
systemctl daemon-reload
systemctl enable --now prometheus
systemctl status prometheus --no-pager
```

### Step 4. Open port 9090 and access the UI

1. Make sure port 9090 is open in the Prometheus EC2 security group, ideally only to your own IP.
2. Open `http://<PROMETHEUS_PUBLIC_IP>:9090/targets` in your browser. Type `http://` explicitly and never use `https://`.
3. Confirm that all **7 of 7 targets are UP**: `prometheus` (1), `node` (3), `nginx` (1), `backend` (1), `mysql` (1).

---

## Troubleshooting cheat sheet

| Symptom / error | Exact cause | Instant fix |
|---|---|---|
| `status=217/USER` on `systemctl status` | The service user (`prometheus` or `mysqld_exporter`) was not created. | Run `useradd --system --no-create-home --shell /sbin/nologin <user>`, then `chown -R`. |
| `status=2` on `prometheus.service` | The file was saved as `.yaml` instead of `.yml`, or it has an unsupported YAML field (such as `scrape_native_histograms`). | Run `/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml` to see the exact line. |
| `ERR_SSL_PROTOCOL_ERROR` on `:9090` | The browser upgraded the URL to `https://`. | Type `http://<IP>:9090/targets` explicitly, or use an Incognito window. |
| `context deadline exceeded` (timeout) | The security group is missing the Prometheus IP, or `prometheus.yml` points at `10.0.x.x` private IPs from another VPC. | Check that `monitoring_cidrs = ["<PROM_IP>/32"]` is applied and that `prometheus.yml` uses public IPs. |
| `connect: connection refused` | The network and firewall are open, but the exporter service is not running on that port. | SSH into that VM, check `netstat -lntp`, and start and enable the exporter service. |
