# Chapter 7: Deploying, Configuring, and Maintaining Systems

## Learning Objectives

By the end of this chapter, you will be able to:

- Schedule tasks using `at` and `cron`
- Schedule tasks using systemd timer units
- Start and stop services and configure services to start automatically at boot
- Configure systems to boot into a specific target automatically
- Configure time service clients (NTP)
- Install and update software packages from Red Hat CDN, remote repositories, or local file system
- Modify the system bootloader (GRUB2)
- Use `systemd-analyze` to measure boot time

---

## Concepts and Theory

### Scheduled Tasks

**Cron:**

Cron is a time-based job scheduler that runs commands at specified times. It uses a tab-separated configuration file with the following format:

```
minute hour day month weekday command
```

- **minute**: 0-59
- **hour**: 0-23
- **day**: 1-31
- **month**: 1-12
- **weekday**: 0-7 (0 and 7 are Sunday)

**Common Cron Expressions:**

| Expression | Meaning |
|------------|---------|
| `*` | Every value |
| `,` | List (e.g., `1,15,30`) |
| `-` | Range (e.g., `1-5`) |
| `/` | Step (e.g., `*/15` every 15 minutes) |

**at:**

The `at` command schedules one-time tasks to run at a specific time. Tasks are stored in `/var/spool/cron/atjobs/` and executed by the `atd` daemon.

**systemd Timers:**

systemd timers provide more flexible scheduling than cron, including:

- Calendar-based scheduling
- On-boot delays
- Persistent timers (run if missed)
- Coalescing (delay until previous task completes)
- Integration with systemd service units

### Systemd Services

**Service Unit Types:**

- **`simple`**: Forking process (default)
- **`forking`**: Executable forks to background
- **`oneshot`**: Runs once and exits
- **`dbus`**: Sends D-Bus message
- **`notify`**: Notifies when ready
- **`idle`**: Runs when system idle

**Service Directives:**

- **`Type`**: Process type
- **`ExecStart`**: Command to start
- **`ExecStop`**: Command to stop
- **`Restart`**: Auto-restart policy
- **`RestartSec`**: Seconds between restarts
- **`After`**: Dependencies
- **`Wants`**: Soft dependencies
- **`Requires`**: Hard dependencies
- **`User`**: Run as user
- **`Group`**: Run as group
- **`WorkingDirectory`**: Working directory
- **`Environment`**: Environment variables

### System Boot Targets

**Systemd Targets:**

- **`default.target`**: Default boot target (symlink)
- **`multi-user.target`**: Multi-user mode without GUI
- **`graphical.target`**: Multi-user mode with GUI
- **`rescue.target`**: Rescue mode
- **`emergency.target`**: Emergency mode
- **`network.target`**: Network configured
- **`network-online.target`**: Network fully online

**Boot Process:**

1. BIOS/UEFI initializes hardware
2. Bootloader (GRUB2) loads kernel and initramfs
3. systemd starts as PID 1
4. Filesystems are mounted
5. Services start according to dependencies
6. System switches to default target

### NTP (Network Time Protocol)

NTP synchronizes system clocks with remote time servers:

- **Protocol**: UDP port 123
- **Servers**: NTP or Chronyd
- **Accuracy**: Synchronize to within milliseconds
- **Security**: Uses authentication and encryption

**Chronyd:**

- Lightweight NTP daemon
- Uses system clock as reference
- Better for low-traffic networks
- Default on RHEL 10

**ntpd:**

- Traditional NTP daemon
- More features
- Higher resource usage

### Software Package Management

**DNF (Dandified YUM):**

- RPM-based package manager
- Automatic dependency resolution
- Repository management
- Transaction history

**DNF Commands:**

- `dnf install`: Install package
- `dnf remove`: Remove package
- `dnf update`: Update packages
- `dnf search`: Search packages
- `dnf info`: Package information

**Repositories:**

- **BaseOS**: Core OS packages
- **AppStream**: Application packages
- **CDN**: Red Hat Content Delivery Network
- **Local**: Local or remote repositories

### GRUB2 Bootloader

**GRUB2 Components:**

- **`/boot/grub2/grub.cfg`**: Main configuration
- **`/etc/default/grub`**: Default settings
- **`/boot/grub2/grubenv`**: Environment variables

**GRUB2 Directives:**

- **`GRUB_DEFAULT`**: Default boot entry
- **`GRUB_TIMEOUT`**: Timeout in seconds
- **`GRUB_CMDLINE_LINUX`**: Kernel parameters
- **`GRUB_TERMINAL`**: Terminal type
- **`GRUB_DISTRIBUTOR`**: Distribution name

---

## How It Works Internally

### Cron Execution

Cron works by:

1. **crond daemon** reads crontab files at startup
2. **Every minute**, crond checks all crontabs
3. **Match found**: Execute command with new shell
4. **Log output** to `/var/log/cron`
5. **Store job history** in `/var/spool/cron/`

