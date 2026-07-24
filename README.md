# 🛡️ Wazuh Lab Setup Guide

This document contains the complete setup process for the local Wazuh Lab used in the **Agentic SOC / Threat Hunting Agent** project.

---

# Objective

The purpose of this lab is to create a local Security Operations Center (SOC) environment where:

EvidenceForge
        ↓
Generate Security Logs
        ↓
Forward Logs to Wazuh
        ↓
Wazuh Parses & Enriches Alerts
        ↓
Distillation Layer
        ↓
Threat Hunting Agent

---

# Environment

## Host Machine

- OS: Windows 10
- RAM: 8 GB
- Storage:
  - C Drive
  - D Drive (Used for Virtual Machines)

---

## Virtualization

Software:
- Oracle VirtualBox

Virtual Machine:
- Name: `wazuh-lab`

Operating System:
- Ubuntu Server 26.04 LTS

ISO Used:

ubuntu-26.04-live-server-amd64.iso

---

# Network

Two adapters were configured.

Adapter 1

NAT

Provides internet access.

Adapter 2

Host-Only Adapter

Allows Windows host to communicate with Ubuntu VM through SSH.

Example Host-Only IP

192.168.56.102

---

# SSH Access

From Windows CMD

```bash
ssh username@192.168.56.102
```

Example

```bash
ssh _123@192.168.56.102
```

Enable SSH

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Verify

```bash
sudo systemctl status ssh
```

---

# Wazuh Installation

Installed:

- Wazuh Manager
- Wazuh API

Main Control Command

```bash
sudo /var/ossec/bin/wazuh-control
```

Start Wazuh

```bash
sudo /var/ossec/bin/wazuh-control start
```

Stop Wazuh

```bash
sudo /var/ossec/bin/wazuh-control stop
```

Restart

```bash
sudo /var/ossec/bin/wazuh-control restart
```

Status

```bash
sudo /var/ossec/bin/wazuh-control status
```

---

# Initial Problem

Symptoms

- Wazuh API failed to start.
- Most Wazuh services were not running.
- SSH occasionally disconnected.
- Ubuntu reported:

```
/ is using 99%
```

---

# Root Cause

Disk usage investigation

```bash
sudo du -xh /var --max-depth=1 | sort -h
```

Result

```
/var/ossec
≈12 GB
```

Further inspection

```bash
sudo du -xh /var/ossec --max-depth=2 | sort -h | tail -30
```

Found

```
/var/ossec/tmp
≈8.3 GB

/var/ossec/queue
≈2.9 GB
```

Inspection

```bash
sudo ls -lah /var/ossec/tmp
```

Large files

```
vd_1.0.0_vd_4.13.0.tar
≈7.9 GB

vd_1.0.0_vd_4.13.0.tar.xz
≈373 MB
```

Cause

The Vulnerability Detection module downloaded a very large vulnerability database, filling the root partition.

---

# Cleanup

Stop Wazuh

```bash
sudo /var/ossec/bin/wazuh-control stop
```

Delete temporary vulnerability database

```bash
sudo rm -f /var/ossec/tmp/vd_1.0.0_vd_4.13.0.tar

sudo rm -f /var/ossec/tmp/vd_1.0.0_vd_4.13.0.tar.xz
```

Verify

```bash
df -h
```

Space recovered

Approximately

11 GB

---

# Filesystem Expansion

Virtual Disk

40 GB

Ubuntu originally used

19 GB

The remaining

19 GB

was already available inside LVM but not allocated to the filesystem.

Check

```bash
lsblk

sudo pvs

sudo vgs

sudo lvs
```

Expand Logical Volume

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Resize Filesystem

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Verify

```bash
df -h
```

Result

Before

```
19 GB
```

After

```
38 GB
```

Available

```
22 GB Free
```

---

# Disable Vulnerability Detection

Reason

The Threat Hunting Agent project does not require local vulnerability scanning.

Disabling it prevents:

- Huge downloads
- High disk usage
- Additional CPU usage
- Additional RAM usage

Edit

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Change

```xml
<vulnerability-detection>
    <enabled>yes</enabled>
</vulnerability-detection>
```

to

```xml
<vulnerability-detection>
    <enabled>no</enabled>
</vulnerability-detection>
```

Verify

```bash
sudo grep -A5 -B2 "vulnerability" /var/ossec/etc/ossec.conf
```

Restart

```bash
sudo /var/ossec/bin/wazuh-control restart
```

---

# Healthy Wazuh Services

Expected

```
wazuh-modulesd
wazuh-monitord
wazuh-logcollector
wazuh-remoted
wazuh-analysisd
wazuh-syscheckd
wazuh-db
wazuh-authd
wazuh-apid
```

Some services may show

```
not running
```

This is normal if optional features are not configured.

Examples

- maild
- clusterd
- integratord
- csyslogd
- dbd
- agentlessd

---

# VirtualBox Snapshot

Snapshot Name

```
Wazuh Stable Base
```

Purpose

Provides a restore point before installing additional software.

Restore Process

Power Off VM

↓

Snapshots

↓

Select

```
Wazuh Stable Base
```

↓

Restore

---

# Weekly Health Check

Disk Usage

```bash
df -h
```

Memory

```bash
free -h
```

Wazuh Status

```bash
sudo /var/ossec/bin/wazuh-control status
```

SSH Status

```bash
sudo systemctl status ssh
```

---

# Current Lab Status

✅ Ubuntu Server Installed

✅ SSH Enabled

✅ Wazuh Manager Working

✅ Wazuh API Working

✅ Root Filesystem Expanded (38 GB)

✅ 22 GB Free Space

✅ Vulnerability Detection Disabled

✅ VirtualBox Snapshot Created

✅ Ready for EvidenceForge Integration

---

# Next Milestone

EvidenceForge
        ↓
Generate Logs
        ↓
Forward Logs to Wazuh
        ↓
Wazuh Alert Generation
        ↓
Distillation Layer
        ↓
Threat Intelligence Enrichment
        ↓
Threat Hunting Agent
