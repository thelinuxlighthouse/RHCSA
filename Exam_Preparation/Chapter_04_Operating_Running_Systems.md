# Chapter 4: Operating Running Systems

## Learning Objectives

By the end of this chapter, you will be able to:

- Boot, reboot, and shut down a system normally using systemd
- Boot systems into different targets manually and interrupt the boot process
- Identify CPU/memory intensive processes and terminate them appropriately
- Adjust process scheduling using `nice` and `renice`
- Manage tuning profiles for system performance
- Locate and interpret system log files and systemd journals
- Preserve system journals for forensic analysis
- Start, stop, and check the status of network services
- Securely transfer files between systems using `scp`, `sftp`, and `rsync`

---

## Concepts and Theory

### Systemd Boot System

RHEL 10 uses `systemd` as its init system and boot manager. Systemd replaces traditional SysV init with parallel service startup, dependency management, and target-based boot states.

**Systemd Targets:**

Targets are unit groups that define system boot states:

- **multi-user.target**: Full system without graphical interface (default for servers)
- **graphical.target**: Full system with graphical interface (default for workstations)
- **rescue.target**: Rescue mode for recovery
- **emergency.target**: Single-user mode for emergency access
- **bluetooth.target**, `network.target`, etc.: Partial boot states

**Boot Process:**

1. BIOS/UEFI performs hardware initialization
2. Bootloader (GRUB2) loads the kernel and initial RAM disk
3. systemd starts, mounts filesystems, and switches to the default target
4. Services start according to their dependencies
5. System reaches the target state

### Process Management

Processes are independent instances of running programs. Each process has:

- **PID (Process ID)**: Unique identifier (1-4194304 on 64-bit systems)
- **PPID (Parent Process ID)**: ID of the parent process
- **State**: Running, sleeping, stopped, zombie, etc.
- **CPU and Memory Usage**: Resource consumption metrics
- **Priority**: Scheduling priority (nice value from -20 to 19)

**Process States:**

- **R (Running)**: CPU time is being consumed
- **S (Sleeping)**: Process is waiting for events
- **D (Uninterruptible Sleep)**: Waiting on I/O
- **Z (Zombie)**: Process has ended but not reaped
- **T (Stopped)**: Process is stopped (signal or job control)

### Process Priority and Scheduling

Linux uses a Completely Fair Scheduler (CFS) that assigns CPU time based on priority. The `nice` value ranges from -20 (highest priority) to 19 (lowest priority), with 0 being the default.

**Priority Inheritance:**

- Parent process inherits nice value from its parent
- Commands can change their nice value at startup
- Running processes can change their nice value with `renice`

### Tuning Profiles

RHEL 10 uses tuning profiles to apply system-wide performance settings:

- **performance**: Optimized for CPU performance
- **balanced**: Balanced CPU and I/O performance (default)
- **power-savings**: Optimized for power efficiency
- **throughput**: Optimized for disk throughput

Profiles adjust kernel parameters for CPU frequency scaling, I/O scheduling, and memory management.

### System Logging

RHEL 10 uses two logging systems:

**Journal (systemd-journald):**

- Binary log files stored in `/var/log/journal/`
- Queryable via `journalctl`
- Contains logs from systemd and applications using libsystemd-journal
- Preserved and rotated automatically

**Syslog:**

- Text-based logs in `/var/log/`
- Traditional format for compatibility
- Includes `/var/log/messages`, `/var/log/secure`, `/var/log/audit/`

### Secure File Transfer

Secure file transfer protocols protect data in transit:

- **SCP (Secure Copy)**: Uses SSH for secure file copying
- **SFTP (SSH File Transfer Protocol)**: Interactive file management over SSH
- **Rsync**: Synchronized copying with delta transfers, can use SSH for security

---

## How It Works Internally

### Systemd Boot Sequence

When systemd starts, it:

1. **Pivot Root**: Switches from initramfs to real root filesystem
2. **Mount Filesystems**: Mounts root, /etc, /var, and other filesystems
3. **Start Early Services**: Initializes hardware, network, and device managers
4. **Parse Unit Files**: Loads all unit files from `/etc/systemd/system/` and `/lib/systemd/system/`
5. **Resolve Dependencies**: Determines startup order based on `After=` and `Requires=` directives
6. **Start Services**: Launches services in parallel where possible
7. **Switch Target**: Changes to the default target state

### How Process Scheduling Works

The CFS scheduler maintains a runqueue for each CPU. Each process has a virtual runtime (`vruntime`) that tracks CPU time consumed. Processes with lower `vruntime` get scheduled first.

The nice value affects the scheduling weight:
- Lower nice values = higher weight = more CPU time
- Higher nice values = lower weight = less CPU time

When a process voluntarily yields CPU (e.g., waiting on I/O), the scheduler recalculates vruntime.

### How Systemd Targets Work

Targets are symlink groups in `/lib/systemd/system/` that point to unit files. When systemd switches to a target:

1. It stops all units not in the target
2. It starts all units in the target
3. Dependencies ensure proper startup order