Crontab files:
- `/etc/crontab`: System-wide crontab
- `/var/spool/cron/crontabs/username`: User crontab
- `~/.crontab`: Alias location

### systemd Timer Execution

systemd timers work by:

1. **Timer unit** defines schedule (OnCalendar, OnBootSec, etc.)
2. **Timer matches** current time
3. **Service unit** is started
4. **Service executes** according to type
5. **Timer state** updates (active, inactive, stopped)

Timer types:
- **oneshot**: Run once
- **state**: Based on service state
- **accurate**: Precise scheduling

### Service Startup Process

When a service starts:

1. **systemd reads** unit file
2. **Dependencies checked** (After, Requires)
3. **Pre-start scripts** execute (if configured)
4. **Main process** starts (ExecStart)
5. **Post-start scripts** execute (if configured)
6. **Service state** set (active, inactive, failed)
7. **Journal** logs activity

### NTP Synchronization

NTP synchronization works by:

1. **Client sends** time request to server
2. **Server responds** with current time
3. **Client calculates** offset and delay
4. **Client adjusts** system clock
5. **Client repeats** to maintain accuracy

Chronyd uses:
- **Stratum levels**: Server hierarchy (1-16)
- **Poll intervals**: Check frequency
- **Reference ID**: Time source identification

### DNF Package Resolution

DNF resolves dependencies by:

1. **Read repository metadata** (repodata)
2. **Parse package dependencies**
3. **Build dependency tree**
4. **Calculate transactions**
5. **Download packages**
6. **Install in order**
7. **Update RPM database**

### GRUB2 Boot Process

GRUB2 boot process:

1. **Bootloader reads** grub.cfg
2. **Parse menu entries**
3. **Display boot menu**
4. **User selects** entry or timeout expires
5. **Load kernel** and initramfs
6. **Transfer control** to kernel
7. **Kernel mounts** root filesystem
8. **systemd starts** as PID 1

---

## Commands and Administration Tasks

### Scheduling Tasks with Cron

#### Viewing and Editing Crontabs

```bash
# View current crontab
crontab -l

# Edit current crontab
crontab -e

# Set user crontab
sudo crontab -u username -e

# Install system crontab
sudo crontab /etc/cron.d/myjob

# List all crontabs
ls /var/spool/cron/crontabs/

# Remove crontab
crontab -r

# Check cron service
systemctl status crond
```

#### Creating Cron Jobs

```bash
# Run command every minute
* * * * * /usr/local/bin/monitor.sh

# Run command every 5 minutes
*/5 * * * * /usr/local/bin/check.sh

# Run command hourly
0 * * * * /usr/local/bin/hourly.sh

# Run command daily at midnight
0 0 * * * /usr/local/bin/daily.sh

# Run command daily at 2 AM
0 2 * * * /usr/local/bin/backup.sh

# Run command weekly on Sunday at 3 AM
0 3 * * 0 /usr/local/bin/weekly.sh

# Run command monthly on 1st at midnight
0 0 1 * * /usr/local/bin/monthly.sh

# Run command every Monday-Friday at 9 AM
0 9 * * 1-5 /usr/local/bin/business-hours.sh

# Run command every 15 minutes
*/15 * * * * /usr/local/bin/quarterly.sh

# Run command at specific times
0,30 * * * * /usr/local/bin/twice-daily.sh

# Run command with arguments
0 2 * * * /usr/local/bin/script.sh /path/to/data

# Run command with environment variables
0 2 * * * MAILTO=admin@example.com /usr/local/bin/backup.sh
```

#### Cron Environment Variables

```bash
# /etc/crontab environment variables
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
LOGNAME=root
```

#### User Crontab Example

```bash
# ~/.crontab or /var/spool/cron/crontabs/username
# Backup files at 3 AM daily
0 3 * * * /usr/local/bin/backup-home.sh

# Clean temp files at 4 AM daily
0 4 * * * /usr/local/bin/cleanup-temp.sh

# Send daily report at 6 AM
0 6 * * * /usr/local/bin/send-report.sh >> /var/log/report.log
```

### Scheduling Tasks with at

#### Using at Command

```bash
# Schedule task for specific time
at 14:30 today

# Schedule task for tomorrow
at tomorrow 14:30

# Schedule task in 5 minutes
at now + 5 minutes

# Schedule task in 1 hour
at now + 1 hour

# Schedule task on specific date
at 2024-01-15 14:30

# Schedule task at end of day
at midnight

# Schedule task for next occurrence
at next 5:00

# Schedule task with command
at 14:30 today << 'EOF'
echo "Task started"
/usr/local/bin/script.sh
echo "Task completed"
EOF

# View scheduled jobs
atq

# Remove job by number
atrm 1

# Remove all jobs
atrm -a

# Edit job
at -e 1

# Delete all at jobs
atq | xargs atrm
```

