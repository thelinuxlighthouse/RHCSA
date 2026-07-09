# Chapter 6: Creating and Configuring File Systems

## Learning Objectives

By the end of this chapter, you will be able to:

- Create, mount, unmount, and use VFAT, ext4, and XFS file systems
- Configure filesystem mount options and permissions
- Mount and unmount network file systems using NFS
- Configure autofs for on-demand mounting
- Extend existing logical volumes and filesystems
- Diagnose and correct file permission problems
- Understand filesystem quotas and disk usage monitoring

---

## Concepts and Theory

### Linux Filesystem Types

**ext4 (Fourth Extended Filesystem):**

- Default filesystem for RHEL 8
- Journaling filesystem for crash recovery
- Supports files up to 16 TB
- Supports volumes up to 1 EB (exabyte)
- Block sizes: 1K, 2K, 4K (default)
- File count: 2 billion
- Inode count: 1 billion

**XFS:**

- Default filesystem for RHEL 9 and 10
- Designed for large files and high performance
- Scalable to multi-terabyte volumes
- Automatic journaling
- Block sizes: 512B, 1K, 2K, 4K (default 512B)
- File count: 16 billion
- Inode count: 1 billion

**VFAT (Virtual FAT):**

- FAT32-compatible filesystem
- Used for USB drives, removable media
- Maximum file size: 4 GB
- Maximum volume size: 32 GB (FAT32)
- Maximum filename length: 255 characters
- Case-insensitive, 8.3 naming convention

**Btrfs:**

- Copy-on-write filesystem
- Snapshots and clones
- Data integrity features
- Default for newer Fedora, not RHEL

### Filesystem Structure

**ext4 Structure:**

- **Superblock**: Filesystem metadata (location, size, features)
- **Group Descriptors**: Block group information
- **Inode Table**: File metadata and pointers
- **Data Blocks**: Actual file data
- **Journal**: Transaction log for crash recovery
- **Block Bitmap**: Free block tracking
- **Inode Bitmap**: Free inode tracking

**XFS Structure:**

- **Superblock**: Filesystem metadata
- **Log**: Transaction log
- **Allocation Groups**: Scalable block allocation
- **Inodes**: File metadata
- **Data Fork**: File data
- **Attribute Fork**: Extended attributes

### Mount Options

**Common Mount Options:**

| Option | Description |
|--------|-------------|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `rw` | Read-write access |
| `ro` | Read-only access |
| `noexec` | No executable files |
| `nosuid` | Ignore setuid bits |
| `nodev` | Ignore device files |
| `noatime` | Don't update access time |
| `relatime` | Update atime only if mtime/ctime changed |
| `sync` | Synchronous writes |
| `async` | Asynchronous writes (default) |
| `soft` | Soft mount (NFS) |
| `hard` | Hard mount (NFS) |
| `nofail` | Don't fail boot if mount fails |
| `user` | Allow user to mount |
| `users` | Allow any user to mount |
| `uid` | Set file owner UID |
| `gid` | Set file owner GID |
| `umask` | Set default permissions |

### Filesystem Quotas

**Quota Types:**

- **Block Quota**: Limits disk space usage
- **Inode Quota**: Limits number of files
- **User Quota**: Per-user limits
- **Group Quota**: Per-group limits

**Quota Tools:**

- `quotacheck`: Scan filesystem for quotas
- `quotaon`: Enable quotas
- `quotaoff`: Disable quotas
- `repquota`: Report quota usage
- `edquota`: Edit quotas interactively
- `setquota`: Set quotas from command line

### Filesystem Monitoring

**Disk Usage:**

- `df`: Disk free space
- `du`: Disk usage of files/directories
- `ncdu`: Interactive disk usage analyzer

**Filesystem Health:**

- `xfs_info`: XFS filesystem information
- `dumpe2fs`: ext4 filesystem information
- `tune2fs`: ext4 filesystem tuning

---

## How It Works Internally

### ext4 Filesystem Creation

When creating an ext4 filesystem:

1. **Superblock** is written with filesystem parameters
2. **Group descriptors** are created for each block group
3. **Inode tables** are allocated
4. **Block and inode bitmaps** are created
5. **Journal** is initialized
6. **Block groups** are formatted

The ext4 superblock contains:
- Filesystem size and block count
- Inode count and block size
- Mount count and maximum mount count
- Last mount time and UUID
- Feature flags (extent, journal, etc.)

### XFS Filesystem Creation

When creating an XFS filesystem:

1. **Superblock** is written with filesystem parameters
2. **Log** is created for transaction management
3. **Allocation groups** are initialized
4. **Ag headers** are created
5. **Inodes** are pre-allocated
6. **Block and inode maps** are created

XFS uses delayed allocation (delayed write):
- Data is written to disk when full, not immediately
- Improves performance for large sequential writes
- Reduces disk I/O overhead

### Mount Process Details

When mounting a filesystem:

1. **Device check**: Verify device is accessible
2. **Mount point check**: Ensure directory exists
3. **Superblock read**: Read filesystem metadata
4. **Mount options apply**: Apply permissions and options
5. **VFS mapping**: Map filesystem to VFS layer
6. **Mount status**: Record in /proc/mounts and /etc/mtab