Example: `graphical.target` includes `multi-user.target` plus display managers and GUI services.

### How Journaling Works

systemd-journald collects logs from:

- **System services**: Logs written via journald's API
- **Kernel**: Messages from printk
- **Userspace applications**: Applications using libsystemd-journal or journal-remote

Logs are stored in binary format in `/var/log/journal/` with:

- **Timestamps**: Precise to microsecond
- **Metadata**: PID, UID, hostname, machine ID
- **Fields**: Key-value pairs from syslog-style logging

### How Secure File Transfer Works

**SCP/SFTP:**

1. SSH connection is established with public key authentication
2. Session is encrypted using SSH encryption (AES, ChaCha20)
3. File transfer occurs over encrypted channel
4. Integrity is verified using message authentication codes

**Rsync with SSH:**

1. SSH tunnel is established
2. rsync sends file listing from remote
3. rsync sends only changed blocks (delta transfer)
4. Files are received and verified

---

## Commands and Administration Tasks

### Booting and Shutting Down Systems

#### Normal Shutdown

```bash
# Shutdown system
sudo systemctl poweroff

# Reboot system
sudo systemctl reboot

# Halt (power off without ACPI)
sudo systemctl halt

# Stop all services and halt
sudo systemctl stop
```

#### Scheduled Shutdown

```bash
# Schedule shutdown in 10 minutes
sudo wall "System will be shutting down in 10 minutes"
sudo systemctl poweroff

# Add shutdown to calendar
sudo systemctl set-default shutdown.target
```

#### Graceful Shutdown Options

```bash
# Shutdown with delay
sudo systemctl poweroff -i 5    # 5 minute delay

# Reboot and sync filesystems
sync
sudo systemctl reboot
```

#### Emergency Shutdown

```bash
# Force shutdown (immediate)
sudo init 0
sudo systemctl reboot --force

# Power off without waiting for services
sudo systemctl poweroff --force
```

### Booting into Different Targets

#### Changing Default Target

```bash
# View current default target
systemctl get-default

# Change default target
sudo systemctl set-default multi-user.target   # CLI mode
sudo systemctl set-default graphical.target    # GUI mode

# Reboot to apply changes
sudo reboot
```

#### Booting into Specific Target

```bash
# Reboot into rescue mode
sudo systemctl reboot --target=rescue.target

# Reboot into emergency mode
sudo systemctl reboot --target=emergency.target

# Reboot into maintenance mode
sudo systemctl reboot --target=maintenance.target
```

#### Booting into Single User Mode

```bash
# Using systemctl
sudo systemctl isolate single-user.target

# Using init
sudo init 1
```

### Interrupting the Boot Process

#### GRUB Boot Menu

```bash
# Press Ctrl+x or Ctrl+c to access GRUB shell
# Use arrow keys to select kernel, Enter to boot

# Edit boot parameters
# 1. Select kernel
# 2. Press 'e' to edit
# 3. Modify parameters (e.g., add "quiet" or "debug")
# 4. Press Ctrl+x to boot
```

#### GRUB Shell Commands

```bash
# Access GRUB shell after boot
init=/bin/bash
# Or
grub> boot

# GRUB shell commands
grub> ls                        # List devices
grub> cat /boot/grub2/grub.cfg  # Show config
grub> insmod part_gpt           # Load partition module
grub> insmod normal             # Load normal module
grub> normal                    # Boot normally
grub> reboot                    # Reboot
grub> quit                      # Exit to shell
```

#### Kernel Parameters

```bash
# Edit GRUB configuration
sudo vi /etc/default/grub

# Common parameters
GRUB_CMDLINE_LINUX="rhgb quiet debug"
GRUB_TIMEOUT=10
GRUB_TERMINAL=serial
```

#### Update GRUB Configuration

```bash
# Update GRUB for BIOS
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Update GRUB for UEFI
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Verify configuration
sudo grub2-editenv list
```

### Process Management

#### Viewing Processes

```bash
# List all processes
ps aux

# List processes by user
ps -u username

# List processes by PID
ps -p PID

# List processes in tree format
pstree

# List processes sorted by CPU
ps aux --sort=-%cpu | head

# List processes sorted by memory
ps aux --sort=-%mem | head

# Real-time process viewer
top
htop
```

#### Identifying Resource-Intensive Processes

```bash
# Top 10 CPU consumers
ps aux --sort=-%cpu | head -11

# Top 10 memory consumers
ps aux --sort=-%mem | head -11

# Processes with high I/O
iostat -x 1
iotop

# View process details
ps -eo pid,ppid,user,%cpu,%mem,vsz,rss,stat,cmd | head -20
```

#### Terminating Processes

```bash
# Send SIGTERM (graceful termination)
kill PID
kill -15 PID

# Send SIGKILL (force termination)
kill -9 PID

# Kill by process name
pkill process_name
killall process_name

# Kill specific user's processes
kill -9 $(whoami)
sudo kill -9 $(whoami)

# Force kill with timeout
timeout -k 5 30 kill PID    # 30s wait, 5s force

# View process tree and kill
pstree -p | grep process
```