#### at Configuration

```bash
# /etc/at.deny - Users who cannot use at
# /etc/at.allow - Users who can use at

# View scheduled jobs
sudo atq

# Remove job
sudo atrm 1

# Edit job
sudo at -e 1

# Delete all jobs
sudo atq | xargs sudo atrm
```

### Scheduling Tasks with systemd Timers

#### Creating Timer Units

```bash
# Create timer unit file
sudo vi /etc/systemd/system/backup.timer

# Timer unit content:
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=daily
OnBootSec=10min
Persistent=true
Unit=backup.service

[Install]
WantedBy=timers.target
```

#### Creating Service Units

```bash
# Create service unit file
sudo vi /etc/systemd/system/backup.service

# Service unit content:
[Unit]
Description=Daily Backup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

#### Timer Scheduling Options

```bash
# Daily at midnight
OnCalendar=daily

# Every 15 minutes
OnCalendar=*:15/15:*:00

# Every Monday at 9 AM
OnCalendar=Mon *-*-* 09:00:00

# First day of every month
OnCalendar=*-*-01 00:00:00

# Every 30 seconds
OnCalendar=*/30 * * * *

# On boot, 5 minutes after
OnBootSec=5min

# Hourly
OnCalendar=hourly

# Daily
OnCalendar=daily

# Weekly
OnCalendar=weekly

# Monthly
OnCalendar=monthly

# Yearly
OnCalendar=yearly

# Persistent (run if missed)
Persistent=true

# Random delay (0-30 minutes)
RandomizedDelaySec=30min

# Coalesce (delay if previous running)
OnUnitActiveSec=10min
```

#### Managing Timers

```bash
# List all timers
systemctl list-timers

# List timer units
systemctl list-timers --all

# Enable timer
sudo systemctl enable backup.timer

# Disable timer
sudo systemctl disable backup.timer

# Start timer
sudo systemctl start backup.timer

# Stop timer
sudo systemctl stop backup.timer

# View timer status
sudo systemctl status backup.timer

# View timer next activation
systemctl show backup.timer | grep NextActivation

# View timer logs
journalctl -u backup.timer

# Trigger timer immediately
systemctl start backup.timer

# List active timers
systemctl list-timers --all
```

### Service Management

#### Starting and Stopping Services

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

# Short status
systemctl status nginx --no-pager

# Check if service is running
systemctl is-active nginx

# Check if service is enabled
systemctl is-enabled nginx

# Check service dependencies
systemctl list-dependencies nginx

# View service unit file
systemctl cat nginx
```

#### Enabling Services at Boot

```bash
# Enable service at boot
sudo systemctl enable nginx

# Disable service at boot
sudo systemctl disable nginx

# Enable with symlink
sudo ln -s /lib/systemd/system/nginx.service /etc/systemd/system/multi-user.target.wants/

# Disable symlink
sudo rm /etc/systemd/system/multi-user.target.wants/nginx.service

# List enabled services
systemctl list-unit-files --type=service --state=enabled

# List all services
systemctl list-units --type=service

# List failed services
systemctl list-units --type=service --state=failed

# List active services
systemctl list-units --type=service --state=running
```

#### Service Management Examples

```bash
# Start and enable HTTP server
sudo systemctl start httpd
sudo systemctl enable httpd

# Restart MySQL service
sudo systemctl restart mysqld

# Check PostgreSQL status
systemctl status postgresql

# Stop and disable firewall
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# Reload SSH configuration
sudo systemctl reload sshd

# Try-restart to check configuration
sudo systemctl try-restart nginx
```

### Configuring Boot Targets

#### Changing Default Boot Target

```bash
# View current default target
systemctl get-default
systemctl list-default

# Change default to multi-user
sudo systemctl set-default multi-user.target

# Change default to graphical
sudo systemctl set-default graphical.target

# Change default to rescue
sudo systemctl set-default rescue.target

# Change default to emergency
sudo systemctl set-default emergency.target

# Reboot to apply changes
sudo reboot

# View target symlink
ls -la /etc/systemd/system/default.target
```

#### Booting into Specific Target

```bash
# Reboot into rescue mode
sudo systemctl reboot --target=rescue.target

# Reboot into emergency mode
sudo systemctl reboot --target=emergency.target

# Switch to target immediately
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target
sudo systemctl isolate rescue.target

# List available targets
systemctl list-units --type=target --state=active

# View target dependencies
systemctl list-dependencies graphical.target
```

### Configuring NTP Time Service

#### Using Chronyd

