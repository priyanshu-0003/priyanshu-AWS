# Monitoring Stack: Datadog for a 3-Tier AWS E-Commerce Project

- This project provides a simple, script-based setup for monitoring a 3-tier AWS e-commerce application (Frontend, Backend, Database) using **Datadog**. The Datadog Agent is installed on both the **frontend (Nginx)** and **backend (Python app managed by PM2)** servers. It collects infrastructure metrics and application logs, which are then visualized in Datadog dashboards and used for alerting.

---

## Architecture Overview

```
                 Users
                   |
           [ AWS ALB / ELB ]
                   |
     +-------------+--------------+
     |   Frontend (EC2 - Nginx)   |   <-- Datadog Agent (service: frontend)
     +-------------+--------------+
                   |
     +-------------+--------------+
     |  Backend (EC2 - Python/PM2)|   <-- Datadog Agent (service: backend)
     +-------------+--------------+
                   |
     +-------------+--------------+
     |     Database (RDS)         |   <-- Datadog AWS Integration (CloudWatch)
     +----------------------------+
                   |
                   v
          Datadog (datadoghq.com)
     Dashboards | Logs | Monitors | Alerts
```

| Tier | AWS Resource | What we monitor | Datadog `service` tag | `role` host tag |
|------|--------------|-----------------|-----------------------|-----------------|
| Web / Frontend | EC2 + Nginx | Host metrics, Nginx access and error logs | `frontend` | `role:frontend` |
| App / Backend | EC2 + Python (PM2) | Host metrics, app stdout and error logs | `backend` | `role:backend` |
| Database | RDS | CPU, connections, storage (via AWS integration) | `database` | n/a |

---

## Components Explained

### 1. Datadog Agent (Backend Server)

**Purpose:**

The Datadog Agent is a lightweight program that runs on your server. It collects system metrics (CPU, memory, disk, network), processes, and logs, and sends them securely to Datadog. On the backend server, we also use it to collect the Python application logs written by PM2.

**How the setup works:**

- **Install:** The official install script downloads and installs Agent 7, and sets the API key, site, environment, and host tags.
- **Enable logs:** Log collection is switched on in `/etc/datadog-agent/datadog.yaml`.
- **Configure log files:** A custom config tells the Agent to tail the PM2 out and error logs.
- **Permissions:** The `dd-agent` user is given read access to the PM2 log folder.
- **Restart:** The Agent is restarted so the new config takes effect.

**Usage (run on the backend server as root or with sudo):**

```sh
DD_API_KEY=<YOUR_DATADOG_API_KEY> \
DD_SITE="datadoghq.com" \
DD_ENV=prod \
DD_HOST_TAGS="role:backend" \
bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"

# Enable the agent's expvar port and log collection
echo "expvar_port: 5002" >> /etc/datadog-agent/datadog.yaml
grep -n "^logs_enabled" /etc/datadog-agent/datadog.yaml || echo "logs_enabled: true" >> /etc/datadog-agent/datadog.yaml

# Tell the agent which backend log files to collect
mkdir -p /etc/datadog-agent/conf.d/pm2_backend.d
tee /etc/datadog-agent/conf.d/pm2_backend.d/conf.yaml > /dev/null <<'EOF'
logs:
  - type: file
    path: /root/.pm2/logs/backend-out.log
    service: backend
    source: python
  - type: file
    path: /root/.pm2/logs/backend-error.log
    service: backend
    source: python
EOF

# Allow the dd-agent user to read the PM2 logs
chmod o+x /root
chmod -R o+r /root/.pm2/logs

systemctl restart datadog-agent
```

> **Security note:** `chmod o+x /root` opens up the root home directory to every user on the server. For production, it is safer to run PM2 under a non-root user (for example `/home/ubuntu/.pm2/logs`) and add `dd-agent` to that user's group, or use `setfacl -m u:dd-agent:rx` only on the needed folders.

**What you get:**

- Real-time CPU, memory, disk, and network metrics for the backend host.
- Backend application logs (stdout and errors) searchable in Datadog Logs under `service:backend`.

---

### 2. Datadog Agent (Frontend Server)

**Purpose:**

The same Agent is installed on the frontend server, tagged as `role:frontend`. It tails the Nginx access and error logs so that web traffic, 4xx/5xx errors, and slow requests can be tracked.

**How the setup works:**