The VFS (Virtual Filesystem) layer provides a uniform interface to all filesystem types.

### NFS Mount Process

When mounting NFS:

1. **Client discovers server** via hostname or IP
2. **RPC connection established** to portmapper (port 111)
3. **Export lookup** requests available exports
4. **Mount request** sent to NFS daemon
5. **Filesystem mounted** with specified options
6. **Data transferred** over RPC/NFS protocol

NFS uses RPC (Remote Procedure Call):
- Client sends RPC requests to server
- Server responds with results
- Multiple RPC services run on NFS server

### Autofs Operation

Autofs works by:

1. **Master configuration** defines mount points
2. **Map files** define specific mounts
3. **Kernel autofs daemon** monitors access
4. **On-demand mount** when path accessed
5. **Automatic unmount** after timeout
6. **Cache** for recently accessed mounts

Autofs uses the autofs kernel module:
- Mounts filesystems on demand
- Unmounts when idle (configurable timeout)
- Reduces resource usage

### File Permission Checks

When accessing a file:

1. **Check if root**: Root bypasses most checks
2. **Check owner**: If UID matches, check user permissions
3. **Check group**: If GID matches, check group permissions
4. **Check others**: Otherwise, check others permissions
5. **Check special bits**: Apply setuid, setgid, sticky

For directories:
- **Read**: List directory contents
- **Write**: Create/delete files in directory
- **Execute**: Enter directory

---

## Commands and Administration Tasks

### Creating Filesystems

#### ext4 Filesystem

```bash
# Create ext4 filesystem
sudo mkfs.ext4 /dev/vg0/lv_home

# Create with label
sudo mkfs.ext4 -L home /dev/vg0/lv_home

# Create with options
sudo mkfs.ext4 -b 4096 -I 4096 /dev/vg0/lv_home
# -b: Block size
# -I: Inode size

# Create with journal size
sudo mkfs.ext4 -J journal_size=64M /dev/vg0/lv_home

# View filesystem info
sudo dumpe2fs /dev/vg0/lv_home | head -30

# Check filesystem
sudo tune2fs -l /dev/vg0/lv_home
```

#### XFS Filesystem

```bash
# Create XFS filesystem
sudo mkfs.xfs /dev/vg0/lv_data

# Create with label
sudo mkfs.xfs -L data /dev/vg0/lv_data

# Create with options
sudo mkfs.xfs -f -s size=1m /dev/vg0/lv_data
# -f: Force creation
# -s: Summary size

# View filesystem info
sudo xfs_info /dev/vg0/lv_data

# Check filesystem
sudo xfs_db -r -c "info" /dev/vg0/lv_data
```

#### VFAT Filesystem

```bash
# Create VFAT filesystem
sudo mkfs.vfat /dev/sdb1

# Create with options
sudo mkfs.vfat -F 32 -n "USB Drive" /dev/sdb1
# -F: FAT version
# -n: Volume label

# View filesystem info
sudo blkid /dev/sdb1

# Check filesystem
sudo dosfsck /dev/sdb1
```

### Mounting Filesystems

#### Manual Mounting

```bash
# Mount filesystem
sudo mount /dev/vg0/lv_home /home

# Mount with options
sudo mount -o rw,noatime /dev/vg0/lv_data /data

# Mount with label
sudo mount -o defaults,LABEL=home /dev/sda3 /mnt/home

# Mount with UUID
sudo mount -o defaults,UUID=abc123 /dev/sda3 /mnt/home

# Mount ext4 filesystem
sudo mount /dev/vg0/lv_home /mnt/ext4

# Mount XFS filesystem
sudo mount /dev/vg0/lv_xfs /mnt/xfs

# Mount VFAT filesystem
sudo mount -t vfat /dev/sdb1 /mnt/usb

# Unmount filesystem
sudo umount /home

# Force unmount
sudo umount -f /home

# Lazy unmount
sudo umount -l /home

# Check mount status
mount | grep /home
cat /proc/mounts | grep /home
```

#### Mounting Filesystems at Boot

```bash
# View current /etc/fstab
cat /etc/fstab

# Add entry to /etc/fstab
echo 'UUID=abc123def4:/home /home ext4 defaults 0 2' | sudo tee -a /etc/fstab

# Test fstab without reboot
sudo mount -a

# View fstab entries
cat /etc/fstab

# Edit fstab
sudo vi /etc/fstab

# Example /etc/fstab entries:
# UUID=abc123def4:/boot /boot xfs defaults 0 2
# UUID=def4567890:/ / xfs defaults 0 1
# UUID=7890123456 swap swap defaults 0 0
# LABEL=home /home ext4 defaults 0 2
# UUID=3456789012 /data xfs defaults,noatime 0 2
```

### Filesystem Options

#### ext4 Options

