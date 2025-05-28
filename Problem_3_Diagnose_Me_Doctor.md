### Scenario

You are working as a DevOps Engineer on a cloud-based infrastructure where a virtual machine (VM), running Ubuntu 24.04, with 64GB of storage is under your management. Recently, your monitoring tools have reported that the VM is consistently running at 99% storage usage. This VM is responsible for only running one service - a NGINX load balancer as a traffic router for upstream services.


## 1. Troubleshooting Steps

### a. Confirm disk usage  
Check overall disk usage with:  
```bash
df -h
```
Identify which partition is at 99% usage

---

### b. Find large files/directories  
Identify large directories to pinpoint where space is consumed:  
```bash
sudo du -xh / | sort -rh | head -40
```
Focus on `/var/log`, `/var/cache`, `/tmp`, or NGINX-specific directories (`/var/log/nginx`, `/var/cache/nginx`). 
Check NGINX log files size. Look for excessive error or access logs that might have grown too large.

---

### c. Check for core dumps or crash files  
Core dumps or unexpected crash files can fill disk:  
```bash
sudo find / -type f -name 'core*' -exec ls -lh {} \;
```

---

### d. Check for stale processes or files open but deleted  
Sometimes, deleted files still held open by processes consume space:  
```bash
sudo lsof | grep '(deleted)'
```

---

## 2. Possible Causes, Impacts & Recovery

### Cause 1: Excessive NGINX log growth  
**Scenario:** Heavy traffic or errors cause access/error logs to balloon.  
**Impact:** Disk fills quickly, service may degrade due to lack of disk.  
**Recovery:**  
**Backup:** We are running in cloud so add a disk to backup log access nginx for debug in future.
```bash
cp -rp <heavylog> <Disk_backup_path>
echo "" > <heavylog>
```

### Cause 2: Cache accumulation  
**Scenario:** NGINX caching or system package caches grow uncontrollably.  
**Impact:** Disk fills, potentially slowing IO and service response.  
**Recovery:**  
- Clear NGINX cache (if enabled):  
  ```bash
  sudo rm -rf /var/cache/nginx/*
  ```  
- Clear apt cache:  
  ```bash
  sudo apt-get clean
  ```  
- Verify cache settings to limit size.

---

### Cause 3: Core dumps or crash files  
**Scenario:** NGINX or other services crash repeatedly, generating core dumps.  
**Impact:** Rapid disk consumption, possible service instability.  
**Recovery:**  
- Locate and remove core dumps:  
  ```bash
  sudo find / -name 'core*' -delete
  ```  
- Investigate and fix the crash root cause.  

---

### Cause 4: Unexpected files or misconfiguration  
**Scenario:** Other services, backups, or misconfigured cron jobs write large files.  
**Impact:** Disk fills unexpectedly.  
**Recovery:**  
- Inspect crontab jobs or unexpected files.  
- Remove unnecessary files.

---
