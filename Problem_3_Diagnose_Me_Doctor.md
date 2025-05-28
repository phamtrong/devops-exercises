
# README - NGINX Load Balancer Disk Usage Troubleshooting Guide

## Overview

This document provides troubleshooting steps and best practices to address the issue of high disk usage (99%) on an Ubuntu 24.04 VM running an NGINX load balancer. The VM has 64GB storage, runs in a **cloud environment**, and is dedicated to routing traffic to upstream services.

---

## 1. Troubleshooting Steps

### a. Confirm Disk Usage

Run the following command to check disk usage and identify the partition running out of space:

```bash
df -h
```

---

### b. Identify Large Files and Directories

To find directories consuming the most space, use:

```bash
sudo du -xh / | sort -rh | head -40
```

Focus on:

- `/var/log`
- `/var/cache`
- `/tmp`
- NGINX-specific directories like `/var/log/nginx` and `/var/cache/nginx`

Check the size of NGINX log files and look for excessive growth in access or error logs.

**Important:**  
Do **not** delete log files directly if NGINX is running and writing to them. Instead, truncate logs safely:

```bash
> /var/log/nginx/access.log
> /var/log/nginx/error.log
```

Consider configuring `logrotate` for automatic log rotation.

---

### c. Check for Core Dumps or Crash Files

Core dumps may consume large amounts of disk space:

```bash
sudo find / -type f -name 'core*' -exec ls -lh {} \;
```

---

### d. Check for Deleted but Open Files

Sometimes deleted files remain open by processes, still consuming space:

```bash
sudo lsof | grep '(deleted)'
```

---

## 2. Common Causes, Impacts & Recovery


### Cause 1: Excessive NGINX Log Growth

- **Scenario:** Heavy traffic or errors cause logs to balloon.  
- **Impact:** Disk fills quickly, causing degraded service or failures.  
- **Recovery:**  
  - Since this VM runs **in a cloud environment**, consider **adding an additional disk volume** to back up large logs for debugging and archival purposes without impacting the running system.  
  - Backup logs before truncating:  
    ```bash
    cp -rp <heavylog> <Disk_backup_path>
    echo "" > <heavylog>  # Safely truncate the log
    ```
  - Validate NGINX configuration before reload:  
    ```bash
    sudo nginx -t
    ```
  - Reload NGINX gracefully to avoid downtime:  
    ```bash
    sudo nginx -s reload
    ```
  - Implement `logrotate` for automatic and safe log management.

---

### Cause 2: Cache Accumulation

- **Scenario:** NGINX or system caches grow excessively.
- **Impact:** Disk fills, slowing IO and response times.
- **Recovery:**

  **Avoid running** `sudo rm -rf /var/cache/nginx/*` directly if the cache contains millions of files.

  Use safer batch deletion methods:

  ```bash
  sudo find /var/cache/nginx/ -type f -print0 | xargs -0 -n 1000 rm -f
  ```

  Or delete files older than 7 days:

  ```bash
  sudo find /var/cache/nginx/ -type f -mtime +7 -delete
  ```

  Clear apt cache:

  ```bash
  sudo apt-get clean
  ```

  Configure cache size limits in NGINX configuration (`proxy_cache_path`).

  Schedule cache cleanup during low traffic periods.

---

### Cause 3: Core Dumps or Crash Files

- **Scenario:** Repeated crashes create large core dumps.
- **Impact:** Disk fills rapidly; service may become unstable.
- **Recovery:**

  ```bash
  sudo find / -name 'core*' -delete
  ```

  Investigate and fix root cause of crashes.

---

### Cause 4: Unexpected Files or Misconfigurations

- **Scenario:** Other services or cron jobs write large files unexpectedly.
- **Impact:** Sudden disk usage spikes.
- **Recovery:**

  Inspect cron jobs and unexpected files; remove unnecessary files carefully.

---

## 3. Additional Recommendations

- Backup important logs and cache before deletion.
- Continuously monitor disk usage and service health.
- Use `logrotate` for log management.
- Always validate NGINX configuration before reload.
- Prefer `reload` over `restart` to avoid downtime.
- Schedule heavy cleanup during maintenance windows or low traffic.
- Configure cache size limits in NGINX to prevent uncontrolled growth.

---

## 4. Example Safe Cache Cleanup Script

```bash
#!/bin/bash

# Remove nginx cache files in batches of 1000 to avoid system overload
find /var/cache/nginx/ -type f -print0 | xargs -0 -n 1000 rm -f

echo "Batch cache cleanup completed."
```

---

## Contact

For further assistance or questions, please reach out to the DevOps team.

---

**End of Document**
