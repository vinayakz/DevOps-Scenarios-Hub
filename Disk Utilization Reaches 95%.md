# Linux Production Scenario: Disk Utilization Reaches 95%

Disk space issues are among the most common production incidents that every Linux Administrator

A disk reaching **95% utilization** can cause:

- Application downtime
- Database failures
- Inability to write logs
- Container failures
- Kubernetes Pod evictions
- CI/CD pipeline failures

In this guide, you'll learn how to troubleshoot high disk utilization using a structured production approach.

---

# Scenario

## Problem Statement

It's Monday morning.

Monitoring alerts show:

```text
Disk Usage: 95%
```

Users begin reporting:

- Slow application performance
- Failed deployments
- Database write errors
- Unable to upload files

You SSH into the Linux server.

How would you troubleshoot and resolve the issue?

---

# Interview Question

A Linux production server has reached **95% disk utilization**.

What actions would you take?

---

# Step 1: Confirm the Issue

Always verify the alert before taking action.

## Check Disk Usage

```bash
df -h
#displays available and used disk space for all mounted filesystems
```

Example Output

```text
Filesystem      Size  Used Avail Use%
/dev/xvda1       50G   48G    2G  95%
```

---

## Check Inode Usage

Sometimes the disk isn't full, but all inodes are consumed.

```bash
df -ih
```

Example Output

```text
Filesystem      Inodes   IUsed   IFree Use%
/dev/xvda1        3.2M    3.2M      0 100%
```

If inode usage is 100%, no new files can be created even if disk space is available.

---

## Check System Load

```bash
uptime
```

Example

```text
load average: 2.15, 1.89, 1.65
```

High disk usage may also increase system load.

---

# Step 2: Identify What's Consuming Disk Space

Never delete files blindly.

Find the root cause first.

---

## Check Top-Level Directories

```bash
du -sh /*
```

Example

```text
18G /var
12G /opt
8G  /home
```

---

## Find Large Directories

```bash
du -h /var --max-depth=1 | sort -hr
```

Example

```text
15G /var/log
2G  /var/cache
500M /var/tmp
```

---

## Find Large Files

```bash
find / -xdev -type f -size +1G -exec ls -lh {} \;
```

Example

```text
5G application.log
3G backup.tar
2G core.dump
```

---

# Step 3: Perform Safe Cleanup

Once you've identified the cause, perform safe cleanup operations.

---

## Clean System Logs

Keep only the last seven days of logs.

```bash
journalctl --vacuum-time=7d
```

---

## Truncate Large Log Files

Instead of deleting active logs:

```bash
truncate -s 0 /var/log/application.log
```

---

## Clear Package Cache

### RHEL / CentOS

```bash
yum clean all
```

### Ubuntu / Debian

```bash
apt-get clean
```

---

## Remove Old Kernels

### RHEL

```bash
package-cleanup --oldkernels --count=2
```

### Ubuntu

```bash
apt autoremove -y
```

---

## Docker Cleanup

If Docker is installed:

```bash
docker system prune -af --volumes
```

This removes:

- Unused Images
- Stopped Containers
- Unused Networks
- Unused Volumes
- Build Cache

---

# Step 4: If Disk Usage Is Still High

Sometimes cleanup isn't enough.

---

## Move Large Files

Move infrequently accessed data to another disk.

```bash
rsync -av /path/to/large/data /mnt/otherdisk/
```

Then remove the original files after verification.

---

## Increase Disk Size (Cloud)

### AWS EC2 (EBS Volume)

Increase the EBS volume size.

After resizing:

```bash
growpart /dev/xvda 1

resize2fs /dev/xvda1
```

For XFS filesystems:

```bash
xfs_growfs /
```

---

## Add a New Disk

Identify the disk:

```bash
fdisk -l
```

Create filesystem:

```bash
mkfs.ext4 /dev/xvdb
```

Mount the disk:

```bash
mount /dev/xvdb /data
```

Persist after reboot:

```bash
echo "/dev/xvdb /data ext4 defaults 0 0" >> /etc/fstab
```

---

# Step 5: Prevent Future Incidents

Production support is not just about fixing today's issue.

Prevent it from happening again.

---

## Configure Log Rotation

Verify:

```bash
cat /etc/logrotate.conf
```

Or create custom logrotate rules.

---

## Enable Monitoring

Use monitoring tools like:

- CloudWatch
- Datadog
- Grafana
- Prometheus
- Zabbix
- Nagios

---

## Configure Alerts

Create alerts when disk usage reaches:

- 70% → Warning
- 80% → High
- 90% → Critical

---

## Schedule Cleanup Jobs

Automate cleanup using cron jobs.

