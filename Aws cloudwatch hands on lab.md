# 🔬 AWS CloudWatch Hands-On Lab
### A Production-Grade Monitoring Setup Guide
# Author: Sajal Jana

![AWS CloudWatch](https://img.shields.io/badge/AWS-CloudWatch-orange?style=for-the-badge&logo=amazon-aws)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-purple?style=for-the-badge&logo=ubuntu)
![Level](https://img.shields.io/badge/Level-Beginner%20to%20Intermediate-green?style=for-the-badge)

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Lab 1 - Enable Detailed Monitoring](#lab-1---enable-detailed-monitoring-on-ec2)
5. [Lab 2 - View EC2 Metrics](#lab-2---view-ec2-metrics-in-cloudwatch)
6. [Lab 3 - Create CloudWatch Alarm](#lab-3---create-a-cloudwatch-alarm)
7. [Lab 4 - Test the Alarm](#lab-4---test-the-alarm-cpu-spike)
8. [Lab 5 - Install CloudWatch Agent](#lab-5---install-cloudwatch-agent)
9. [Lab 6 - Configure Log Collection](#lab-6---configure-log-collection)
10. [Lab 7 - CloudWatch Log Insights](#lab-7---cloudwatch-log-insights)
11. [Lab 8 - CloudWatch Dashboard](#lab-8---cloudwatch-dashboard)
12. [Troubleshooting](#troubleshooting)
13. [Real-World Production Tips](#real-world-production-tips)
14. [Next Steps](#next-steps)

---

## Overview

This hands-on lab walks you through setting up **production-grade monitoring** on AWS using CloudWatch. By the end of this lab, you will have:

- EC2 metrics monitored with **1-minute granularity**
- **Automated alarms** that notify you via email
- **Centralized log collection** from your EC2 instance
- **Log Insights queries** for production troubleshooting
- A **real-time dashboard** showing CPU, Memory, and Disk metrics

> 💡 **Real-World Context:** This setup mirrors what companies like Netflix, Amazon, and Uber use for production monitoring. It forms the foundation of **Site Reliability Engineering (SRE)** practices.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **AWS Account** | Free tier account is sufficient |
| **EC2 Instance** | Ubuntu EC2 instance (running) |
| **SSH Access** | Ability to connect to EC2 via SSH or EC2 Instance Connect |
| **IAM Permissions** | Ability to create IAM roles and policies |

---

## Architecture

The following diagram shows what we will build in this lab:

```
┌─────────────────────────────────────────────────────────┐
│                    EC2 Instance (Ubuntu)                  │
│                                                           │
│  ┌─────────────────┐    ┌──────────────────────────────┐ │
│  │  stress tool    │    │   CloudWatch Agent           │ │
│  │  (CPU/Memory    │    │   - Collects /var/log/syslog │ │
│  │   spike test)   │    │   - Collects mem_used_%      │ │
│  └─────────────────┘    │   - Collects disk_used_%     │ │
│                         └──────────────┬─────────────── ┘ │
└──────────────────────────────────────── │ ────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────┐
│                     AWS CloudWatch                        │
│                                                           │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │   Metrics    │  │    Logs     │  │    Alarms        │ │
│  │  - CPU       │  │  Log Group: │  │  CPU > 80%       │ │
│  │  - Memory    │  │  Prod-EC2-  │  │  for 2 mins      │ │
│  │  - Disk      │  │  SysLog     │  │                  │ │
│  └──────────────┘  └─────────────┘  └────────┬─────────┘ │
│                                               │           │
│  ┌────────────────────────────────────────┐   │           │
│  │         Dashboard                      │   │           │
│  │  [CPU Graph] [Memory Graph]            │   │           │
│  │  [Disk Graph] [Alarm Status]           │   │           │
│  └────────────────────────────────────────┘   │           │
└───────────────────────────────────────────────│───────────┘
                                                │
                                                ▼
                                    ┌───────────────────┐
                                    │    SNS Topic       │
                                    │  EC2-CPU-Alert     │
                                    └─────────┬─────────┘
                                              │
                                              ▼
                                    ┌───────────────────┐
                                    │  📧 Email Alert    │
                                    │  (On-Call Engineer)│
                                    └───────────────────┘
```

---

## Lab 1 - Enable Detailed Monitoring on EC2

### Why This Matters in Production

By default, EC2 sends metrics to CloudWatch every **5 minutes**. In production, 5 minutes is too slow. If your server spikes in CPU or runs out of memory, you need to know **within 1 minute**.

> 💡 **Real-World Example:** During a flash sale, your e-commerce site gets a traffic spike. With basic monitoring, you'd only know your EC2 was overloaded 5 minutes later — by then, customers have already faced downtime. With detailed monitoring, your alarm fires within 1 minute and auto-scaling kicks in faster.

### Steps

1. Go to **EC2 Console** → **Instances**
2. Select your running instance (check the checkbox)
3. Click **Actions** → **Monitor and troubleshoot** → **Manage detailed monitoring**
4. Check the **Enable** checkbox
5. Click **Save**

```
EC2 Console
    └── Instances
        └── Select Instance ✓
            └── Actions
                └── Monitor and troubleshoot
                    └── Manage detailed monitoring
                        └── ✅ Enable → Save
```

### Result
- Metrics now collected every **1 minute** instead of 5 minutes
- More granular data for faster incident detection

---

## Lab 2 - View EC2 Metrics in CloudWatch

### Key Metrics to Monitor in Production

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| **CPUUtilization** | Is your server overloaded? | > 80% |
| **NetworkIn / NetworkOut** | Is traffic unusually high or low? | Depends on baseline |
| **StatusCheckFailed** | Is the instance healthy? | > 0 |
| **DiskReadOps / WriteOps** | Is disk I/O a bottleneck? | Depends on baseline |

> 💡 **Real-World Example:** A sudden drop in **NetworkIn** could mean your load balancer stopped routing traffic to the instance — a silent failure that affects users but doesn't crash the server.

### Steps

1. In AWS Console search bar, type **CloudWatch** and click on it
2. On the left sidebar, click **Metrics** → **All metrics**
3. Click **EC2** → **Per-Instance Metrics**
4. Find your **Instance ID** and locate **CPUUtilization**
5. Check the checkbox next to **CPUUtilization**

You will now see a **real-time graph** of your CPU usage! 📊

---

## Lab 3 - Create a CloudWatch Alarm

### Why This Matters in Production

Just **watching** a graph is not enough in production. You can't stare at dashboards 24/7. **Alarms automatically notify you** when something goes wrong — even at 3 AM.

> 💡 **Real-World Example:** In production, teams set a CPU alarm at **80% threshold**. If CPU crosses 80% for 2 consecutive minutes, it triggers an alert to the **on-call engineer** via email/SMS/PagerDuty. This is the foundation of any **SRE (Site Reliability Engineering)** practice.

### Step 1 - Create the Alarm

1. CloudWatch Console → **Alarms** → **All Alarms**
2. Click **Create Alarm**
3. Click **Select Metric** → **EC2** → **Per-Instance Metrics**
4. Select your instance's **CPUUtilization**
5. Click **Select Metric**

### Step 2 - Set Alarm Conditions

| Setting | Value | Reason |
|---|---|---|
| **Period** | 1 minute | We enabled detailed monitoring |
| **Threshold type** | Static | Fixed threshold |
| **Condition** | Greater than | Alert when CPU is high |
| **Threshold value** | **80** | Industry standard |
| **Datapoints to alarm** | **2 out of 2** | Avoids false alerts from brief spikes |

> ⚠️ **Alert Fatigue Warning:**
> - Too **low** threshold (e.g. 30%) = too many false alarms → engineers ignore alerts
> - Too **high** threshold (e.g. 95%) = notified too late → damage already done
> - **80% for 2 consecutive minutes** = industry best practice ✅

### Step 3 - Configure SNS Notification

| Setting | Value |
|---|---|
| **Alarm state trigger** | In Alarm |
| **SNS Topic Name** | EC2-CPU-Alert |
| **Email endpoint** | your-email@example.com |

> ⚠️ **Important:** After creating the SNS topic, AWS sends a **confirmation email**. You MUST click the confirmation link — otherwise notifications won't be delivered!

### Step 4 - Name the Alarm

Use production naming convention: `{Environment}-{Service}-{Metric}-{Threshold}`

```
Alarm Name:        Prod-EC2-CPUUtilization-80percent
Alarm Description: Fires when EC2 CPU exceeds 80% for 2 consecutive minutes
```

> 💡 **Pro Tip:** Always write a clear description. When an alarm fires at 3 AM, the on-call engineer needs to **instantly understand** what it means without digging through documentation.

---

## Lab 4 - Test the Alarm (CPU Spike)

### Why Test Alarms?

In production, you should **always test your alarms** after creating them. An untested alarm is as good as no alarm.

> 💡 **Real-World Practice:** Teams conduct **"Game Days"** — intentionally simulating failures to verify alerts fire, notifications reach the right people, and runbooks are accurate.

### Step 1 - Connect to EC2

```
EC2 Console → Instances → Select Instance → Connect
→ EC2 Instance Connect → Connect
```

Or connect via SSH:
```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

### Step 2 - Install the Stress Tool

Update package manager:
```bash
sudo apt update -y
```

Install stress tool:
```bash
sudo apt install stress -y
```

> 💡 **Real-World Use:** The `stress` tool is used by DevOps/SRE teams to load test servers before production launches and validate auto-scaling policies.

### Step 3 - Run CPU + Memory Stress Test

```bash
stress --cpu 2 --vm 2 --vm-bytes 256M --timeout 300
```

| Parameter | Meaning |
|---|---|
| `--cpu 2` | Spawn 2 workers to stress CPU cores |
| `--vm 2` | Spawn 2 workers to stress Memory |
| `--vm-bytes 256M` | Each worker consumes 256MB of RAM |
| `--timeout 300` | Run for 300 seconds (5 minutes) then stop |

### Expected Results

| Timeline | What Happens |
|---|---|
| **0-1 min** | CPU spikes to 90-100% |
| **2 min** | CloudWatch alarm changes: `OK` → `IN ALARM` |
| **2-3 min** | 📧 Email notification arrives |
| **After 5 min** | Stress stops, CPU returns to normal |
| **~7 min** | Alarm changes back: `IN ALARM` → `OK` |

### Verify in CloudWatch Console

1. CloudWatch Console → **Alarms** → **All Alarms**
2. Your alarm `Prod-EC2-CPUUtilization-80percent` should show **🔴 In alarm**

---

## Lab 5 - Install CloudWatch Agent

### Why This Matters in Production

By default, EC2 does **NOT** send these metrics to CloudWatch:
- ❌ Memory utilization
- ❌ Disk utilization
- ❌ Application logs

The **CloudWatch Agent** fills this gap and is essential for full observability.

### Step 1 - Download the Agent

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

### Step 2 - Install the Agent

```bash
sudo dpkg -i amazon-cloudwatch-agent.deb
```

> 💡 **Real-World Tip:** In production, this installation is automated using **Terraform** or **AWS Systems Manager (SSM)** so every new EC2 instance gets the CloudWatch agent automatically.

### Step 3 - Create IAM Role for CloudWatch

The agent needs **IAM permissions** to send data to CloudWatch.

**Create IAM Role:**
1. IAM Console → **Roles** → **Create Role**
2. Trusted entity: **AWS Service** → **EC2**
3. Attach policy: **`CloudWatchAgentServerPolicy`**
4. Role name: **`EC2-CloudWatch-Role`**

**Attach Role to EC2:**
1. EC2 Console → **Instances** → Select your instance
2. **Actions** → **Security** → **Modify IAM Role**
3. Select **`EC2-CloudWatch-Role`**
4. Click **Update IAM Role**

> 💡 **Real-World Tip:** In production, IAM Roles are attached automatically during EC2 launch via **Terraform** `iam_instance_profile` or **CloudFormation** `IamInstanceProfile` — no manual attachment needed.

---

## Lab 6 - Configure Log Collection

### Step 1 - Create Configuration File

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null << 'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "Prod-EC2-SysLog",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "disk_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF
```

### Configuration Explained

| Section | What It Collects | Why It Matters |
|---|---|---|
| `logs → /var/log/syslog` | System logs from EC2 | Captures crashes, errors, warnings |
| `log_group_name` | Groups logs in CloudWatch | Organizes logs by environment/service |
| `{instance_id}` | Unique stream per instance | Critical when you have multiple EC2s |
| `mem_used_percent` | Memory usage % | EC2 doesn't send RAM metrics by default! |
| `disk_used_percent` | Disk usage % | Catch disk full issues before they crash app |

### Step 2 - Start the Agent

```bash
# Load config and start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
    -s

# Enable auto-start on reboot
sudo systemctl enable amazon-cloudwatch-agent

# Start via systemd
sudo systemctl start amazon-cloudwatch-agent
```

### Step 3 - Verify Agent Status

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

**Expected Output:**
```json
{
  "status": "running",
  "starttime": "2026-03-26T09:12:15+00:00",
  "configstatus": "configured",
  "version": "1.300064.1b1344"
}
```

### Step 4 - Generate Test Log Entry

```bash
logger "CloudWatch Lab Test - Production Log Entry from EC2"
```

Verify it was written locally:
```bash
sudo tail -5 /var/log/syslog
```

---

## Lab 7 - CloudWatch Log Insights

### Why This Matters in Production

In production, you may have **millions of log lines** per day. You can't read them manually. **Log Insights** lets you run SQL-like queries to find patterns, errors, and anomalies instantly.

> 💡 **Real-World Example:** Imagine your application throws 50,000 log lines per hour. At 3 AM an alarm fires. Instead of scrolling through logs manually, you run a Log Insights query in seconds and find the exact error.

### How to Access

CloudWatch Console → **Log Insights** → Select log group **"Prod-EC2-SysLog"**

---

### Query 1 - View Recent Logs

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 25
```

| Part | Meaning |
|---|---|
| `fields @timestamp, @message` | Show timestamp and log message |
| `sort @timestamp desc` | Show newest logs first |
| `limit 25` | Show only 25 results |

---

### Query 2 - Search for Specific Messages

```sql
fields @timestamp, @message
| filter @message like /CloudWatch Lab Test/
| sort @timestamp desc
| limit 25
```

> 💡 **In Production you'd use:**
> ```sql
> filter @message like /ERROR/
> ```
> To instantly find all error messages across millions of logs!

---

### Query 3 - Count Log Events Over Time

```sql
fields @timestamp, @message
| stats count(*) as LogCount by bin(5m)
| sort @timestamp desc
```

| Part | Meaning |
|---|---|
| `stats count(*)` | Count total log events |
| `as LogCount` | Name the count column |
| `by bin(5m)` | Group into 5-minute buckets |

> 💡 **Real-World Use:** If your app normally generates 500 logs/minute and suddenly drops to 0 — that's a **silent failure**. Your app crashed but no error was thrown. This query catches that!

---

## Lab 8 - CloudWatch Dashboard

### Why This Matters in Production

In production, engineers need a **single pane of glass** to see the health of entire systems at a glance.

> 💡 **Real-World Example:** At companies like Netflix and Uber, every microservice has its own CloudWatch Dashboard displayed on **TV screens** in the office. On-call engineers check these during any incident.

### Create Dashboard

1. CloudWatch Console → **Dashboards** → **Create Dashboard**
2. Name: **`Prod-EC2-Dashboard`**
3. Click **Create Dashboard**

### Widget 1 - CPU Utilization

- Widget type: **Line**
- Metric: **EC2** → **Per-Instance Metrics** → **CPUUtilization**
- Period: **1 minute**
- Statistic: **Average**
- Title: **"EC2 CPU Utilization"**

### Widget 2 - Memory Utilization

- Widget type: **Line**
- Metric: **CWAgent** → **Per-Instance Metrics** → **mem_used_percent**
- Period: **1 minute**
- Statistic: **Average**
- Title: **"EC2 Memory Utilization"**

### Widget 3 - Disk Utilization

- Widget type: **Line**
- Metric: **CWAgent** → **Per-Instance Metrics** → **disk_used_percent**
- Period: **1 minute**
- Statistic: **Average**
- Title: **"EC2 Disk Utilization"**

### Widget 4 - Alarm Status

- Widget type: **Alarm status**
- Select: **`Prod-EC2-CPUUtilization-80percent`**
- Title: **"EC2 Alarm Status"**

### Production Thresholds Reference

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| **CPU** | < 70% | 70-80% | > 80% 🚨 |
| **Memory** | < 70% | 70-85% | > 85% 🚨 |
| **Disk** | < 70% | 70-85% | > 85% 🚨 |

---

## Troubleshooting

### Issue 1 - CloudWatch Agent Not Starting

**Symptom:**
```json
{
  "status": "stopped",
  "configstatus": "configured"
}
```

**Diagnose:**
```bash
sudo journalctl -u amazon-cloudwatch-agent -n 50 --no-pager
```

**Common Fix - Corrupted config file:**
```bash
# Remove corrupted file
sudo rm -f /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/file_amazon-cloudwatch-agent.json

# Stop agent
sudo systemctl stop amazon-cloudwatch-agent

# Reload config and restart
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
    -s
```

---

### Issue 2 - Log Group Not Appearing in CloudWatch

**Cause:** Missing IAM permissions on EC2 instance

**Fix:**
1. Create IAM Role with **`CloudWatchAgentServerPolicy`**
2. Attach role to EC2 instance
3. Restart CloudWatch Agent:
```bash
sudo systemctl restart amazon-cloudwatch-agent
```

---

### Issue 3 - Package Not Found (Ubuntu)

**Symptom:**
```
E: Unable to locate package amazon-cloudwatch-agent
```

**Fix:** Download directly from AWS:
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

---

## Real-World Production Tips

### 1. Naming Conventions

```
Alarms:     {Environment}-{Service}-{Metric}-{Threshold}
            Prod-EC2-CPUUtilization-80percent

Log Groups: {Environment}-{Service}-{LogType}
            Prod-EC2-SysLog

Dashboards: {Environment}-{Service}-Dashboard
            Prod-EC2-Dashboard

IAM Roles:  {Service}-{Purpose}-Role
            EC2-CloudWatch-Role
```

### 2. Production Notification Hierarchy

```
CloudWatch Alarm fires
        ↓
SNS Topic
        ↓
├── Email (immediate)
├── SMS (critical alerts)
├── PagerDuty (on-call rotation)
└── Slack Channel (team visibility)
```

### 3. Key Production Practices

| Practice | Why It Matters |
|---|---|
| **Detailed Monitoring** | 1-minute granularity catches issues faster |
| **2-datapoint alarm rule** | Avoids false alerts from brief spikes |
| **systemd enable** | Agent auto-starts after EC2 reboot |
| **Game Day Testing** | Verify alarms work before real incidents |
| **Log Insights queries** | Find root cause in millions of logs instantly |
| **Auto Refresh Dashboard** | Set to 10 seconds during active incidents |

### 4. What CloudWatch Agent Adds

| Metric | Default EC2 | With CloudWatch Agent |
|---|---|---|
| CPU | ✅ Available | ✅ Available |
| Network | ✅ Available | ✅ Available |
| **Memory** | ❌ Not available | ✅ Available |
| **Disk** | ❌ Not available | ✅ Available |
| **Application Logs** | ❌ Not available | ✅ Available |

---

## Full Production Flow Summary

```
EC2 CPU spiked to 100%
        ↓
CloudWatch detected CPU > 80% for 2 consecutive minutes
        ↓
Alarm state changed: OK → IN ALARM
        ↓
CloudWatch triggered SNS Topic "EC2-CPU-Alert"
        ↓
SNS delivered Email Notification to on-call engineer
        ↓
Engineer opens CloudWatch Dashboard
        ↓
Runs Log Insights query to find root cause
        ↓
Issue resolved, Alarm returns to OK
```

---

## Next Steps

### Level 2 — Intermediate AWS Monitoring

- **AWS Auto Scaling** → Automatically add EC2s when CPU is high
- **AWS CloudTrail** → Audit who did what in your AWS account
- **AWS Config** → Track infrastructure configuration changes
- **AWS Systems Manager** → Manage 1000s of EC2s without SSH

### Level 3 — Advanced Observability

- **AWS X-Ray** → Distributed tracing for microservices
- **CloudWatch Synthetics** → Monitor URLs automatically 24/7
- **CloudWatch Container Insights** → Monitor ECS/EKS containers
- **Amazon Managed Grafana** → Advanced visualization dashboards
- **AWS OpenSearch** → Advanced log analytics at scale

---

## Lab Completion Checklist

| Step | Task | Status |
|---|---|---|
| 1 | Enable Detailed Monitoring on EC2 | ✅ |
| 2 | View EC2 Metrics in CloudWatch | ✅ |
| 3 | Create CPU Alarm with SNS Notification | ✅ |
| 4 | Test Alarm with Stress Tool | ✅ |
| 5 | Verify Alarm Email Notification | ✅ |
| 6 | Install CloudWatch Agent | ✅ |
| 7 | Configure Log Collection | ✅ |
| 8 | Verify Logs in CloudWatch Log Groups | ✅ |
| 9 | Run Log Insights Queries | ✅ |
| 10 | Build CloudWatch Dashboard | ✅ |

---

## Author

This lab was created as a hands-on production-grade AWS CloudWatch tutorial.

> 🌟 **You are now capable of setting up production-grade monitoring on AWS!**

---

*Last updated: March 2026*
*AWS Region: Any*
*EC2 OS: Ubuntu 22.04 LTS*