#### Process Signals

```bash
# Common signals
kill -1 PID     # SIGHUP - Hangup
kill -2 PID     # SIGINT - Interrupt
kill -9 PID     # SIGKILL - Kill (cannot be caught)
kill -15 PID    # SIGTERM - Termination
kill -0 PID     # Check process exists
kill -17 PID    # SIGCHLD - Child status
kill -30 PID    # SIGUSR1 - User-defined
```

#### Adjusting Process Scheduling

```bash
# Check current nice value
ps -o pid,ppid,user,nice,%cpu -p PID

# View current scheduling parameters
cat /proc/PID/sched

# Start process with specific nice value
nice -n 10 command

# Start with lower priority (higher nice)
nice -n 19 command

# Start with higher priority (lower nice)
nice -n -10 command

# Change running process priority
renice -n 10 -p PID
renice -n 0 -p PID
```

#### Process Priority Examples

```bash
# Background process with low priority
nice -n 10 -o command &

# Run CPU-intensive task with low priority
nice -n 19 compile_program.sh

# Run interactive task with normal priority
nice -n 0 interactive_app

# Change priority of background job
renice -n 5 -p JOB_PID
```

### Tuning Profiles

#### Viewing Current Profile

```bash
# List available profiles
systemctl list-unit-files | grep profile
systemctl list-unit-files | grep tuning

# View current profile
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Check system profile
rpm -q kernel -f | grep profile
```

#### Changing Tuning Profile

```bash
# List available profiles
rpm -qa | grep -i profile

# Install tuning profile (if available)
sudo dnf install -y tuning-profile

# Set performance profile
sudo tuning-set-profile performance

# Set balanced profile
sudo tuning-set-profile balanced

# Set power-savings profile
sudo tuning-set-profile power-savings

# Set throughput profile
sudo tuning-set-profile throughput
```

#### Viewing Profile Effects

```bash
# View CPU governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# View I/O scheduler
cat /sys/block/sda/queue/scheduler

# View memory cgroup settings
cat /proc/sys/vm/swappiness
cat /proc/sys/vm/drop_caches
```

### System Logging

#### Viewing System Logs

```bash
# View journal logs
journalctl

# View recent logs
journalctl -n 50

# Follow logs in real-time
journalctl -f

# View logs from specific service
journalctl -u nginx

# View logs with time range
journalctl --since "1 hour ago"
journalctl --until "1 hour ago"
journalctl --since "2024-01-01" --until "2024-01-02"

# View logs by priority
journalctl -p err
journalctl -p warn
journalctl -p alert

# View logs by PID
journalctl -u service -p err
```

#### Viewing Traditional Logs

```bash
# View system messages
tail -f /var/log/messages

# View secure authentication logs
tail -f /var/log/secure

# View kernel messages
tail -f /var/log/kern.log

# View boot messages
dmesg | tail -50

# View audit logs
tail -f /var/log/audit/audit.log
```

#### Searching Logs

```bash
# Search journal for pattern
journalctl -u nginx | grep "error"

# Search with regex
journalctl -u nginx | grep "connection.*refused"

# Search by file
grep -r "error" /var/log/

# Search with line numbers
grep -n "error" /var/log/messages

# Search in recent logs
grep "error" /var/log/messages | tail -20
```

#### Preserving System Journals

```bash
# View journal storage
journalctl --disk-usage

# Preserve journal (prevent automatic purging)
journalctl --vacuum-time=7d
journalctl --vacuum-size=100M
journalctl --vacuum-files=100

# Save journal to file
journalctl > /tmp/journal.log

# Save journal to remote
journalctl --output=journal | journalctl -u --file=/tmp/journal.log

# Send journal to remote syslog
journalctl --output=remote
```

#### Journal Forensics

```bash
# View all journal entries
journalctl --all

# View verbose journal
journalctl -v

# View with full output
journalctl -o verbose

# View boot logs
journalctl -b -1    # Previous boot
journalctl -b       # Current boot

# View specific time
journalctl -b -1 --since "00:00:00" --until "01:00:00"

# Export journal to text
journalctl --output=cat > /tmp/journal.txt
```

### Starting, Stopping, and Checking Services

#### Service Management Commands

```bash
# Start service
sudo systemctl start nginx

# Stop service
sudo systemctl stop nginx

# Restart service
sudo systemctl restart nginx

# Reload service (re-read config)
sudo systemctl reload nginx

# Try-restart service
sudo systemctl try-restart nginx

# Check service status
systemctl status nginx

# View service details
systemctl cat nginx

# Enable service at boot
sudo systemctl enable nginx

# Disable service at boot
sudo systemctl disable nginx

# Check if enabled
systemctl is-enabled nginx

# View enabled services
systemctl list-unit-files --type=service | grep enabled
```

#### Service Status Examples

```bash
# Detailed status
systemctl status nginx -l

# Short status
systemctl status nginx --no-pager

# Check active services
systemctl list-units --type=service --state=running

# Check failed services
systemctl list-units --type=service --state=failed

# Check all services
systemctl list-units --type=service

# View service dependencies
systemctl list-dependencies nginx

# View service logs
journalctl -u nginx -f
```

