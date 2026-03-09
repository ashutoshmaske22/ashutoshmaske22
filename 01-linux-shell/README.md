# 🐧 Linux & Shell Scripting

## Core Concepts

### File System Hierarchy
```
/           Root
├── /bin    Essential binaries (ls, cp, mv)
├── /etc    Config files
├── /home   User directories
├── /var    Variable data (logs, databases)
├── /tmp    Temporary files
├── /proc   Virtual FS (process/kernel info)
└── /sys    Virtual FS (hardware/driver info)
```

### Process Management
```bash
ps aux                   # All running processes
top / htop               # Real-time process viewer
kill -9 <PID>            # Force kill
nice -n 10 command       # Run with lower priority
nohup command &          # Run after logout
jobs / fg / bg           # Job control
```

### File Permissions
```
-rwxr-xr--  1 user group size date file
 ^^^  ^^^  ^^^
 owner group others

chmod 755 file    # rwxr-xr-x
chmod u+x file    # add execute for owner
chown user:group file
```

---

## 📋 Cheatsheet

### Text Processing (The Big 4)
```bash
# grep - search
grep -r "error" /var/log/       # recursive search
grep -i "warning" app.log       # case insensitive
grep -v "debug" app.log         # exclude matches
grep -c "ERROR" app.log         # count matches

# awk - column-based processing
awk '{print $1, $3}' file       # print col 1 and 3
awk -F: '{print $1}' /etc/passwd  # use : as delimiter
awk '/ERROR/ {print NR, $0}' log  # print line number with ERROR

# sed - stream editor
sed 's/old/new/g' file          # replace all
sed -i 's/old/new/g' file       # in-place edit
sed -n '10,20p' file            # print lines 10-20
sed '/pattern/d' file           # delete matching lines

# sort, uniq, wc
sort -k2 -n file                # sort by column 2 numerically
sort file | uniq -c | sort -rn  # frequency count
wc -l file                      # line count
```

### Networking
```bash
curl -I https://example.com          # HTTP headers only
curl -o file.zip https://url/file    # download file
wget -q --spider http://url          # check if URL exists
netstat -tuln                        # listening ports
ss -tulnp                            # modern netstat
nc -zv host 80                       # check port connectivity
dig domain.com                       # DNS lookup
nslookup domain.com
traceroute google.com
```

### Disk & System
```bash
df -h                    # disk usage
du -sh /var/log/*        # folder sizes
free -h                  # memory usage
vmstat 1 5               # CPU/mem stats every 1s, 5 times
iostat -x 1              # disk I/O stats
lsof -i :8080            # what's using port 8080
uptime                   # load average
```

### Cron Syntax
```
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-6, Sun=0)
# │ │ │ │ │
  * * * * * command

0 2 * * *    # Daily at 2 AM
*/15 * * * * # Every 15 minutes
0 9 * * 1   # Every Monday at 9 AM
```

---

## 🛠️ Hands-On Project: System Health Monitor

A shell script that monitors CPU, memory, disk, and top processes — sends alert if thresholds are breached.

**📁 File:** [`scripts/health_monitor.sh`](./scripts/health_monitor.sh)

```bash
#!/bin/bash
# ==============================================
# System Health Monitor
# Usage: ./health_monitor.sh [--alert-email you@example.com]
# ==============================================

set -euo pipefail

# ── Thresholds ──────────────────────────────
CPU_THRESHOLD=80
MEM_THRESHOLD=85
DISK_THRESHOLD=90
LOG_FILE="/var/log/health_monitor.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# ── Colors ──────────────────────────────────
RED='\033[0;31m'; GREEN='\033[0;32m'
YELLOW='\033[1;33m'; NC='\033[0m'

log() { echo "[$TIMESTAMP] $1" | tee -a "$LOG_FILE"; }
warn() { echo -e "${YELLOW}[WARN]${NC} $1"; log "WARN: $1"; }
alert() { echo -e "${RED}[ALERT]${NC} $1"; log "ALERT: $1"; }
ok() { echo -e "${GREEN}[OK]${NC} $1"; }

# ── CPU Check ───────────────────────────────
check_cpu() {
  CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
  CPU=${CPU%.*}
  if [ "$CPU" -gt "$CPU_THRESHOLD" ]; then
    alert "CPU usage is ${CPU}% (threshold: ${CPU_THRESHOLD}%)"
    echo "Top CPU consumers:"
    ps aux --sort=-%cpu | head -6 | awk '{print $1, $2, $3, $11}'
  else
    ok "CPU: ${CPU}%"
  fi
}

# ── Memory Check ────────────────────────────
check_memory() {
  MEM=$(free | awk '/Mem:/ {printf "%.0f", $3/$2*100}')
  if [ "$MEM" -gt "$MEM_THRESHOLD" ]; then
    alert "Memory usage is ${MEM}% (threshold: ${MEM_THRESHOLD}%)"
    echo "Top memory consumers:"
    ps aux --sort=-%mem | head -6 | awk '{print $1, $2, $4, $11}'
  else
    ok "Memory: ${MEM}%"
  fi
}

# ── Disk Check ──────────────────────────────
check_disk() {
  while IFS= read -r line; do
    USAGE=$(echo "$line" | awk '{print $5}' | tr -d '%')
    MOUNT=$(echo "$line" | awk '{print $6}')
    if [ "$USAGE" -gt "$DISK_THRESHOLD" ]; then
      alert "Disk ${MOUNT} is at ${USAGE}% (threshold: ${DISK_THRESHOLD}%)"
    else
      ok "Disk ${MOUNT}: ${USAGE}%"
    fi
  done < <(df -h | grep '^/dev/')
}

# ── Main ─────────────────────────────────────
echo "=============================="
echo " System Health Report"
echo " $TIMESTAMP"
echo "=============================="
check_cpu
check_memory
check_disk
echo "=============================="
log "Health check completed"
```

---

## 🎯 Key Interview Questions

1. **What is the difference between a hard link and a soft link?**  
   Hard links share the same inode. Soft (symbolic) links are pointers to the file path — break if the original is deleted.

2. **How do you find all files modified in the last 24 hours?**  
   `find /path -mtime -1 -type f`

3. **What does `set -euo pipefail` do in a script?**  
   `-e` exits on error, `-u` treats unset variables as errors, `-o pipefail` catches pipe failures.

4. **How do you schedule a job to run every 5 minutes?**  
   `*/5 * * * * /path/to/script.sh`

5. **Difference between `>` and `>>`?**  
   `>` overwrites, `>>` appends.