```bash
# Mount with noatime
sudo mount -o noatime /dev/vg0/lv_home /home

# Mount with nobarrier
sudo mount -o nobarrier /dev/vg0/lv_home /home

# Mount with journal mode
sudo mount -o journal_data=ordered /dev/vg0/lv_home /home

# Set mount options permanently
echo 'defaults,noatime' | sudo tee /etc/fstab.fstab

# Resize ext4 filesystem
sudo resize2fs /dev/vg0/lv_home

# Check ext4 filesystem
sudo e2fsck -f /dev/vg0/lv_home

# Tune ext4 filesystem
sudo tune2fs -c 30 /dev/vg0/lv_home  # Check after 30 mounts
sudo tune2fs -i 7d /dev/vg0/lv_home  # Check every 7 days
```

#### XFS Options

```bash
# Mount with noatime
sudo mount -o noatime /dev/vg0/lv_data /data

# Mount with nodiratime
sudo mount -o nodiratime /dev/vg0/lv_data /data

# Resize XFS filesystem
sudo xfs_growfs /data

# Check XFS filesystem
sudo xfs_repair -n /dev/vg0/lv_data

# View XFS info
sudo xfs_info /data

# Set mount options permanently
echo 'defaults,noatime,nodiratime' | sudo tee -a /etc/fstab
```

### Unmounting Filesystems

```bash
# Unmount filesystem
sudo umount /home

# Check if mounted
mount | grep /home

# Force unmount
sudo umount -f /home

# Lazy unmount (defer unmounting)
sudo umount -l /home

# Unmount all filesystems
sudo umount -a

# Unmount from /etc/fstab
sudo umount -a -f

# Check mount points
df -h
mount | grep -E "ext4|xfs|vfat"
```

### NFS Configuration

#### NFS Server Setup

```bash
# Install NFS server
sudo dnf install -y nfs-utils

# Create export directory
sudo mkdir -p /export/data
sudo mkdir -p /export/home

# Set permissions
sudo chown -R nobody:nogroup /export/data
sudo chmod -R 755 /export/data

# Configure exports
sudo vi /etc/exports

# Example /etc/exports:
# /export/data 192.168.1.0/24(rw,sync,no_root_squash,sec=sys)
# /export/home *(ro,sync,root_squash,sec=sys)
# /export/backup 10.0.0.0/8(rw,sync,root_squash,sec=sys)

# Export filesystems
sudo exportfs -a

# Restart NFS service
sudo systemctl restart nfs-server

# Enable NFS at boot
sudo systemctl enable nfs-server

# View exports
cat /etc/exports
exportfs -v

# Show NFS statistics
nfsstat

# Check NFS status
systemctl status nfs-server
```

#### NFS Client Mounting

```bash
# Mount NFS temporarily
sudo mount -t nfs 192.168.1.10:/export/data /mnt/nfs

# Mount NFS with options
sudo mount -t nfs 192.168.1.10:/export/data /mnt/nfs -o rw,noatime

# Mount NFS by hostname
sudo mount -t nfs server:/export/data /mnt/nfs

# Mount NFS with version
sudo mount -t nfs -o vers=3 192.168.1.10:/export/data /mnt/nfs

# Mount NFS with timeout
sudo mount -t nfs -o soft,timeo=30,retrans=2 192.168.1.10:/export/data /mnt/nfs

# Add NFS to /etc/fstab
echo '192.168.1.10:/export/data /mnt/nfs nfs defaults 0 0' | sudo tee -a /etc/fstab

# Unmount NFS
sudo umount /mnt/nfs

# Check NFS mounts
mount | grep nfs
df -T | grep nfs
```

### Autofs Configuration

#### Autofs Installation

```bash
# Install autofs
sudo dnf install -y autofs

# Create mount points
sudo mkdir -p /mnt/nfs
sudo mkdir -p /mnt/data

# Create autofs configuration
sudo vi /etc/auto.master
```

#### Autofs Configuration

```bash
# Edit auto.master
sudo vi /etc/auto.master

# Add entry:
# /mnt/nfs /etc/auto.nfs --timeout=60
# /mnt/data /etc/auto.data --timeout=300

# Edit auto.nfs
sudo vi /etc/auto.nfs

# Add mount entries:
# data -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/export/data
# home -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/export/home

# Edit auto.data
sudo vi /etc/auto.data

# Add mount entries:
# backup -fstype=ext4,defaults  /dev/vg0/lv_backup
# archive -fstype=xfs,defaults  /dev/vg0/lv_archive

# Restart autofs
sudo systemctl restart autofs

# Enable autofs at boot
sudo systemctl enable autofs

# Start autofs
sudo systemctl start autofs

# Test autofs
touch /mnt/nfs/data/test.txt
ls /mnt/nfs/data/
sleep 60
ls /mnt/nfs/  # Should be empty
```

### Extending Logical Volumes

#### Extending LV and Filesystem

```bash
# Check current LV size
sudo lvdisplay /dev/vg0/lv_home
sudo lvs -o lv_size /dev/vg0/lv_home

# Extend logical volume
sudo lvextend -L +10G /dev/vg0/lv_home
sudo lvextend -l +100%FREE /dev/vg0/lv_home
sudo lvextend -L 50G /dev/vg0/lv_home

# Resize ext4 filesystem
sudo resize2fs /dev/vg0/lv_home
sudo resize2fs -M /dev/vg0/lv_home  # Resize to minimum

# Resize XFS filesystem
sudo xfs_growfs /home
sudo xfs_growfs /dev/vg0/lv_home  # Use mount point or device

# View changes
df -h /home
sudo lvdisplay /dev/vg0/lv_home

# Shrink logical volume (requires unmount)
sudo umount /home
sudo lvreduce -L -5G /dev/vg0/lv_home
sudo resize2fs /dev/vg0/lv_home

# View VG free space
sudo vgs
sudo vgdisplay -o vg_free
```