#### Service Type Commands

```bash
# Service types
systemctl list-unit-files --type=service
systemctl list-unit-files --type=socket
systemctl list-unit-files --type=target
systemctl list-unit-files --type=timer

# Service states
systemctl list-units --state=running
systemctl list-units --state=stopped
systemctl list-units --state=failed
```

### Secure File Transfer

#### SCP (Secure Copy)

```bash
# Copy file from remote to local
scp user@remote:/path/to/file /local/path/

# Copy file from local to remote
scp /local/path/file user@remote:/remote/path/

# Copy directory recursively
scp -r /local/path/user@remote:/remote/path/

# Copy with verbose output
scp -v user@remote:/path/file /local/path/

# Copy with progress
scp -p user@remote:/path/file /local/path/

# Copy with specific port
scp -P 2222 user@remote:/path/file /local/path/

# Copy with timeout
scp -o Timeout=60 user@remote:/path/file /local/path/
```

#### SFTP (SSH File Transfer Protocol)

```bash
# Start SFTP session
sftp user@remote

# SFTP commands
put localfile.txt          # Upload file
get remotefile.txt         # Download file
lcd /local/path            # Change local directory
cd /remote/path            # Change remote directory
ls                         # List remote files
rm remotefile.txt          # Remove remote file
mkdir directory            # Create directory
exit                       # Exit SFTP
```

#### Rsync with SSH

```bash
# Sync directory to remote
rsync -avz -e ssh /local/path/ user@remote:/remote/path/

# Sync with progress
rsync -avz --progress -e ssh /local/path/ user@remote:/remote/path/

# Sync delete files not in source
rsync -avz --delete -e ssh /local/path/ user@remote:/remote/path/

# Sync with specific port
rsync -avz -e "ssh -p 2222" /local/path/ user@remote:/remote/path/

# Sync with exclude patterns
rsync -avz --exclude='*.log' --exclude='*.tmp' -e ssh /local/path/ user@remote:/remote/path/

# Sync with compression
rsync -avz -e ssh /local/path/ user@remote:/remote/path/

# Sync with checksum verification
rsync -avz --checksum -e ssh /local/path/ user@remote:/remote/path/

# Run after transfer
rsync -avz -e ssh /local/path/ user@remote:/remote/path/ --after="ssh user@remote 'systemctl restart nginx'"
```

#### SSH Configuration for File Transfer

```bash
# Create ~/.ssh/config
cat > ~/.ssh/config << 'EOF'
Host web01
    HostName 192.168.1.10
    User admin
    Port 22
    IdentityFile ~/.ssh/web01_key

Host db01
    HostName 192.168.1.20
    User dbadmin
    Port 2222
    IdentityFile ~/.ssh/db01_key
EOF

# Use config for transfers
scp web01:/path/file /local/path/
scp /local/path/file db01:/path/
```

---

## Configuration Examples

### Systemd Service Unit File

File: `/etc/systemd/scripts/myapp.service`

```ini
[Unit]
Description=My Application Service
Documentation=man:myapp(8)
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment="PORT=8080"
Environment="MAX_CONNECTIONS=100"
ExecStart=/opt/myapp/bin/server
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStartSec=60
TimeoutStopSec=30

# Security hardening
PrivateTmp=true
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Resource limits
MemoryLimit=512M
CPUQuota=50%

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

### GRUB Configuration

File: `/etc/default/grub`

```bash
# GRUB configuration for RHEL 10

# Boot loader configuration
GRUB_DEFAULT=0
GRUB_TIMEOUT=10
GRUB_TIMEOUT_STYLE=menu
GRUB_DISTRIBUTOR="Red Hat Enterprise Linux"

# Boot parameters
GRUB_CMDLINE_LINUX="rhgb quiet console=:0 loglevel=3"
GRUB_CMDLINE_LINUX_DEFAULT="rhgb quiet"

# GRUB settings
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT=console
GRUB_TERMINAL=serial
GRUB_SERIAL_COMMAND="serial --speed=115200 --word=8 --parity=no --stop=1"
GRUB_CMDLINE_SERIAL="console=ttyS0,115200n8"

# GRUB security
GRUB_DISABLE_RECOVERY="true"
GRUB_DISABLE_OS_PROBER=false

# Boot loader location
GRUB_ENABLE_BLSCFG=true
```

### Sysctl Performance Tuning

File: `/etc/sysctl.d/99-performance.conf`

```bash
# Network performance tuning
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Memory tuning
vm.swappiness = 10
vm.vfs_cache_pressure = 50

# File descriptor limits
fs.file-max = 2097152

# Network buffer sizes
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# Enable BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

### Journal Preservation Configuration

File: `/etc/systemd/journald.conf`