- **Install:** Same install script as the backend, with `DD_HOST_TAGS="role:frontend"`.
- **Nginx logs:** Access and error logs are collected with `service: frontend` and `source: nginx`.
- **Noise filter:** AWS load balancer health checks (`ELB-HealthChecker`) are excluded so they do not fill up your logs and your Datadog bill.

**Usage (run on the frontend server as root or with sudo):**

```sh
DD_API_KEY=<YOUR_DATADOG_API_KEY> \
DD_SITE="datadoghq.com" \
DD_ENV=prod \
DD_HOST_TAGS="role:frontend" \
bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"

grep -n "^logs_enabled" /etc/datadog-agent/datadog.yaml || echo "logs_enabled: true" >> /etc/datadog-agent/datadog.yaml

# Allow the dd-agent user to read Nginx logs
usermod -a -G adm dd-agent

mkdir -p /etc/datadog-agent/conf.d/nginx.d
tee /etc/datadog-agent/conf.d/nginx.d/conf.yaml > /dev/null <<'EOF'
logs:
  - type: file
    path: /var/log/nginx/access.log
    service: frontend
    source: nginx
    log_processing_rules:
      - type: exclude_at_match
        name: exclude_elb_healthchecks
        pattern: ELB-HealthChecker
  - type: file
    path: /var/log/nginx/error.log
    service: frontend
    source: nginx
EOF

systemctl restart datadog-agent
```

**What you get:**

- Real-time host metrics for the frontend server.
- Nginx access and error logs in Datadog Logs under `service:frontend`, with health-check noise removed.

---

### 3. Database Tier (RDS)

**Purpose:**

RDS does not allow installing an Agent, so it is monitored through the **Datadog AWS Integration**, which reads metrics from CloudWatch.

**Setup steps:**

1. In Datadog, go to **Integrations > AWS** and connect your AWS account (CloudFormation template or IAM role).
2. Enable the **RDS** and **EC2** and **ELB/ALB** services in the integration.
3. Within a few minutes, RDS metrics such as CPU, connections, free storage, and read/write latency appear in Datadog.

---

### 4. Alerting (Monitors)

**Purpose:**

Datadog Monitors watch your metrics and logs and notify you (email, Slack, PagerDuty, etc.) when a threshold is crossed.

**Recommended monitors for this project:**

| Monitor | Type | Condition |
|---------|------|-----------|
| High CPU on any host | Metric | `avg(last_5m)` of `system.cpu.user + system.cpu.system` above 85% |
| Low memory | Metric | `system.mem.pct_usable` below 0.15 |
| Disk almost full | Metric | `system.disk.in_use` above 0.85 |
| Frontend 5xx spike | Log | `service:frontend status:error` count above 10 in 5 minutes |
| Backend errors | Log | `service:backend status:error` count above 5 in 5 minutes |
| Agent not reporting | Metric (host) | `datadog.agent.up` missing for 5 minutes |
| RDS high CPU | Metric | `aws.rds.cpuutilization` above 80% |

---

## Step-by-Step Setup

1. **Create a Datadog account** and copy your **API key** from *Organization Settings > API Keys*. Note your **Datadog site** (for example `datadoghq.com`).
2. **Set up the backend server:** run the backend script from Section 1.
3. **Set up the frontend server:** run the frontend script from Section 2.
4. **Connect AWS (RDS, ALB):** follow Section 3.
5. **Verify the Agent on each server:**
   ```sh
   systemctl is-active datadog-agent
   datadog-agent status | grep -E "API Key|Last error"
   ```
   - Expected: `active`, an API Key line showing `API key ending with ...: valid`, and no `Last error`.
6. **Check the data in Datadog:**
   - **Infrastructure > Host Map / Infrastructure List:** both hosts should appear with tags `role:frontend` and `role:backend`.
   - **Logs:** filter by `service:frontend` and `service:backend`.
7. **Create dashboards and monitors** using the queries below.

---

## Prerequisites

- Unix-like OS on the servers (Ubuntu/Debian or Amazon Linux)
- `curl` installed and outbound internet access (HTTPS, port 443) to Datadog
- `sudo` or root access to install the Agent
- A Datadog account and API key
- AWS security group allowing outbound HTTPS from the EC2 instances

---

## Verifying Logs

After restarting the Agent, check that log files are being tailed:

```sh
sudo datadog-agent status
```

Look for the **Logs Agent** section. Each file should show `Status: OK`:

- `/root/.pm2/logs/backend-out.log` and `backend-error.log` (backend)
- `/var/log/nginx/access.log` and `error.log` (frontend)