### File Permission Diagnosis

#### Diagnosing Permission Problems

```bash
# View file permissions
ls -la /path/to/file

# View detailed permissions
stat /path/to/file

# Check ownership
ls -ln /path/to/file

# Check directory permissions for access
ls -ld /path/to/directory

# View access attempts
sudo auditctl -w /path/to/file -p wa -k access

# Check SELinux context
ls -Z /path/to/file

# Check for ACLs
getfacl /path/to/file

# Find permission denied errors
grep "Permission denied" /var/log/messages

# Find files with wrong ownership
find /path -user nobody -ls

# Find files with wrong permissions
find /path -perm 777 -ls

# Check mount options
mount | grep /path

# Check if directory is read-only
df -T /path
```

#### Correcting Permission Problems

```bash
# Change file owner
sudo chown user:group /path/to/file

# Change permissions
sudo chmod 755 /path/to/directory
sudo chmod 644 /path/to/file

# Set setuid bit
sudo chmod u+s /path/to/file

# Set setgid bit
sudo chmod g+s /path/to/directory

# Set sticky bit
sudo chmod +t /path/to/directory

# Fix directory permissions
sudo chmod -R 755 /path/to/directory

# Fix ownership recursively
sudo chown -R user:group /path/to/directory

# Fix permissions recursively
sudo chmod -R 755 /path/to/directory

# Remove setuid/setgid bits
sudo chmod u-s,g-s /path/to/file

# Remove sticky bit
sudo chmod -t /path/to/directory

# Check for immutable attribute
lsattr /path/to/file

# Remove immutable flag
sudo chattr -i /path/to/file

# Add immutable flag
sudo chattr +i /path/to/file

# Check ACLs
getfacl /path/to/file

# Set ACL
setfacl -m u:user:rwx /path/to/file
setfacl -d -m u:user:rwx /path/to/file  # Default ACL

# Clear ACLs
setfacl -b /path/to/file

# Fix SELinux context
sudo restorecon -Rv /path/to/file

# Check SELinux mode
getenforce

# Set SELinux to permissive
sudo setenforce 0

# Restore SELinux context
sudo restorecon -Rv /path/to/file

# Restore SELinux enforcing mode
sudo setenforce 1
```

---

## Configuration Examples

### /etc/fstab Configuration

```bash
# /etc/fstab - Filesystem table
# Device           Mount Point    Type    Options              Dump Pass
UUID=abc123def4   /boot          xfs     defaults             0 2
UUID=def4567890   /              xfs     defaults             0 1
UUID=7890123456   swap           swap    defaults             0 0
UUID=3456789012   /home          ext4    defaults,noatime     0 2
LABEL=home        /home          ext4    defaults,noatime     0 2
192.168.1.10:/export/data /mnt/nfs   nfs     defaults,soft        0 0
/dev/vg0/lv_data  /data          xfs     defaults,noatime     0 2
```

### ext4 Filesystem Options

```bash
# /etc/fstab with ext4 options
# UUID=3456789012 /home ext4 defaults,noatime,nodiratime,errors=remount-ro 0 2

# Mount options explained:
# defaults = rw,suid,dev,exec,auto,nouser,async
# noatime = Don't update access time
# nodiratime = Don't update directory access time
# errors=remount-ro = Remount read-only on error
```

### XFS Filesystem Options

```bash
# /etc/fstab with XFS options
# UUID=3456789012 /data xfs defaults,noatime,nodiratime,inode64 0 2

# XFS specific options:
# noatime = Don't update access time
# nodiratime = Don't update directory access time
# inode64 = Use 64-bit inode numbers
```

### Autofs Configuration

```bash
# /etc/auto.master
/mnt/nfs /etc/auto.nfs --timeout=60
/mnt/data /etc/auto.data --timeout=300
/mnt/backup /etc/auto.backup --timeout=120

# /etc/auto.nfs
data -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/export/data
home -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/export/home
web -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.20:/var/www

# /etc/auto.data
backup -fstype=ext4,defaults /dev/vg0/lv_backup
archive -fstype=xfs,defaults /dev/vg0/lv_archive

# /etc/auto.backup
daily -fstype=ext4,defaults /dev/vg0/lv_daily
weekly -fstype=ext4,defaults /dev/vg0/lv_weekly
```

### NFS Server Configuration

```bash
# /etc/exports
/export/data 192.168.1.0/24(rw,sync,no_root_squash,sec=sys,no_subtree_check)
/export/home *(ro,sync,root_squash,sec=sys)
/export/backup 10.0.0.0/8(rw,sync,root_squash,sec=sys,all_squash,anonuid=65534,anongid=65534)
/export/public *(rw,async,all_squash,anonuid=65534,anongid=65534,anonroot=0)
```