```ini
# Journal configuration
[Journal]
# Storage type (files, volatile, auto)
Storage=persistent

# Maximum disk usage
SystemMaxUse=10G

# Maximum number of files
SystemKeepFree=1G
SystemMaxFiles=10000

# Retention time
SystemMaxUse=10G
KeepFree=1G
MaxRetentionSec=7d

# Field storage
FieldStore=verbose

# Rate limiting
RateLimitIntervalSec=300
RateLimitBurst=1000

[Remote]
# Remote syslog configuration
RemoteHost=192.168.1.10
RemoteTransport=socket
```

### SSH Configuration for Transfers

File: `/etc/ssh/ssh_config`

```bash
# SSH client configuration
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    BatchMode no
    GSSAPIAuthentication no
    GSSAPIDelegateCredentials no

# Performance settings
Compression yes
CompressionLevel 9
UseDNS no

# Security settings
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512

# Host-specific settings
Host internal-*
    User admin
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

Host *.internal
    ProxyJump internal-gateway.internal
```

---

## Real-World Examples

### Scenario 1: Automated System Health Monitor

A script that monitors system health and takes corrective actions.

```bash
#!/bin/bash
#
# System Health Monitor Script
# Checks CPU, memory, disk, and services
#

set -euo pipefail

LOG_FILE="/var/log/system_health.log"
CRITICAL_THRESHOLD=90
WARNING_THRESHOLD=75

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Check CPU usage
check_cpu() {
    local cpu_usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    
    if (( $(echo "$cpu_usage > $CRITICAL_THRESHOLD" | bc -l) )); then
        log "CRITICAL: CPU usage at ${cpu_usage}%"
        echo "CPU usage at ${cpu_usage}% exceeds ${CRITICAL_THRESHOLD}%"
        # Kill top CPU consumer
        ps aux --sort=-%cpu | head -2 | tail -1 | awk '{print $2}' | xargs kill -15 2>/dev/null || true
    elif (( $(echo "$cpu_usage > $WARNING_THRESHOLD" | bc -l) )); then
        log "WARNING: CPU usage at ${cpu_usage}%"
    else
        log "INFO: CPU usage at ${cpu_usage}%"
    fi
}

# Check memory usage
check_memory() {
    local mem_info
    mem_info=$(free | grep Mem)
    local total=$(echo $mem_info | awk '{print $2}')
    local used=$(echo $mem_info | awk '{print $3}')
    local percent=$((used * 100 / total))
    
    if [ "$percent" -gt "$CRITICAL_THRESHOLD" ]; then
        log "CRITICAL: Memory usage at ${percent}%"
        echo "Memory usage at ${percent}% exceeds ${CRITICAL_THRESHOLD}%"
        # Free memory
        echo 3 > /proc/sys/vm/drop_caches 2>/dev/null || true
    elif [ "$percent" -gt "$WARNING_THRESHOLD" ]; then
        log "WARNING: Memory usage at ${percent}%"
    else
        log "INFO: Memory usage at ${percent}%"
    fi
}

# Check disk usage
check_disk() {
    while IFS= read -r line; do
        local usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        local mount=$(echo "$line" | awk '{print $6}')
        
        if [ "$usage" -gt "$CRITICAL_THRESHOLD" ]; then
            log "CRITICAL: Disk usage on $mount at ${usage}%"
            echo "Disk usage on $mount at ${usage}% exceeds ${CRITICAL_THRESHOLD}%"
        elif [ "$usage" -gt "$WARNING_THRESHOLD" ]; then
            log "WARNING: Disk usage on $mount at ${usage}%"
        else
            log "INFO: Disk usage on $mount at ${usage}%"
        fi
    done < <(df -h | grep -v tmpfs)
}

# Check critical services
check_services() {
    local services=("sshd" "cron" "firewalld")
    
    for service in "${services[@]}"; do
        if ! systemctl is-active "$service" > /dev/null 2>&1; then
            log "ERROR: Service $service is not running"
            echo "Service $service is not running, attempting to start..."
            systemctl start "$service" 2>/dev/null || true
        else
            log "INFO: Service $service is running"
        fi
    done
}

# Main execution
log "=== System Health Check Started ==="
check_cpu
check_memory
check_disk
check_services
log "=== System Health Check Completed ==="
```

### Scenario 2: Automated Backup with Verification

A script that backs up critical data and verifies the backup.