Example:

```bash
crontab -e
```

```cron
0 2 * * 0 docker system prune -af
```

---

# Production Troubleshooting Flow

```text
Disk Alert (95%)
        │
        ▼
Verify Disk Usage
(df -h)
        │
        ▼
Check Inodes
(df -ih)
        │
        ▼
Find Large Directories
(du)
        │
        ▼
Find Large Files
(find)
        │
        ▼
Safe Cleanup
        │
        ▼
Free Space
        │
        ▼
Increase Disk (If Needed)
        │
        ▼
Monitor & Prevent
```

---

# Common Root Causes

| Root Cause | Result |
|------------|---------|
| Large Log Files | Disk Full |
| Docker Images | High Disk Usage |
| Stopped Containers | Storage Consumption |
| Old Backups | Disk Exhaustion |
| Package Cache | Wasted Space |
| Core Dump Files | Large File Growth |
| Old Kernel Versions | Unused Storage |
| Application Bug | Continuous Log Growth |

---

# Real Production Scenario

## Problem

An alert is received:

```text
Disk Usage: 95%
```

Application starts throwing errors:

```text
No space left on device
```

---

### Step 1

Check disk usage.

```bash
df -h
```

Output:

```text
95%
```

---

### Step 2

Find the largest directory.

```bash
du -sh /*
```

Output:

```text
30G /var
```

---

### Step 3

Inspect `/var`.

```bash
du -h /var --max-depth=1 | sort -hr
```

Output:

```text
25G /var/log
```

---

### Step 4

Find large files.

```bash
find /var/log -type f -size +500M
```

Output:

```text
application.log
```

Size:

```text
18 GB
```

---

### Step 5

Review logs.

Application was generating repeated database connection errors.

Every failure was logged.

---

### Root Cause

A database outage caused excessive logging.

The log file grew to **18 GB**, filling the disk.

---

### Resolution

- Truncate the log file.
- Fix the database issue.
- Configure logrotate.
- Add disk usage monitoring.

---

# Scenario-Based Interview Questions

## Scenario 1

### Question

Disk usage is 95%, but no large files are visible.

What would you check?

### Answer

Check inode usage.

```bash
df -ih
```

Millions of small files can exhaust inodes.

---

## Scenario 2

### Question

Docker server suddenly runs out of disk space.

What will you check?

### Answer

```bash
docker system df
```

Then clean up unused resources:

```bash
docker system prune -af --volumes
```

---

## Scenario 3

### Question

Disk usage remains high even after deleting a large log file.

Why?

### Answer

The application still has the deleted file open.

Restart the application or identify the open deleted file:

```bash
lsof | grep deleted
```

---

## Scenario 4

### Question

Disk is only 60% utilized, but the server shows:

```text
No space left on device
```

Why?

### Answer

Inodes are exhausted.

Verify using:

```bash
df -ih
```

---

## Scenario 5

### Question

A filesystem becomes full every night.

How would you prevent it?

### Answer

- Configure logrotate
- Schedule cleanup jobs
- Monitor disk growth
- Archive old backups
- Investigate the application generating excessive files

---

# Follow-Up Interview Questions

## What is the difference between `df` and `du`?

### `df`

Shows filesystem usage.

### `du`

Shows directory or file usage.

---

## Why should you check inode usage?

Because a filesystem can run out of inodes even when free disk space remains.

---

## What happens when the root filesystem reaches 100%?

Possible impacts:

- Applications cannot write data
- Logging stops
- Databases fail
- SSH sessions may become unstable
- Docker containers may crash

---

## How do you safely delete a large log file?

Instead of deleting:

```bash
rm application.log
```

Use:

```bash
truncate -s 0 application.log
```

This avoids issues with processes holding the file open.

---

## How do you identify deleted files still consuming disk space?

```bash
lsof | grep deleted
```

Restart the process if necessary.

---

# Interview Tip

Most candidates immediately delete files.

A Senior DevOps Engineer follows this workflow:

```text
Verify
    ↓
Analyze
    ↓
Identify Root Cause
    ↓
Perform Safe Cleanup
    ↓
Restore Service
    ↓
Prevent Recurrence
```

This structured approach demonstrates production troubleshooting expertise.

---

# Key Takeaway

**Never start by deleting files.**

Always follow this sequence:

1. Verify the issue
2. Identify what's consuming space
3. Perform safe cleanup
4. Resolve the root cause
5. Implement monitoring and preventive measures

Remember:

> **Identify → Analyze → Clean → Recover → Prevent**

A production-ready DevOps Engineer doesn't just free disk space—they determine **why** it filled up and ensure the issue doesn't happen again.
