
# README - NGINX Load Balancer Disk Usage Troubleshooting Guide

## Overview

This guide helps troubleshoot and resolve high disk usage (99%) on an Ubuntu 24.04 VM (64GB storage) running an NGINX load balancer in a **cloud environment**.

---

## 1. Troubleshooting Steps

### a. Check Disk Usage

```bash
df -h
```

Identify the partition near full capacity.

---

### b. Find Large Files/Directories

```bash
sudo du -xh / | sort -rh | head -40
```

Focus on:

- `/var/log`, `/var/cache`, `/tmp`  
- NGINX-specific paths: `/var/log/nginx`, `/var/cache/nginx`

**Note:**  
Don’t delete log files directly if NGINX is running. Truncate safely:

```bash
> /var/log/nginx/access.log
> /var/log/nginx/error.log
```

Use `logrotate` to automate log management.

---

### c. Check Core Dumps

```bash
sudo find / -type f -name 'core*' -exec ls -lh {} \;
```

---

### d. Check Deleted But Open Files

```bash
sudo lsof | grep '(deleted)'
```

---

## 2. Common Causes & Recovery

### Cause 1: Excessive NGINX Log Growth

- **Scenario:** Heavy traffic/errors balloon logs.  
- **Impact:** Disk fills, degrading service.  
- **Recovery:**  
  - **Cloud:** Add an extra disk volume for backing up logs.  
  - Backup then truncate logs:  
    ```bash
    cp -rp <heavylog> <Disk_backup_path>
    echo "" > <heavylog>
    ```  
  - Validate config:  
    ```bash
    sudo nginx -t
    ```  
  - Reload gracefully:  
    ```bash
    sudo nginx -s reload
    ```  
  - Setup `logrotate` for automated rotation.

---

### Cause 2: Cache Accumulation

- **Scenario:** NGINX or system caches grow too large.  
- **Recovery:**  
  - **Avoid:** `sudo rm -rf /var/cache/nginx/*` (risk of system overload).  
  - **Safer:** Batch delete files:  
    ```bash
    sudo find /var/cache/nginx/ -type f -print0 | xargs -0 -n 1000 rm -f
    ```  
  - Or delete files older than 7 days:  
    ```bash
    sudo find /var/cache/nginx/ -type f -mtime +7 -delete
    ```  
  - Clean apt cache:  
    ```bash
    sudo apt-get clean
    ```  
  - Set cache size limits in NGINX config (`proxy_cache_path`).  
  - Schedule cleanup during low traffic periods.

---

### Cause 3: Core Dumps or Crash Files

- Delete core dumps:  
  ```bash
  sudo find / -name 'core*' -delete
  ```  
- Investigate crashes to fix root causes.

---

### Cause 4: Unexpected Files or Misconfigurations

- Inspect cron jobs or unexpected file writes.  
- Remove unnecessary files carefully.

---

## 3. Best Practices

- Backup important logs/cache before deletion.  
- Monitor disk usage and service health continuously.  
- Use `logrotate` for logs.  
- Always validate NGINX config before reload (`nginx -t`).  
- Prefer reload over restart to avoid downtime.  
- Schedule maintenance during low traffic.  
- Limit cache size in NGINX.

---

## 4. Example Safe Cache Cleanup Script

```bash
#!/bin/bash
find /var/cache/nginx/ -type f -print0 | xargs -0 -n 1000 rm -f
echo "Batch cache cleanup completed."
```

---

## Contact

Reach out to the DevOps team for support.

---