```bash
#!/bin/bash
#
# Automated Backup Script with Verification
#

set -euo pipefail

BACKUP_DIR="/var/backups"
SOURCE_DIR="/etc"
RETENTION_DAYS=7
DATE_STAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="backup_${DATE_STAMP}.tar.gz"
LOG_FILE="/var/log/backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create backup
create_backup() {
    log "Creating backup of $SOURCE_DIR"
    mkdir -p "$BACKUP_DIR"
    
    if tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" -C / "${SOURCE_DIR#/}" 2>/dev/null; then
        log "Backup created: ${BACKUP_FILE}"
        local size
        size=$(du -sh "${BACKUP_DIR}/${BACKUP_FILE}" | cut -f1)
        log "Backup size: $size"
        echo "true"
    else
        log "ERROR: Backup creation failed"
        echo "false"
    fi
}

# Verify backup
verify_backup() {
    local backup_file="${BACKUP_DIR}/${BACKUP_FILE}"
    
    log "Verifying backup integrity"
    if tar -tzf "$backup_file" > /dev/null 2>&1; then
        log "Backup verification: PASSED"
        echo "true"
    else
        log "ERROR: Backup verification: FAILED"
        echo "false"
    fi
}

# Rotate old backups
rotate_backups() {
    log "Rotating old backups (older than ${RETENTION_DAYS} days)"
    local deleted
    deleted=$(find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +${RETENTION_DAYS} -delete -print | wc -l)
    log "Deleted $deleted old backup(s)"
    
    # Keep only last 10 backups
    local count
    count=$(ls -1t "${BACKUP_DIR}"/backup_*.tar.gz 2>/dev/null | wc -l)
    if [ "$count" -gt 10 ]; then
        local excess=$((count - 10))
        ls -1t "${BACKUP_DIR}"/backup_*.tar.gz | tail -n "$excess" | xargs rm -f 2>/dev/null || true
        log "Removed $excess excess backup(s)"
    fi
}

# Main execution
log "=== Backup Process Started ==="
if create_backup && verify_backup; then
    rotate_backups
    log "=== Backup Process Completed Successfully ==="
    exit 0
else
    log "=== Backup Process Failed ==="
    exit 1
fi
```

### Scenario 3: Service Auto-Recovery

A script that monitors services and restarts them if they fail.

```bash
#!/bin/bash
#
# Service Auto-Recovery Script
# Monitors services and restarts if needed
#

set -euo pipefail

SERVICES=("sshd" "httpd" "firewalld" "cron")
CHECK_INTERVAL=60
MAX_RESTARTS=3
LOG_FILE="/var/log/service_autorecovery.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

# Restart service with retry
restart_service() {
    local service="$1"
    local max_retries="$2"
    local retry_count=0
    
    log "Attempting to restart $service"
    
    while [ "$retry_count" -lt "$max_retries" ]; do
        if systemctl start "$service" > /dev/null 2>&1; then
            if systemctl is-active "$service" > /dev/null 2>&1; then
                log "Service $service restarted successfully"
                return 0
            fi
        fi
        ((retry_count++))
        log "Retry $retry_count: $service failed to start, waiting 30s"
        sleep 30
    done
    
    log "ERROR: Failed to start $service after $max_retries attempts"
    return 1
}

# Main monitoring loop
log "Service auto-recovery started"

while true; do
    for service in "${SERVICES[@]}"; do
        if ! systemctl is-active "$service" > /dev/null 2>&1; then
            log "WARNING: Service $service is not running"
            restart_service "$service" "$MAX_RESTARTS" || true
        fi
    done
    
    sleep "$CHECK_INTERVAL"
done
```

---

## Troubleshooting

### Boot Issues

**Problem**: System fails to boot

```bash
# Access GRUB shell
grub> ls
grub> normal
# Or
grub> insmod part_gpt
grub> insmod normal
grub> normal

# Boot into rescue mode
sudo systemctl reboot --target=rescue.target

# Edit kernel parameters
grub> edit
# Modify parameters, press Ctrl+x to boot
```

**Problem**: System stuck at GRUB menu

```bash
# Increase timeout
sudo vi /etc/default/grub
GRUB_TIMEOUT=10

# Disable graphical boot
sudo vi /etc/default/grub
GRUB_CMDLINE_LINUX="quiet"

# Update GRUB
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Process Issues

**Problem**: Process not responding

```bash
# Check process state
ps -p PID -o stat,cmd

# Send SIGSTOP (pause process)
kill -STOP PID

# Send SIGCONT (resume process)
kill -CONT PID

# Check if process is zombie
ps -o pid,stat,ppid,args | grep Z

# Find zombie processes
ps aux | awk '$8 ~ /Z/'
```

**Problem**: Process using excessive resources

```bash
# Find top CPU consumer
ps aux --sort=-%cpu | head -5

# Find top memory consumer
ps aux --sort=-%mem | head -5

# Kill process
kill -9 PID

# Or use pkill
pkill -9 process_name
```

### Service Issues

**Problem**: Service fails to start

```bash
# Check service status
systemctl status nginx

# View service logs
journalctl -u nginx

# Check service dependencies
systemctl list-dependencies nginx

# Enable service
sudo systemctl enable nginx

# Check for errors
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

**Problem**: Service starts but fails

```bash
# Check service journal
journalctl -u nginx -xe

# View service unit file
systemctl cat nginx

# Check for permission issues
ls -la /var/log/nginx/

# Restart service
sudo systemctl restart nginx

# Check for port conflicts
ss -tlnp | grep :80
```

### Log Issues

**Problem**: Journal not working

```bash
# Check journal status
systemctl status systemd-journald

# Check journal directory
ls -la /var/log/journal/

# Clear journal
sudo journalctl --vacuum-time=-1

# Restart journal
sudo systemctl restart systemd-journald
```

**Problem**: Logs growing too large