### File Quota Configuration

```bash
# /etc/fstab with quota options
# UUID=3456789012 /home ext4 defaults,usrquota,grpquota 0 2

# Enable quotas
sudo quotacheck -cvug /home
sudo quotaon /home

# View quotas
sudo repquota -av /home

# Edit quotas
sudo edquota -u username
sudo edquota -g groupname

# Set quotas from command line
sudo setquota -u username 10G 20G 5000 10000 /home
# Block soft, Block hard, Inode soft, Inode hard
```

---

## Real-World Examples

### Scenario 1: Setting Up a Multi-Tier Web Server

A web server needs separate filesystems for web content, logs, and databases.

```bash
# Create partition layout
sudo gdisk /dev/sdb
# Part 1: 20GB /var/www (ext4)
# Part 2: 10GB /var/log (ext4)
# Part 3: 30GB /var/lib/mysql (XFS)

# Create LVM
sudo pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb3
sudo vgcreate webvg /dev/sdb1 /dev/sdb2 /dev/sdb3

# Create logical volumes
sudo lvcreate -L 20G -n lv_www webvg
sudo lvcreate -L 10G -n lv_logs webvg
sudo lvcreate -L 30G -n lv_mysql webvg

# Create filesystems
sudo mkfs.ext4 -L www /dev/webvg/lv_www
sudo mkfs.ext4 -L logs /dev/webvg/lv_logs
sudo mkfs.xfs -L mysql /dev/webvg/lv_mysql

# Create mount points
sudo mkdir -p /var/www /var/log/webserver /var/lib/mysql

# Mount filesystems
sudo mount /dev/webvg/lv_www /var/www
sudo mount /dev/webvg/lv_logs /var/log/webserver
sudo mount /dev/webvg/lv_mysql /var/lib/mysql

# Add to /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_www | grep UUID | cut -d= -f2) /var/www ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_logs | grep UUID | cut -d= -f2) /var/log/webserver ext4 defaults 0 2' | sudo tee -a /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_mysql | grep UUID | cut -d= -f2) /var/lib/mysql xfs defaults,noatime 0 2' | sudo tee -a /etc/fstab
```

### Scenario 2: Setting Up Shared Storage with NFS and Autofs

Multiple servers need to access shared files with on-demand mounting.

```bash
# Server: Set up NFS server
sudo dnf install -y nfs-utils
sudo mkdir -p /shared/data /shared/home
sudo chown -R nobody:nogroup /shared/data
sudo chmod -R 755 /shared/data

# Configure exports
echo '/shared/data 192.168.1.0/24(rw,sync,no_root_squash,sec=sys)' | sudo tee -a /etc/exports
echo '/shared/home *(ro,sync,root_squash,sec=sys)' | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-server
sudo systemctl enable nfs-server

# Client: Configure autofs
sudo dnf install -y autofs
sudo mkdir -p /mnt/shared

# Configure auto.master
echo '/mnt/shared /etc/auto.shared --timeout=120' | sudo tee -a /etc/auto.master

# Configure auto.shared
cat > /etc/auto.shared << 'EOF'
data -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/shared/data
home -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/shared/home
backup -fstype=nfs,soft,timeo=30,retrans=2,vers=3 192.168.1.10:/shared/backup
EOF

# Start and enable autofs
sudo systemctl restart autofs
sudo systemctl enable autofs

# Test autofs
touch /mnt/shared/data/test.txt
ls /mnt/shared/data/
# After timeout, mount point should be unmounted
```

### Scenario 3: Setting Up Filesystem Quotas

A company needs to limit user disk usage on a shared filesystem.

```bash
# Enable quota support in /etc/fstab
echo 'UUID=3456789012 /home ext4 defaults,usrquota,grpquota 0 2' | sudo tee -a /etc/fstab

# Remount with quota options
sudo mount -o remount,usrquota,grpquota /home

# Scan filesystem for quotas
sudo quotacheck -cvug /home

# Enable quotas
sudo quotaon /home

# View current quotas
sudo repquota -av /home

# Edit user quotas interactively
sudo edquota -u alice
# Set limits:
# soft limit: 10G
# hard limit: 15G
# inode soft: 10000
# inode hard: 20000

# Edit group quotas
sudo edquota -g developers

# Set quotas from command line
sudo setquota -u alice 10G 15G 10000 20000 /home
sudo setquota -u bob 5G 10G 5000 10000 /home
sudo setquota -g developers 20G 30G 20000 50000 /home

# View quota usage
sudo repquota -av /home

# Check if user is over quota
quota -u alice
```

### Scenario 4: Troubleshooting NFS Mount Issues

A client cannot mount an NFS share.

```bash
# Check NFS server is running
ssh server "systemctl status nfs-server"

# Check exports configuration
ssh server "cat /etc/exports"
ssh server "exportfs -v"

# Check firewall
ssh server "firewall-cmd --list-all"

# Check network connectivity
ping server

# Check client mount options
mount | grep nfs

# Test NFS mount
sudo mount -t nfs server:/export/data /mnt/nfs

# Check NFS statistics
nfsstat

# Check RPC services
rpcinfo -p server

# Check /var/log/messages for errors
grep nfs /var/log/messages | tail -20

# Try with different options
sudo mount -t nfs -o soft,timeo=30,retrans=2 server:/export/data /mnt/nfs

# Check SELinux
getenforce
ls -Z /mnt/nfs

# Fix SELinux context
sudo restorecon -Rv /mnt/nfs
```