```bash
# Install chronyd
sudo dnf install -y chrony

# Configure NTP servers
sudo vi /etc/chrony.conf

# Example /etc/chrony.conf:
# NTP servers
pool 0.pool.ntp.org iburst
pool 1.pool.ntp.org iburst
pool 2.pool.ntp.org iburst
pool 3.pool.ntp.org iburst

# Local stratum 10 server
server 192.168.1.1 iburst

# Drift file
driftfile /var/lib/chrony/drift

# Clock step
makestep 1.0 3

# Log files
logdir = /var/log/chrony

# Enable and start
sudo systemctl enable --now chronyd

# View status
chronyc tracking

# View sources
chronyc sources

# View statistics
chronyc stats

# Reset clock
chronyc adjust -s 100

# Stop chronyd
sudo systemctl stop chronyd
```

#### Using NTPD

```bash
# Install ntp
sudo dnf install -y ntp

# Configure NTP servers
sudo vi /etc/ntp.conf

# Example /etc/ntp.conf:
# NTP servers
server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst
server 3.pool.ntp.org iburst

# Drift file
driftfile /var/lib/ntp/ntp.drift

# Enable and start
sudo systemctl enable --now ntpd

# View status
ntpq -p

# View statistics
ntpq -c "rv 0"*

# Reset clock
ntpdate -s -u pool.ntp.org

# Stop ntpd
sudo systemctl stop ntpd
```

### Managing Software Packages

#### Installing Packages

```bash
# Install package
sudo dnf install nginx

# Install without prompt
sudo dnf install -y nginx

# Install multiple packages
sudo dnf install -y nginx php-fpm mariadb

# Install specific version
sudo dnf install nginx-1.24.0-1.el10

# Install from local RPM
sudo dnf install /path/to/package.rpm

# Install from URL
sudo dnf install https://example.com/packages/package.rpm

# Install from repository
sudo dnf install --enablerepo=custom-repo package

# Install with dependencies only
sudo dnf install --nodocs package

# Install with specific options
sudo dnf install --allowerasing package
```

#### Updating Packages

```bash
# Check for updates
sudo dnf check-update

# List available updates
sudo dnf list updates

# Update all packages
sudo dnf update

# Update specific package
sudo dnf update nginx

# Update and show obsoletes
sudo dnf update --best

# Update with minimal changes
sudo dnf update --minimal

# Update security packages only
sudo dnf update --security

# Update and remove unnecessary packages
sudo dnf update --clean

# Update from local file
sudo dnf update /path/to/package.rpm
```

#### Removing Packages

```bash
# Remove package
sudo dnf remove nginx

# Remove with dependencies
sudo dnf autoremove nginx

# Remove multiple packages
sudo dnf remove nginx php-fpm

# Remove without prompt
sudo dnf remove -y nginx

# Remove and clean metadata
sudo dnf remove nginx --allmetas

# List removed packages
dnf history list | grep REMOVE
```

#### Package Management

```bash
# Search for packages
sudo dnf search keyword

# List installed packages
sudo dnf list installed

# List available packages
sudo dnf list available

# List all packages
sudo dnf list all

# Get package info
sudo dnf info nginx

# List package files
sudo dnf repoquery -l nginx

# Find package owning file
sudo dnf provides /usr/bin/nginx

# List dependencies
sudo dnf repoquery --requires nginx

# List packages depending on package
sudo dnf repoquery --whatrequires nginx

# Check package integrity
rpm -V nginx

# View transaction history
sudo dnf history

# View transaction details
sudo dnf history info 42

# Undo transaction
sudo dnf history undo 42

# Redo transaction
sudo dnf history redo 42

# Clean DNF cache
sudo dnf clean all

# Clear specific cache
sudo dnf clean packages
sudo dnf clean metadata

# Make cache
sudo dnf makecache
```

### Modifying GRUB2 Bootloader

#### Viewing GRUB Configuration

```bash
# View GRUB configuration
cat /boot/grub2/grub.cfg

# View GRUB defaults
cat /etc/default/grub

# List GRUB entries
grub2-bls-list

# View GRUB environment
grub2-editenv list /boot/grub2/grubenv

# Test GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### Modifying GRUB Settings

```bash
# Edit GRUB defaults
sudo vi /etc/default/grub

# Common settings:
GRUB_DEFAULT=0                    # Default entry
GRUB_DEFAULT=saved                # Use saved entry
GRUB_TIMEOUT=10                   # Timeout in seconds
GRUB_TIMEOUT_STYLE=menu           # Show menu
GRUB_DISTRIBUTOR="RHEL"           # Distribution name
GRUB_CMDLINE_LINUX="rhgb quiet"   # Kernel parameters
GRUB_TERMINAL=console             # Terminal type

# Update GRUB configuration
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# For UEFI systems
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Set default entry
sudo grub2-reboot nginx 10        # Boot nginx for 10 seconds
sudo grub2-set-default 1          # Set entry 1 as default
```

#### GRUB2 Kernel Parameters

```bash
# Common kernel parameters:
rhgb quiet                        # Graphical boot, quiet messages
debug                             # Enable debug mode
single                            # Boot into single user mode
init=/bin/bash                    # Init to bash
rd.break                          # Break in initramfs
nomodeset                         # No video modes
nolapic                           # No LAPIC
net.ifnames=0                     # Disable predictable network names
```

#### GRUB2 Recovery

```bash
# Access GRUB menu
Reboot and press any key at the GRUB menu