```bash
# Check journal size
journalctl --disk-usage

# Set size limit
sudo vi /etc/systemd/journald.conf
SystemMaxUse=10G

# Restart journal
sudo systemctl restart systemd-journald
```

### File Transfer Issues

**Problem**: SCP/SFTP connection fails

```bash
# Check SSH connectivity
ssh user@remote

# Verify SSH key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "user@host"
ssh-copy-id user@remote

# Check SSH config
cat ~/.ssh/config

# Test with verbose SSH
ssh -v user@remote
```

**Problem**: Rsync transfer fails

```bash
# Check SSH connectivity
ssh user@remote

# Test rsync with verbose
rsync -avz -e ssh /local/path/ user@remote:/remote/path/

# Check for permission issues
ssh user@remote "ls -la /remote/path/"

# Test with specific options
rsync -avz --partial --progress -e ssh /local/path/ user@remote:/remote/path/
```

---

## Verification Procedures

### Boot Verification

```bash
# Verify GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-editenv list

# Verify systemd targets
systemctl list-units --type=target --state=active

# Verify default target
systemctl get-default
```

### Process Verification

```bash
# Verify process management
ps aux | head -5
top -bn1

# Verify process termination
kill PID
ps -p PID

# Verify process priority
ps -o pid,nice,cmd
```

### Service Verification

```bash
# Verify service status
systemctl status nginx

# Verify service management
systemctl start nginx
systemctl stop nginx
systemctl restart nginx

# Verify service logging
journalctl -u nginx -n 20
```

### Log Verification

```bash
# Verify journal functionality
journalctl -n 10

# Verify log files
ls -la /var/log/messages
ls -la /var/log/secure

# Verify log rotation
ls -la /var/log/*.gz
```

### File Transfer Verification

```bash
# Verify SSH connectivity
ssh -v user@remote

# Verify SCP transfer
scp /local/file user@remote:/remote/
ls -la /remote/

# Verify SFTP
sftp user@remote
put /local/file
ls

# Verify rsync
rsync -avz -e ssh /local/path/ user@remote:/remote/path/
```

---

## RHCSA Exam Notes

### Exam Relevance

Operating running systems is a heavily tested area on the RHCSA. Expect 3-5 questions covering boot management, process control, service management, logging, and secure file transfer.

### Key Exam Points

1. **Systemd Commands**: Know `systemctl start/stop/restart/status/enabled` for all operations.

2. **Service Management**: Understand service states (running, stopped, failed, disabled) and how to transition between them.

3. **Process Control**: Know how to identify resource-intensive processes using `ps`, `top`, and `htop`, and terminate them with `kill` and `pkill`.

4. **Journal Logging**: Know how to query and filter journal logs with `journalctl`.

5. **Secure File Transfer**: Know SCP, SFTP, and rsync syntax for file transfers over SSH.

6. **GRUB Boot**: Know how to access GRUB shell and modify boot parameters.

### Common Exam Traps

- Forgetting `sudo` for privileged operations
- Not using the correct service name (check with `systemctl list-units`)
- Confusing `systemctl start` with `systemctl enable`
- Using `kill` without specifying signal number
- Forgetting to check service dependencies
- Not using verbose mode for debugging SSH issues

### Time Management Tips

- Use `systemctl status service` to quickly check service status
- Use `journalctl -u service -n 20` to quickly view recent service logs
- Use `ps aux --sort=-%cpu` to quickly find CPU-intensive processes
- Use `scp -r` for recursive directory transfers

### Exam Environment Notes

- You will have access to both desktop and server systems
- SSH keys may need to be configured for remote access
- Service names may differ from common names (e.g., `httpd` vs `nginx`)
- Journal logs are available via `journalctl`
- File transfer may require SSH key setup

---

## Chapter Summary

This chapter covered essential system operation tasks including boot management, process control, service management, logging, and secure file transfer. You learned how to use systemd for boot control and service management, identify and terminate resource-intensive processes, query and preserve system journals, and transfer files securely using SCP, SFTP, and rsync.

These skills form the foundation for daily system administration. Whether recovering from a failed boot, managing runaway processes, troubleshooting services, or transferring data between systems, these commands are used constantly on RHEL systems.

---

## Quick Reference

### Systemd Commands

```bash
systemctl start service           # Start service
systemctl stop service            # Stop service
systemctl restart service         # Restart service
systemctl reload service          # Reload configuration
systemctl status service          # Check status
systemctl enable service          # Enable at boot
systemctl disable service         # Disable at boot
systemctl is-enabled service      # Check if enabled
systemctl list-units              # List all units
systemctl list-unit-files         # List unit files
systemctl cat service             # View unit file
systemctl isolate target          # Switch to target
```

### Process Management

```bash
ps aux                            # List all processes
ps -u user                        # List by user
ps -p PID                         # List by PID
pstree                            # Tree view
top                               # Real-time monitor
htop                              # Enhanced monitor
kill PID                          # Send SIGTERM
kill -9 PID                       # Send SIGKILL
pkill process_name                # Kill by name
renice -n N -p PID                # Change priority
nice -n N command                 # Start with priority
```