---

## Troubleshooting

### Filesystem Mount Issues

**Problem**: Filesystem won't mount

```bash
# Check if filesystem is mounted
mount | grep /dev/vg0/lv_home

# Check /etc/fstab
cat /etc/fstab

# Check filesystem type
sudo blkid /dev/vg0/lv_home

# Test filesystem
sudo xfs_repair -n /dev/vg0/lv_home
# or
sudo fsck.ext4 -f /dev/vg0/lv_home

# Try mounting with verbose
sudo mount -v /dev/vg0/lv_home /mnt

# Check SELinux
getenforce
setenforce 0
sudo mount /dev/vg0/lv_home /mnt
setenforce 1

# Check mount point exists
ls -ld /mnt

# Check permissions on mount point
ls -la /mnt
```

**Problem**: NFS mount fails

```bash
# Check NFS server is running
ssh server "systemctl status nfs-server"

# Check exports configuration
ssh server "cat /etc/exports"

# Check firewall
ssh server "firewall-cmd --list-all"

# Check network connectivity
ping server

# Check client mount options
mount | grep nfs

# Test NFS mount
sudo mount -t nfs server:/export/data /mnt/nfs

# Check NFS statistics
nfsstat

# Check RPC services
rpcinfo -p server

# Check /var/log/messages for errors
grep nfs /var/log/messages | tail -20

# Try with different options
sudo mount -t nfs -o soft,timeo=30,retrans=2 server:/export/data /mnt/nfs

# Check SELinux
getenforce
ls -Z /mnt/nfs
```

### Filesystem Extension Issues

**Problem**: Cannot extend logical volume

```bash
# Check available space in VG
sudo vgs
sudo vgdisplay -o vg_free

# Check if PV is added to VG
sudo pvs
sudo pvdisplay

# Check LV status
sudo lvs

# Check if LV is mounted
mount | grep /dev/vg0/lv_home

# Extend LV
sudo lvextend -L +10G /dev/vg0/lv_home

# Resize filesystem
sudo xfs_growfs /home
# or
sudo resize2fs /dev/vg0/lv_home

# View changes
df -h /home
```

**Problem**: Cannot resize XFS filesystem

```bash
# Check XFS filesystem info
sudo xfs_info /home

# Check if mounted
mount | grep /home

# Resize XFS (must use mount point, not device)
sudo xfs_growfs /home

# Verify resize
sudo xfs_info /home
df -h /home
```

### Permission Issues

**Problem**: Cannot access directory

```bash
# Check directory permissions
ls -ld /path/to/directory

# Check ownership
ls -ln /path/to/directory

# Check if directory is mounted
mount | grep /path

# Check SELinux context
ls -Z /path/to/file

# Check for ACLs
getfacl /path/to/file

# Fix permissions
sudo chmod 755 /path/to/directory
sudo chown user:group /path/to/directory

# Fix SELinux context
sudo restorecon -Rv /path/to/directory

# Fix ACLs
setfacl -m u:user:rwx /path/to/file
```

**Problem**: Cannot write to directory

```bash
# Check write permission
ls -ld /path/to/directory

# Check directory is not read-only
df -T /path

# Check filesystem is not mounted read-only
mount | grep /path

# Fix permissions
sudo chmod +w /path/to/directory

# Fix ownership
sudo chown user:group /path/to/directory

# Check for immutable attribute
lsattr /path/to/file

# Remove immutable flag
sudo chattr -i /path/to/file

# Check SELinux context
ls -Z /path/to/file
```

### Autofs Issues

**Problem**: Autofs not mounting

```bash
# Check autofs status
systemctl status autofs

# Check autofs configuration
cat /etc/auto.master
cat /etc/auto.nfs

# Check mount points
ls -la /mnt/nfs

# Restart autofs
sudo systemctl restart autofs

# Test autofs
touch /mnt/nfs/data/test.txt
ls /mnt/nfs/data/
sleep 60
ls /mnt/nfs/  # Should be empty

# Check autofs logs
journalctl -u autofs -f
```

### Filesystem Corruption Issues

**Problem**: Filesystem errors detected

```bash
# Check filesystem status
sudo xfs_repair -n /dev/vg0/lv_home
# or
sudo fsck.ext4 -f /dev/vg0/lv_home

# View dmesg for errors
dmesg | grep -i error
dmesg | grep -i filesystem

# Check journal for errors
journalctl -k | grep -i filesystem

# Unmount filesystem
sudo umount /dev/vg0/lv_home

# Repair filesystem
sudo xfs_repair /dev/vg0/lv_home
# or
sudo fsck.ext4 -y /dev/vg0/lv_home

# Remount filesystem
sudo mount /dev/vg0/lv_home /mnt
```

---

## Verification Procedures