Then open **Datadog > Logs** and filter:

| Filter | Shows |
|--------|-------|
| `service:frontend` | Nginx access and error logs |
| `service:backend` | Python application logs |
| `status:error` | Only errors across all services |
| `env:prod` | Production logs only |

> You may also see `auditd` and `ssm-agent-worker` logs in Datadog. These are system-level logs that Datadog picks up from the host (for example, security audit and AWS SSM agent logs). They are normal, but if they create too much noise, exclude them in **Logs > Configuration > Indexes > Exclusion Filters**.

---

## Troubleshooting & Tips

- **No logs appearing?** Make sure `logs_enabled: true` is set in `/etc/datadog-agent/datadog.yaml` and the Agent was restarted.
- **Permission denied on log files?** Check `sudo datadog-agent status` for the file error. Fix with `usermod -a -G adm dd-agent` (Nginx) or correct folder permissions (PM2 logs), then restart the Agent.
- **Agent not starting?** Run `journalctl -u datadog-agent -n 50` and check the config for YAML indentation mistakes.
- **API key invalid?** Confirm that `DD_SITE` matches the region where your Datadog account was created (`datadoghq.com`, `datadoghq.eu`, `us5.datadoghq.com`, and so on).
- **Do not commit your real API key to Git.** Keep `<YOUR_DATADOG_API_KEY>` as a placeholder in this repository and pass the real key through an environment variable, AWS Secrets Manager, or SSM Parameter Store.
- **Control log costs:** use `log_processing_rules` (like the ELB health check filter above) to exclude noisy logs before they leave the server.
- **Use tags consistently:** `env`, `service`, and `role` tags make dashboards and monitors much easier to filter.

### Generate test load (stress commands)

```sh
sudo apt-get install stress-ng -y   # Ubuntu/Debian
stress-ng --cpu 2 --timeout 300
```

Run this on a server and watch CPU rise in Datadog to confirm that your dashboards and monitors work.

---

## Datadog Dashboard Queries

Use these when creating a custom dashboard (**Dashboards > New Dashboard > Timeseries widget**).

### CPU

- Total CPU usage % per host
```
avg:system.cpu.user{env:prod} by {host} + avg:system.cpu.system{env:prod} by {host}
```
- CPU idle (for reference)
```
avg:system.cpu.idle{env:prod} by {host}
```
- Top 5 hosts by CPU
```
top(avg:system.cpu.user{env:prod} by {host}, 5, 'mean', 'desc')
```
- CPU by role (frontend vs backend)
```
avg:system.cpu.user{env:prod} by {role}
```

### Memory

- Memory usage % (usable memory left)
```
avg:system.mem.pct_usable{env:prod} by {host}
```
- Memory used (GB)
```
avg:system.mem.used{env:prod} by {host}
```

### Disk

- Disk usage %
```
avg:system.disk.in_use{env:prod} by {host,device}
```
- Disk read and write rate
```
avg:system.io.r_s{env:prod} by {host}
avg:system.io.w_s{env:prod} by {host}
```

### Network

- Incoming traffic
```
avg:system.net.bytes_rcvd{env:prod} by {host}
```
- Outgoing traffic
```
avg:system.net.bytes_sent{env:prod} by {host}
```

### Load balancer and database (AWS integration)

- ALB request count
```
sum:aws.applicationelb.request_count{*}.as_count()
```
- ALB 5xx errors
```
sum:aws.applicationelb.httpcode_target_5xx{*}.as_count()
```
- RDS CPU
```
avg:aws.rds.cpuutilization{*} by {dbinstanceidentifier}
```
- RDS connections
```
avg:aws.rds.database_connections{*} by {dbinstanceidentifier}
```
- RDS free storage
```
avg:aws.rds.free_storage_space{*} by {dbinstanceidentifier}
```

### Log Explorer Queries

```
service:frontend status:error
service:backend status:error
service:frontend @http.status_code:[500 TO 599]
service:backend "Traceback"
host:* env:prod -service:auditd
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Check Agent running | `systemctl is-active datadog-agent` |
| Check Agent health and API key | `datadog-agent status \| grep -E "API Key\|Last error"` |
| Restart Agent | `systemctl restart datadog-agent` |
| View Agent logs | `tail -f /var/log/datadog/agent.log` |
| Agent config file | `/etc/datadog-agent/datadog.yaml` |
| Custom log configs | `/etc/datadog-agent/conf.d/<name>.d/conf.yaml` |