# Edit boot entry
1. Select kernel
2. Press 'e' to edit
3. Modify parameters
4. Press Ctrl+x to boot

# GRUB shell commands
grub> ls                          # List devices
grub> cat /boot/grub2/grub.cfg    # Show config
grub> insmod part_gpt             # Load partition module
grub> insmod normal               # Load normal module
grub> normal                      # Boot normally
grub> reboot                      # Reboot
grub> quit                        # Exit to shell

# GRUB2 menu editor
grub> edit
# Modify parameters
grub> boot
```

---

## Configuration Examples

### Cron Job Configuration

File: `/etc/cron.d/system-monitor`

```bash
# System monitoring cron job
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# Check disk usage every 15 minutes
*/15 * * * * root /usr/local/bin/check-disk.sh >> /var/log/disk-check.log 2>&1

# Check memory usage every 5 minutes
*/5 * * * * root /usr/local/bin/check-memory.sh >> /var/log/memory-check.log 2>&1

# Check services every 10 minutes
*/10 * * * * root /usr/local/bin/check-services.sh >> /var/log/service-check.log 2>&1

# Send daily system report at 8 AM
0 8 * * * root /usr/local/bin/daily-report.sh >> /var/log/daily-report.log 2>&1
```

File: `/etc/cron.daily/backup`

```bash
#!/bin/bash
# Daily backup script

DATE=$(date +%Y%m%d)
BACKUP_DIR="/var/backups"
LOG_FILE="/var/log/backup.log"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup /etc
tar -czf "${BACKUP_DIR}/etc_${DATE}.tar.gz" /etc

# Verify backup
if tar -tzf "${BACKUP_DIR}/etc_${DATE}.tar.gz" > /dev/null 2>&1; then
    echo "[$(date)] Backup completed successfully" >> "$LOG_FILE"
else
    echo "[$(date)] Backup verification failed" >> "$LOG_FILE"
    exit 1
fi

# Remove backups older than 30 days
find "$BACKUP_DIR" -name "etc_*.tar.gz" -mtime +30 -delete

echo "[$(date)] Backup rotation completed" >> "$LOG_FILE"
```

### systemd Timer Configuration

File: `/etc/systemd/system/logrotate.timer`

```ini
[Unit]
Description=Daily Log Rotation Timer
After=local-fs.target

[Timer]
OnCalendar=daily
OnBootSec=10min
Persistent=true
Unit=logrotate.service

[Install]
WantedBy=timers.target
```

File: `/etc/systemd/system/logrotate.service`

```ini
[Unit]
Description=Log Rotation Service
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/logrotate /etc/logrotate.conf
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

File: `/etc/systemd/system/health-check.timer`

```ini
[Unit]
Description=System Health Check Timer
After=network-online.target

[Timer]
OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 06:00:00
OnCalendar=*-*-* 12:00:00
OnCalendar=*-*-* 18:00:00
Persistent=true
Unit=health-check.service

[Install]
WantedBy=timers.target
```

File: `/etc/systemd/system/health-check.service`

```ini
[Unit]
Description=System Health Check
After=network-online.target local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/health-check.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### NTP Configuration

File: `/etc/chrony.conf`

```conf
# NTP configuration for RHEL 10

# NTP servers
pool 0.pool.ntp.org iburst
pool 1.pool.ntp.org iburst
pool 2.pool.ntp.org iburst
pool 3.pool.ntp.org iburst

# Local stratum 10 server (replace with your gateway)
# server 192.168.1.1 iburst

# Allow clients
allow 192.168.1.0/24
allow 10.0.0.0/8
allow 172.16.0.0/12

# Drift file
driftfile /var/lib/chrony/drift

# Clock adjustment
makestep 1.0 3

# Log files
logdir = /var/log/chrony
log measurements statistics tracking

# RTC synchronization
rtcsync

# Bind to specific interface
bindaddr 0.0.0.0

# Key file for authentication
# keyfile /etc/chrony/chrony.keys
# commandkeyfile /etc/chrony/chrony.keys
```

### GRUB2 Configuration

File: `/etc/default/grub`

```bash
# GRUB2 configuration for RHEL 10

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

---

## Real-World Examples

### Scenario 1: Automated System Maintenance

A system needs automated maintenance tasks including log rotation, cleanup, and backups.