### Filesystem Creation Verification

```bash
# Verify ext4 filesystem
sudo dumpe2fs /dev/vg0/lv_home | head -30
sudo tune2fs -l /dev/vg0/lv_home

# Verify XFS filesystem
sudo xfs_info /dev/vg0/lv_data

# Verify VFAT filesystem
sudo blkid /dev/sdb1
sudo dosfsck /dev/sdb1
```

### Mount Verification

```bash
# Verify mounts
mount
cat /proc/mounts

# Verify /etc/fstab
cat /etc/fstab

# Verify autofs
systemctl status autofs

# Test autofs
touch /mnt/nfs/data/test.txt
ls /mnt/nfs/data/
```

### NFS Verification

```bash
# Verify NFS exports
exportfs -v
cat /etc/exports

# Verify NFS mount
mount | grep nfs

# Verify NFS statistics
nfsstat

# Test NFS connectivity
ssh server "showmount -e server"
```

### LVM Verification

```bash
# Verify physical volumes
sudo pvs
sudo pvdisplay

# Verify volume groups
sudo vgs
sudo vgdisplay

# Verify logical volumes
sudo lvs
sudo lvdisplay

# Verify LV metadata
sudo vgck vg0
```

### Permission Verification

```bash
# Verify permissions
ls -la /path/to/file
stat /path/to/file

# Verify ownership
ls -ln /path/to/file

# Verify SELinux context
ls -Z /path/to/file

# Verify ACLs
getfacl /path/to/file
```

### Quota Verification

```bash
# Verify quotas enabled
quotaon /home

# View quotas
sudo repquota -av /home

# Check quota usage
quota -u username
quota -g groupname
```

---

## RHCSA Exam Notes

### Exam Relevance

Creating and configuring filesystems is heavily tested on the RHCSA. Expect 2-3 questions covering filesystem creation, mounting, NFS configuration, and permission troubleshooting.

### Key Exam Points

1. **Filesystem Creation**: Know `mkfs.ext4`, `mkfs.xfs`, `mkfs.vfat` and their options.

2. **Mounting**: Know how to mount by UUID, label, and device name. Know how to add to `/etc/fstab`.

3. **NFS**: Know how to configure NFS server and client, and mount NFS shares.

4. **Autofs**: Know how to configure autofs for on-demand mounting.

5. **File Permissions**: Know how to diagnose and fix permission problems using `chmod`, `chown`, `chattr`, `restorecon`.

6. **Filesystem Extension**: Know how to extend LVM and resize filesystems (especially XFS online resize).

### Common Exam Traps

- Using device names in `/etc/fstab` instead of UUIDs
- Forgetting to resize the filesystem after extending the LV
- Not using the correct mount options for XFS
- Forgetting to enable autofs service
- Using `umount` when filesystem is busy (use `umount -l`)
- Confusing ext4 and XFS resize commands

### Time Management Tips

- Use `blkid` to quickly get UUIDs for /etc/fstab
- Use `df -T` to check mounted filesystems
- Use `mount | grep` to find specific mounts
- Use `xfs_growfs /mount/point` for XFS resize (not device)
- Use `resize2fs /device` for ext4 resize

### Exam Environment Notes

- You will have access to LVM tools
- NFS utilities may need to be installed
- UUIDs can be obtained with `blkid`
- Autofs may require manual configuration
- Filesystem types must be specified for some mounts

---

## Chapter Summary

This chapter covered filesystem creation and configuration including ext4, XFS, and VFAT filesystems, NFS mounting and configuration, autofs setup, logical volume extension, and file permission troubleshooting. You learned how to create various filesystem types, mount them manually and at boot, configure network file sharing with NFS, set up on-demand mounting with autofs, extend storage without downtime, and diagnose and fix permission problems.

These skills are essential for managing disk storage on RHEL systems. Understanding filesystem creation, mounting, and configuration helps administrators optimize storage usage, share files across systems, and troubleshoot common storage issues.

---

## Quick Reference

### Filesystem Creation

```bash
mkfs.ext4 /dev/vg0/lv1       # Create ext4
mkfs.ext4 -L label /dev/vg0/lv1  # Create ext4 with label
mkfs.xfs /dev/vg0/lv1        # Create XFS
mkfs.xfs -L label /dev/vg0/lv1  # Create XFS with label
mkfs.vfat -F 32 /dev/sdb1    # Create VFAT
```

### Filesystem Info

```bash
dumpe2fs /dev/vg0/lv1        # ext4 info
tune2fs -l /dev/vg0/lv1      # ext4 details
xfs_info /mount/point        # XFS info
xfs_db -r -c "info" /dev/vg0/lv1  # XFS details
```

### Mount Commands

```bash
mount /dev/vg0/lv1 /mnt      # Mount by device
mount -a                     # Mount from /etc/fstab
mount -o UUID=abc /dev/sda1 /mnt  # Mount by UUID
mount -o LABEL=home /dev/sda1 /mnt  # Mount by label
umount /mnt                  # Unmount
umount -f /mnt               # Force unmount
umount -l /mnt               # Lazy unmount
```

### NFS Commands