### System Logging

```bash
journalctl                        # View all logs
journalctl -u service             # View service logs
journalctl -f                     # Follow logs
journalctl -n 50                  # Last 50 entries
journalctl --since "1 hour ago"   # Time range
journalctl -p err                 # Error priority
journalctl --vacuum-time=7d       # Retention
```

### Secure File Transfer

```bash
scp user@host:/path/file /local/  # Copy from remote
scp /local/file user@host:/path/  # Copy to remote
scp -r /local/dir user@host:/path/ # Recursive copy
sftp user@host                    # Interactive transfer
rsync -avz -e ssh /local/ user@host:/remote/ # Sync
```

### Boot Management

```bash
systemctl get-default             # View default target
systemctl set-default target      # Change default
systemctl reboot --target=target  # Boot to target
grub2-mkconfig -o /boot/grub2/grub.cfg  # Update GRUB
```

### Service States

```bash
systemctl list-units --state=running   # Running services
systemctl list-units --state=failed    # Failed services
systemctl list-unit-files --enabled    # Enabled services
```

---

## Review Questions

1. How do you view the current default systemd target on a RHEL 10 system?

2. What is the difference between `systemctl start` and `systemctl enable`?

3. How do you identify the top 5 CPU-consuming processes on a system?

4. What signal does the `kill` command send by default, and how do you send SIGKILL?

5. How do you search for error messages in the system journal for a specific service?

6. What command would you use to preserve system journal logs for forensic analysis?

7. How do you securely copy a file from a remote server to a local system using SCP?

8. What is the purpose of the `rsync` command, and how does it differ from `scp`?

9. How do you boot a system into rescue mode using systemd?

10. What command would you use to change the default boot target from graphical to multi-user?

11. How do you view the unit file for a specific systemd service?

12. What is the purpose of the `journalctl --vacuum` command?

13. How do you check if a service is enabled at boot time?

14. What command would you use to send a process to the background with low CPU priority?

15. How do you verify that a journal entry was successfully written?

---

## Answers

1. Use `systemctl get-default` to view the current default systemd target. Alternatively, `systemctl list-default` shows the symlink to the default target.

2. `systemctl start service` starts a service immediately but does not persist across reboots. `systemctl enable service` creates a symlink to make the service start automatically at boot time. You typically use both: `systemctl start service` for immediate effect and `systemctl enable service` for persistence.

3. Use `ps aux --sort=-%cpu | head -6` to list all processes sorted by CPU usage in descending order, showing the top 6 (header + 5 processes). Alternatively, `top` and press `Shift+P` to sort by CPU.

4. The `kill` command sends SIGTERM (signal 15) by default. To send SIGKILL, use `kill -9 PID` or `kill --signal=KILL PID`. SIGKILL cannot be caught or ignored by the process.

5. Use `journalctl -u service_name -p err -n 200` to view the last 200 error-level log entries for a specific service. You can also use `journalctl -u service_name | grep -i error`.

6. Use `journalctl --vacuum-time=7d` or `journalctl --vacuum-size=10G` to prevent automatic purging of journal logs. This keeps logs for the specified time or size. For manual preservation, use `journalctl > /path/to/file`.

7. Use `scp user@hostname:/remote/path/file /local/path/` to copy a file from a remote server to a local system. This uses SSH for secure encrypted transfer.

8. `rsync` is designed for efficient file synchronization, transferring only changed blocks and supporting delta transfers. It can resume interrupted transfers with the `--partial` option. `scp` is simpler and better for one-time file copies. Both can use SSH for security, but `rsync` is faster for large directory transfers.

9. Use `sudo systemctl reboot --target=rescue.target` to reboot into rescue mode. Alternatively, access the GRUB menu, select the kernel, press `e`, add `rescue` to kernel parameters, and boot.

10. Use `sudo systemctl set-default multi-user.target` to change the default boot target. This creates a symlink from `/etc/systemd/system/default.target` to `multi-user.target`. Reboot for the change to take effect.

11. Use `systemctl cat service_name` to view the unit file for a specific systemd service. This shows the complete configuration including `[Unit]`, `[Service]`, and `[Install]` sections.

12. The `journalctl --vacuum` command controls journal log retention. `--vacuum-time` sets retention time (e.g., `--vacuum-time=7d` keeps logs for 7 days). `--vacuum-size` sets maximum disk usage (e.g., `--vacuum-size=10G`). `--vacuum-files` limits the number of files.

13. Use `systemctl is-enabled service_name` to check if a service is enabled at boot time. It returns `enabled`, `disabled`, or `static`. Alternatively, `systemctl list-unit-files --type=service --state=enabled` lists all enabled services.

14. Use `nice -n 10 command` to start a command with a nice value of 10 (lower priority). Use `nice -n -10 command` for higher priority. For background processes, use `nice -n 10 command &`.

15. Verify journal entries by checking the journal disk usage with `journalctl --disk-usage` and viewing recent entries with `journalctl -n 20`. You can also check the journal directory `/var/log/journal/` for the presence of log files.