```bash
# Create maintenance script
sudo vi /usr/local/bin/system-maintenance.sh

# Script content:
#!/bin/bash
# System maintenance script

DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/maintenance.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Clean old logs
log "Cleaning old logs..."
find /var/log -name "*.log.*" -mtime +30 -delete

# Clean package cache
log "Cleaning package cache..."
dnf clean all

# Clean temp files
log "Cleaning temp files..."
find /tmp -type f -mtime +7 -delete

# Check disk usage
log "Checking disk usage..."
df -h | tee -a "$LOG_FILE"

# Check for failed services
log "Checking services..."
sudo systemctl list-units --type=service --state=failed | tee -a "$LOG_FILE"

# Rotate logs
log "Rotating logs..."
sudo logrotate -f /etc/logrotate.conf

log "Maintenance completed"
```

# Create systemd timer
sudo vi /etc/systemd/system/maintenance.timer

# Create service
sudo vi /etc/systemd/system/maintenance.service

# Enable timer
sudo systemctl enable maintenance.timer
sudo systemctl start maintenance.timer

### Scenario 2: Service Auto-Recovery

A critical service needs automatic restart if it fails.

```bash
# Create service unit with restart
sudo vi /usr/local/bin/myapp.service

# Service content:
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
ExecStart=/usr/local/bin/myapp
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

# Create timer for health check
sudo vi /usr/local/bin/myapp-health.timer

# Timer content:
[Unit]
Description=My Application Health Check

[Timer]
OnCalendar=*:0/5
Persistent=true
Unit=myapp.service

[Install]
WantedBy=timers.target

# Enable and start
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl enable myapp-health.timer
sudo systemctl start myapp-health.timer
```

### Scenario 3: Automated Backup System

A comprehensive backup system with verification and retention.

```bash
# Create backup script
sudo vi /usr/local/bin/automated-backup.sh

# Script content:
#!/bin/bash
# Automated backup system

BACKUP_DIR="/var/backups"
SOURCE="/etc"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="backup_${DATE}.tar.gz"
LOG_FILE="/var/log/backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Create backup
log "Creating backup of $SOURCE..."
if tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" -C / "${SOURCE#/}" 2>/dev/null; then
    BACKUP_SIZE=$(du -sh "${BACKUP_DIR}/${BACKUP_FILE}" | cut -f1)
    log "Backup created: ${BACKUP_FILE} (${BACKUP_SIZE})"
else
    log "ERROR: Backup creation failed"
    exit 1
fi

# Verify backup
log "Verifying backup..."
if tar -tzf "${BACKUP_DIR}/${BACKUP_FILE}" > /dev/null 2>&1; then
    log "Backup verification: PASSED"
else
    log "ERROR: Backup verification: FAILED"
    exit 1
fi

# Rotate old backups
log "Rotating old backups..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +${RETENTION_DAYS} -delete
log "Old backups removed"

# Keep only last 10 backups
ls -1t "${BACKUP_DIR}"/backup_*.tar.gz 2>/dev/null | tail -n +11 | xargs rm -f 2>/dev/null || true
log "Backup rotation completed"

log "Backup completed successfully"
```

# Create timer
sudo vi /etc/systemd/system/backup.timer

# Enable and start
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

---

## Troubleshooting

### Cron Issues

**Problem**: Cron job not running

```bash
# Check cron service
systemctl status crond

# Check crontab
crontab -l

# Check logs
tail -f /var/log/cron

# Check permissions
ls -la /var/spool/cron/crontabs/

# Check syntax
crontab -u username -l

# Check if user can use cron
cat /etc/cron.deny
cat /etc/cron.allow
```

**Problem**: Cron job running but not outputting

```bash
# Check log file
tail -f /var/log/cron

# Check mail delivery
mail -s "test" user@example.com

# Check cron permissions
ls -la /var/spool/cron/

# Check cron environment
cat /etc/crontab | head -10
```

### Timer Issues

**Problem**: systemd timer not triggering

```bash
# Check timer status
systemctl status backup.timer

# Check timer schedule
systemctl show backup.timer | grep NextActivation

# Check timer logs
journalctl -u backup.timer

# Check service status
systemctl status backup.service

# Enable and start timer
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

# Check if timer is active
systemctl list-timers
```

### Service Issues

**Problem**: Service fails to start

```bash
# Check service status
systemctl status nginx

# View service logs
journalctl -u nginx

# Check service unit file
systemctl cat nginx

# Check dependencies
systemctl list-dependencies nginx

# Check for errors
sudo systemctl daemon-reload

# Try-restart to check configuration
sudo systemctl try-restart nginx

# Check service journal
journalctl -u nginx -xe
```

**Problem**: Service starts but fails

```bash
# Check service logs
journalctl -u nginx -f

# Check service status
systemctl status nginx

# View service unit file
systemctl cat nginx

# Check for port conflicts
ss -tlnp | grep :80

# Check for permission issues
ls -la /var/log/nginx/

# Restart service
sudo systemctl restart nginx

# Check for SELinux issues
getenforce
restorecon -Rv /var/log/nginx/
```

### NTP Issues

**Problem**: NTP not synchronizing