```bash
exportfs -a                  # Export all
exportfs -v                  # List exports
showmount -e server          # View exports
mount -t nfs server:/export /mnt  # Mount NFS
nfsstat                      # NFS statistics
```

### Autofs Commands

```bash
vi /etc/auto.master          # Edit master config
vi /etc/auto.nfs             # Edit mount points
systemctl restart autofs     # Restart autofs
```

### Filesystem Resize

```bash
resize2fs /dev/vg0/lv1       # Resize ext4
xfs_growfs /mount/point      # Resize XFS
lvextend -L +10G /dev/vg0/lv1  # Extend LV
```

### Permission Commands

```bash
chmod 755 /path              # Change permissions
chown user:group /path       # Change ownership
ls -la /path                 # List permissions
stat /path                   # Detailed info
getfacl /path                # View ACLs
setfacl -m u:user:rwx /path  # Set ACL
```

### Quota Commands

```bash
quotacheck -cvug /path       # Scan for quotas
quotaon /path                # Enable quotas
quotaoff /path               # Disable quotas
repquota -av /path           # Report quotas
edquota -u user              # Edit user quota
setquota -u user ... /path   # Set quota command line
```

---

## Review Questions

1. What is the difference between ext4 and XFS filesystems?

2. How do you create an ext4 filesystem with a label?

3. What command would you use to mount a filesystem by UUID?

4. How do you add a filesystem to /etc/fstab to mount at boot?

5. What is the advantage of using autofs over manual mounting?

6. How do you configure a directory to be exported via NFS?

7. What command would you use to resize an XFS filesystem?

8. How do you extend a logical volume and resize the filesystem at the same time?

9. What is the difference between soft and hard NFS mounts?

10. How do you set up filesystem quotas for users and groups?

11. What command would you use to check if a filesystem is mounted?

12. How do you diagnose a permission problem where a user cannot access a file?

13. What is the purpose of the noatime mount option?

14. How do you unmount a filesystem that is busy?

15. What command would you use to repair a corrupted ext4 filesystem?

---

## Answers

1. ext4 is the default filesystem for RHEL 8 with support for files up to 16 TB and volumes up to 1 EB. XFS is the default for RHEL 9 and 10, designed for large files and high performance with automatic journaling and scalable block allocation. XFS can be resized online without unmounting, while ext4 requires unmounting for resize.

2. Use `mkfs.ext4 -L label /dev/vg0/lv1` to create an ext4 filesystem with a label. The `-L` option sets the filesystem label, which can be used for mounting instead of UUIDs.

3. Use `mount -o UUID=abc123 /dev/sda1 /mnt` to mount a filesystem by UUID. The UUID is obtained from `blkid /dev/sda1`. Alternatively, add the UUID to `/etc/fstab` and run `mount -a`.

4. Add an entry to `/etc/fstab` with the format: `UUID=abc123 /mount/point filesystem_type options dump pass`. For example: `UUID=abc123 /home ext4 defaults 0 2`. Then run `mount -a` to test the configuration.

5. Autofs mounts filesystems only when they are accessed, reducing resource usage and improving security. It automatically unmounts filesystems when idle (configurable timeout), which is ideal for network filesystems that are accessed infrequently.

6. Add an entry to `/etc/exports`: `/export/path user_or_network(options)`. For example: `/export/data 192.168.1.0/24(rw,sync,no_root_squash)`. Then run `exportfs -a` and restart the NFS service.

7. Use `xfs_growfs /mount/point` to resize an XFS filesystem. XFS requires the mount point, not the device. The filesystem can be resized online without unmounting.

8. Use `lvextend -L +10G /dev/vg0/lv1` to extend the logical volume, then use `resize2fs /dev/vg0/lv1` for ext4 or `xfs_growfs /mount/point` for XFS. XFS can be grown online without unmounting.

9. Soft mounts use a timeout and will fail gracefully if the server is unavailable. Hard mounts will block and wait for the server to respond. Soft mounts are safer but may cause data inconsistency; hard mounts are safer for data integrity.

10. Add `usrquota,grpquota` to the mount options in `/etc/fstab`. Then run `quotacheck -cvug /mount/point` to scan for quotas, `quotaon /mount/point` to enable quotas, and `edquota` to edit user and group limits.

11. Use `mount | grep /mount/point` or `df /mount/point` to check if a filesystem is mounted. You can also check `/proc/mounts` for detailed mount information.

12. Check file permissions with `ls -la /path`, check ownership with `ls -ln /path`, verify the file is accessible, and check SELinux context with `ls -Z /path`. Use `chmod` to fix permissions and `chown` to fix ownership.

13. The `noatime` option prevents the filesystem from updating the access time (atime) when files are read. This reduces disk I/O and improves performance, especially for frequently-read files.

14. Use `umount -l /mnt` for lazy unmount, which defers unmounting until all references are released. Alternatively, use `fuser -vm /mnt` to find and kill processes using the mount point, then unmount normally.

15. Use `fsck.ext4 -f /dev/vg0/lv1` to force check and repair an ext4 filesystem. For XFS, use `xfs_repair /dev/vg0/lv1`. Always unmount the filesystem before repairing.