```bash
# Check NTP status
chronyc tracking

# View NTP sources
chronyc sources

# Check NTP logs
journalctl -u chronyd

# Check if NTP is running
systemctl status chronyd

# View statistics
chronyc stats

# Reset clock
chronyc adjust -s 100

# Check network connectivity
ping pool.ntp.org

# Check firewall
firewall-cmd --list-all
```

### Package Issues

**Problem**: Package installation fails

```bash
# Check repository access
dnf repolist

# Check for errors
dnf check

# Clear cache
dnf clean all

# Make cache
dnf makecache

# Install with options
dnf install --allowerasing --best package

# Check dependencies
dnf deplist package

# Update all packages
dnf update

# Check for broken packages
rpm -Va
```

### GRUB2 Issues

**Problem**: GRUB2 not booting

```bash
# Access GRUB shell
grub> normal
# Or
grub> insmod part_gpt
grub> insmod normal
grub> normal

# Edit kernel parameters
grub> edit
# Modify parameters
grub> boot

# Update GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg

# View GRUB configuration
cat /boot/grub2/grub.cfg

# List GRUB entries
grub2-bls-list

# Set default entry
grub2-set-default 1

# Reboot
reboot
```

---

## Verification Procedures

### Cron Verification

```bash
# Verify cron service
systemctl status crond

# Verify crontab
crontab -l

# Test cron job
echo "* * * * * date >> /tmp/cron-test.log" | crontab -

# Wait and check
sleep 60
cat /tmp/cron-test.log

# Verify logs
tail -f /var/log/cron
```

### Timer Verification

```bash
# Verify timer status
systemctl status backup.timer

# Verify schedule
systemctl show backup.timer | grep NextActivation

# Test timer
sudo systemctl start backup.timer

# Wait and check
sleep 60
systemctl list-timers

# Verify logs
journalctl -u backup.timer
```

### Service Verification

```bash
# Verify service status
systemctl status nginx

# Verify service logs
journalctl -u nginx -n 20

# Verify service dependencies
systemctl list-dependencies nginx

# Test service start/stop
sudo systemctl start nginx
sudo systemctl stop nginx
```

### NTP Verification

```bash
# Verify NTP status
chronyc tracking

# Verify sources
chronyc sources -v

# Verify synchronization
chronyc sourcestats

# Verify logs
journalctl -u chronyd -n 20

# Test time difference
date
ntpdate -u pool.ntp.org
date
```

### Package Verification

```bash
# Verify package installation
dnf list installed nginx

# Verify package files
rpm -ql nginx | head -10

# Verify package integrity
rpm -V nginx

# Verify transaction history
dnf history

# Verify cache
dnf clean all
dnf makecache
```

### GRUB2 Verification

```bash
# Verify GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg

# Verify GRUB entries
grub2-bls-list

# Verify GRUB environment
grub2-editenv list /boot/grub2/grubenv

# Test GRUB configuration
grub2-testconfig

# Verify bootloader
ls /boot/grub2/
```

---

## RHCSA Exam Notes

### Exam Relevance

Deploying, configuring, and maintaining systems is heavily tested on the RHCSA. Expect 2-3 questions covering cron, systemd timers, service management, NTP, and GRUB2 configuration.

### Key Exam Points

1. **Cron**: Know how to create and edit crontabs, common cron expressions, and cron environment variables.

2. **systemd Timers**: Know how to create timer and service units, OnCalendar syntax, and timer management commands.

3. **Service Management**: Know systemctl commands for start/stop/restart, enable/disable, and status checking.

4. **Boot Targets**: Know how to change and view default targets, and boot into specific targets.

5. **NTP**: Know how to install and configure chronyd, and verify NTP synchronization.

6. **DNF**: Know package installation, updates, removal, and cache management.

7. **GRUB2**: Know how to modify GRUB configuration, kernel parameters, and recovery procedures.

### Common Exam Traps

- Forgetting `sudo` for privileged operations
- Using wrong cron syntax (wrong field order)
- Not enabling systemd timers (need both timer and service)
- Using `systemctl start` instead of `systemctl enable` for persistence
- Not using `--now` with systemctl enable
- Confusing chronyd and ntpd commands
- Forgetting to update GRUB configuration after changes

### Time Management Tips

- Use `systemctl status service` to quickly check service status
- Use `crontab -l` to quickly view crontab
- Use `journalctl -u service -n 20` to quickly view recent logs
- Use `dnf check-update` to quickly check for updates
- Use `grub2-mkconfig -o /boot/grub2/grub.cfg` to update GRUB

### Exam Environment Notes

- You will have access to systemd tools
- Cron jobs may need to be created manually
- NTP may need to be installed
- GRUB configuration may need to be updated
- Service units may need to be created

---

## Chapter Summary

This chapter covered system deployment, configuration, and maintenance including scheduled tasks with cron and systemd timers, service management, boot target configuration, NTP time synchronization, software package management with DNF, and GRUB2 bootloader configuration. You learned how to create and manage scheduled tasks, start and stop services, configure automatic startup, synchronize system time, manage software packages, and modify the system bootloader.

These skills are essential for system administration tasks including automated maintenance, service management, time synchronization, and software updates. Understanding these concepts helps administrators automate routine tasks, manage system state, and maintain system reliability.

---

## Quick Reference

### Cron Commands

```bash
crontab -l                  # View crontab
crontab -e                  # Edit crontab
crontab -r                  # Remove crontab
crontab -u user -e          # Edit user crontab
at 14:30                    # Schedule task
atq                         # List at jobs
atrm 1                      # Remove at job
```

### systemd Timer Commands

```bash
systemctl list-timers       # List timers
systemctl enable timer      # Enable timer
systemctl disable timer     # Disable timer
systemctl start timer       # Start timer
systemctl stop timer        # Stop timer
systemctl status timer      # Check timer status
```

### Service Commands

```bash
systemctl start service     # Start service
systemctl stop service      # Stop service
systemctl restart service   # Restart service
systemctl reload service    # Reload service
systemctl status service    # Check status
systemctl enable service    # Enable at boot
systemctl disable service   # Disable at boot
```

### NTP Commands

```bash
chronyc tracking            # View NTP status
chronyc sources             # View NTP sources
chronyc stats               # View statistics
systemctl status chronyd    # Check service
```

### DNF Commands

```bash
dnf install package         # Install package
dnf remove package          # Remove package
dnf update                  # Update packages
dnf search keyword          # Search packages
dnf info package            # Package info
dnf clean all               # Clean cache
dnf makecache               # Make cache
```

### GRUB2 Commands

```bash
grub2-mkconfig              # Update GRUB config
grub2-editenv               # Edit GRUB env
grub2-bls-list              # List GRUB entries
grub2-set-default           # Set default entry
```

---

## Review Questions

1. What is the difference between cron and systemd timers?

2. How do you schedule a task to run every 15 minutes using cron?

3. What command would you use to view the current crontab for a user?

4. How do you enable a systemd timer to start at boot?

5. What is the purpose of the `Persistent` option in systemd timers?

6. How do you change the default boot target from graphical to multi-user?

7. What command would you use to synchronize system time with NTP servers?

8. How do you install a package using DNF without interactive prompts?

9. What is the purpose of `GRUB_TIMEOUT` in `/etc/default/grub`?

10. How do you update the GRUB configuration after modifying `/etc/default/grub`?

11. What command would you use to check the status of a systemd service?

12. How do you configure a service to start automatically at boot?

13. What is the difference between `systemctl start` and `systemctl enable`?

14. How do you view the NTP synchronization status with chrony?

15. What command would you use to list all available updates?

---

## Answers

1. Cron is a traditional time-based scheduler that runs commands at specified times using crontab files. systemd timers are more flexible, supporting calendar-based scheduling, on-boot delays, persistent timers, and better integration with systemd services. Timers can also coalesce (delay until previous task completes) and are managed as units.

2. Use `*/15 * * * * /path/to/script.sh` in crontab. The `*/15` in the minute field means "every 15 minutes" (0, 15, 30, 45). The other fields are hour, day, month, and weekday.

3. Use `crontab -l` to view the current crontab for the current user. For another user, use `crontab -u username -l`.

4. Use `sudo systemctl enable mytimer.timer` to enable a timer at boot. This creates a symlink from the timer unit to `timers.target.wants/`.

5. The `Persistent` option ensures that if a timer is missed (e.g., system was down), the timer will run when the system comes back up. It queues up missed executions.

6. Use `sudo systemctl set-default multi-user.target` to change the default boot target. This creates a symlink from `/etc/systemd/system/default.target` to `multi-user.target`.

7. Use `chronyc tracking` or `chronyc adjust -s 100` to synchronize time with chrony. The system automatically synchronizes when chronyd is running.

8. Use `sudo dnf install -y package` to install without interactive prompts. The `-y` flag automatically answers "yes" to all questions.

9. `GRUB_TIMEOUT` sets the number of seconds to display the GRUB boot menu before automatically booting the default entry.

10. Use `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` to regenerate the GRUB configuration file after modifying settings.

11. Use `systemctl status service_name` to check the status of a systemd service. This shows whether the service is active, inactive, or failed.

12. Use `sudo systemctl enable service_name` to enable a service at boot. This creates a symlink to make the service start automatically when the system reaches the appropriate target.

13. `systemctl start service` starts the service immediately. `systemctl enable service` creates a symlink so the service starts automatically at the next boot. Use both for persistent services.

14. Use `chronyc tracking` to view NTP synchronization status. It shows the offset, jitter, and synchronization state. Use `chronyc sources` to view NTP server details.

15. Use `dnf check-update` to list all available updates without installing them. Use `dnf list updates` for a more detailed list.