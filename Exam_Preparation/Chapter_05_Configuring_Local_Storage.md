# Chapter 5: Configuring Local Storage

## Learning Objectives

By the end of this chapter, you will be able to:

- List, create, and delete partitions on GPT disks
- Create and remove physical volumes using LVM
- Assign physical volumes to volume groups
- Create and delete logical volumes
- Configure systems to mount file systems at boot by UUID or label
- Add new partitions and logical volumes to a system non-destructively
- Create, mount, unmount, and use VFAT, ext4, and XFS file systems
- Mount and unmount network file systems using NFS
- Configure autofs for on-demand mounting
- Extend existing logical volumes
- Diagnose and correct file permission problems

---

## Concepts and Theory

### Storage Hierarchy

Linux storage follows a hierarchical structure:

1. **Physical Disks**: Hardware devices (e.g., `/dev/sda`, `/dev/nvme0n1`)
2. **Partitions**: Divisions of physical disks (e.g., `/dev/sda1`, `/dev/sda2`)
3. **Physical Volumes (PV)**: Partitions or entire disks used by LVM
4. **Volume Groups (VG)**: Collections of physical volumes
5. **Logical Volumes (LV)**: Virtual partitions created from volume groups
6. **File Systems**: Filesystem structures created on logical volumes

### Partition Tables

**MBR (Master Boot Record):**
- Supports up to 4 primary partitions
- Maximum disk size: 2 TB
- Legacy standard still used on some systems

**GPT (GUID Partition Table):**
- Supports up to 128 partitions
- Maximum disk size: 9.4 ZB
- Modern standard for disks larger than 2 TB
- Uses UUIDs to identify partitions
- Contains backup partition table for recovery

### LVM (Logical Volume Manager)

LVM provides flexible storage management by abstracting physical storage into virtual storage:

**Physical Volume (PV):**
- A physical disk or partition that can be used by LVM
- Created using `pvcreate`
- Example: `/dev/sda1`, `/dev/nvme0n1p2`

**Volume Group (VG):**
- A pool of physical volumes
- Created using `vgcreate`
- Storage capacity is the sum of all PVs
- Example: `vg0` containing `/dev/sda1` and `/dev/sda2`

**Logical Volume (LV):**
- A virtual partition carved from a volume group
- Created using `lvcreate`
- Can be resized without downtime
- Example: `vg0/lv_home`

### File Systems

**ext4:**
- Default filesystem for RHEL 8
- Journaling filesystem
- Supports files up to 16 TB
- Supports volumes up to 1 EB

**XFS:**
- Default filesystem for RHEL 9 and 10
- Designed for large files and high performance
- Scalable to multi-terabyte volumes
- Journaling is automatic

**VFAT:**
- FAT32-compatible filesystem
- Used for USB drives and removable media
- Maximum file size: 4 GB
- Maximum volume size: 32 GB (FAT32)

### Mounting Mechanisms

**Manual Mounting:**
- Mounting done by administrator
- Requires explicit mount command
- Can be temporary or permanent

**Automatic Mounting:**
- Mounting at boot via `/etc/fstab`
- Uses UUID or filesystem label for identification
- Ensures consistent mounting

**On-Demand Mounting (autofs):**
- Mounts filesystems only when accessed
- Unmounts when idle
- Ideal for network filesystems and rarely accessed data

### File Permissions

Linux uses a permission model based on:

**User Classes:**
- **User (u)**: File owner
- **Group (g)**: File group
- **Others (o)**: Everyone else

**Permission Types:**
- **Read (r)**: Access file contents or list directory
- **Write (w)**: Modify file or create/delete in directory
- **Execute (x)**: Execute file or enter directory

**Special Permissions:**
- **SetUID (s)**: Run file as owner
- **SetGID (s)**: Run file as group
- **Sticky (t)**: Only owner can delete in directory

---

## How It Works Internally

### Partition Creation Process

When creating partitions on a GPT disk:

1. **gdisk** tool writes GPT header to disk
2. **Partition entries** are created with unique GUIDs
3. **Partition type** is specified (e.g., Linux filesystem, EFI)
4. **Size and alignment** are calculated
5. **Partition table** is written to disk

The GPT structure includes:
- Primary header (first sector)
- Backup header (last sector)
- Partition entry array
- Protective MBR (for compatibility)

### LVM Volume Creation

When creating LVM structures:

1. **pvcreate**: Marks partitions as physical volumes, writes PV header
2. **vgcreate**: Combines PVs into volume group, writes VG metadata
3. **lvcreate**: Allocates space from VG, creates LV metadata
4. **mkfs**: Creates filesystem on LV
5. **mount**: Mounts LV to directory

LVM metadata is stored separately from data, allowing for:
- Online volume resizing
- Volume migration between PVs
- Snapshot creation

### Filesystem Creation

When creating a filesystem:

1. **Superblock** is written (contains filesystem metadata)
2. **Block groups** are created (for data storage)
3. **Inode tables** are created (for file metadata)
4. **Journal** is created (for crash recovery)
5. **Block bitmap** is created (for free space tracking)

ext4 and XFS use different approaches:
- **ext4**: Fixed block group structure
- **XFS**: Scalable block group allocation

### Mounting Process

When mounting a filesystem:

1. **Device is checked** for filesystem integrity
2. **Mount point is prepared** (directory exists)
3. **Superblock is read** from device
4. **Mount options are applied** (permissions, options)
5. **Filesystem is mapped** to mount point
6. **Mount status is recorded** in `/proc/mounts`

### UUID and Label Identification

**UUID:**
- 128-bit identifier stored in filesystem superblock
- Unique per filesystem
- Used in `/etc/fstab` for reliable mounting

**Label:**
- Human-readable identifier
- Can be set per filesystem or partition
- Used in `/etc/fstab` with `LABEL=` syntax

Both methods are more reliable than device names (e.g., `/dev/sda1`) which can change.

---

## Commands and Administration Tasks

### Partition Management

#### Listing Partitions

```bash
# List all partitions
lsblk
fdisk -l
gdisk -l

# View GPT partition table
sudo gdisk -l /dev/sda

# View partition details
sudo parted -l
```

#### Creating Partitions with gdisk

```bash
# Start gdisk (GPT partition editor)
sudo gdisk /dev/sda

# gdisk commands:
n           # Create new partition
p           # Print partition table
w           # Write changes to disk
q           # Quit without saving

# Example: Create partition
gdisk /dev/sda
n
# Primary partition number: 5
# First sector: 2048 (or Enter for default)
# Last sector: +10G (or Enter for default)
# Partition name: Enter
w
# Write changes? (Y/N): Y
```

#### Creating Partitions with parted

```bash
# Start parted (non-interactive)
sudo parted /dev/sda

# parted commands:
mkpart primary ext4 1MiB 10GiB
mkpart primary xfs 10GiB 100%
print
quit

# Or non-interactively:
sudo parted /dev/sda mkpart primary ext4 1MiB 10GiB
sudo parted /dev/sda mkpart primary xfs 10GiB 100%
```

#### Creating Partitions with fdisk

```bash
# Start fdisk
sudo fdisk /dev/sda

# fdisk commands:
n           # Create new partition
p           # Print partition table
d           # Delete partition
w           # Write changes to disk
q           # Quit without saving

# Example: Create partition
fdisk /dev/sda
n
p           # Primary partition
1          # Partition number
Enter      # Default start sector
+10G      # Size
w
```

#### Deleting Partitions

```bash
# Using gdisk
sudo gdisk /dev/sda
d          # Delete partition
1          # Partition number
w          # Write changes

# Using parted
sudo parted /dev/sda rm 1

# Using fdisk
sudo fdisk /dev/sda
d          # Delete partition
1          # Partition number
w          # Write changes
```

### Physical Volume Management

#### Creating Physical Volumes

```bash
# Create physical volume on partition
sudo pvcreate /dev/sda1

# Create on multiple partitions
sudo pvcreate /dev/sda1 /dev/sda2 /dev/sdb1

# Create on entire disk
sudo pvcreate /dev/sda

# View PV status
sudo pvdisplay
sudo pvs

# View PV with detailed info
sudo pvdisplay -v
```

#### Removing Physical Volumes

```bash
# Remove PV from use (data remains on disk)
sudo pvremove /dev/sda1

# Remove multiple PVs
sudo pvremove /dev/sda1 /dev/sda2

# Force remove (use with caution)
sudo pvremove -f /dev/sda1
```

### Volume Group Management

#### Creating Volume Groups

```bash
# Create VG from PVs
sudo vgcreate vg0 /dev/sda1 /dev/sda2

# Create VG with name and PVs
sudo vgcreate myvg /dev/sdb1 /dev/sdc1

# View VG status
sudo vgdisplay
sudo vgs

# View VG with detailed info
sudo vgdisplay -v
```

#### Removing Volume Groups

```bash
# Remove all LVs from VG first
sudo lvremove -y vg0/lv1
sudo lvremove -y vg0/lv2

# Remove VG
sudo vgremove vg0

# Force remove VG
sudo vgremove -f vg0
```

#### Extending Volume Groups

```bash
# Add PV to existing VG
sudo pvcreate /dev/sdb1
sudo vgextend vg0 /dev/sdb1

# View VG after extension
sudo vgdisplay vg0

# Add multiple PVs
sudo vgextend vg0 /dev/sdb1 /dev/sdc1
```

### Logical Volume Management

#### Creating Logical Volumes

```bash
# Create LV with size
sudo lvcreate -L 10G -n lv_home vg0

# Create LV with percentage
sudo lvcreate -l 100%FREE -n lv_data vg0

# Create multiple LVs
sudo lvcreate -L 5G -n lv1 vg0
sudo lvcreate -L 10G -n lv2 vg0

# Create LV with specific name
sudo lvcreate -L 20G -n my_backup vg0

# View LV status
sudo lvdisplay
sudo lvs

# View LV with detailed info
sudo lvdisplay -v
```

#### Resizing Logical Volumes

```bash
# Extend LV
sudo lvextend -L +10G /dev/vg0/lv_home
sudo lvextend -l +100%FREE /dev/vg0/lv_data

# Shrink LV (requires unmounting and filesystem support)
sudo lvreduce -L -5G /dev/vg0/lv_home

# View LV size
sudo lvdisplay /dev/vg0/lv_home
```

#### Removing Logical Volumes

```bash
# Remove LV (data is lost)
sudo lvremove /dev/vg0/lv_home

# Force remove
sudo lvremove -f /dev/vg0/lv_home

# Remove multiple LVs
sudo lvremove -y /dev/vg0/lv1 /dev/vg0/lv2
```

### Filesystem Creation

#### Creating Filesystems

```bash
# Create ext4 filesystem
sudo mkfs.ext4 /dev/vg0/lv_home

# Create XFS filesystem
sudo mkfs.xfs /dev/vg0/lv_data

# Create VFAT filesystem
sudo mkfs.vfat -F 32 /dev/sdb1

# Create filesystem with options
sudo mkfs.ext4 -L home /dev/vg0/lv_home
sudo mkfs.xfs -f -L data /dev/vg0/lv_data

# View filesystem info
sudo df -T
sudo blkid
```

### Mounting Filesystems

#### Manual Mounting

```bash
# Mount filesystem
sudo mount /dev/vg0/lv_home /home

# Mount with options
sudo mount -o rw,noatime /dev/vg0/lv_data /data

# Mount XFS filesystem
sudo mount /dev/vg0/lv_xfs /mnt/xfs

# Unmount filesystem
sudo umount /home

# Check mount status
mount | grep lv_home
```

#### Mounting by UUID

```bash
# Find UUID
sudo blkid /dev/vg0/lv_home

# Mount by UUID
sudo mount -a

# View /etc/fstab
cat /etc/fstab

# Example /etc/fstab entry:
# UUID=abc123 /home ext4 defaults 0 2
```

#### Mounting by Label

```bash
# Create label
sudo xfs_admin -L mylabel /dev/vg0/lv_home
# or for ext4:
sudo e2label /dev/vg0/lv_home mylabel

# Mount by label
sudo mount -a

# Example /etc/fstab entry:
# LABEL=home /home ext4 defaults 0 2
```

### NFS Mounting

#### Configuring NFS Server

```bash
# Install NFS server
sudo dnf install -y nfs-utils

# Export directory
sudo exportfs -a
sudo systemctl enable --now nfs-server

# View exports
cat /etc/exports

# Edit exports
sudo vi /etc/exports

# Example /etc/exports:
# /export/data 192.168.1.0/24(rw,sync,no_root_squash)
# /export/home *(ro,sync,root_squash)

# Restart NFS
sudo systemctl restart nfs-server
```

#### Mounting NFS Client

```bash
# Mount NFS temporarily
sudo mount -t nfs 192.168.1.10:/export/data /mnt/nfs

# Mount NFS with options
sudo mount -t nfs 192.168.1.10:/export/data /mnt/nfs -o rw,noatime

# Mount NFS by UUID (requires NFS server to export with UUID)
sudo mount -t nfs server:/export/data /mnt/nfs

# Add to /etc/fstab
# 192.168.1.10:/export/data /mnt/nfs nfs defaults 0 0

# Unmount NFS
sudo umount /mnt/nfs
```

### Autofs Configuration

#### Installing Autofs

```bash
# Install autofs
sudo dnf install -y autofs

# Create autofs configuration
sudo vi /etc/auto.master

# Create mount points
sudo mkdir -p /mnt/nfs
```

#### Configuring Autofs

```bash
# Edit auto.master
sudo vi /etc/auto.master

# Add entry:
# /mnt/nfs /etc/auto.nfs --timeout=60

# Edit auto.nfs
sudo vi /etc/auto.nfs

# Add mount entries:
# data -fstype=nfs,soft,timeo=30,retrans=2 server:/export/data
# home -fstype=nfs,soft,timeo=30,retrans=2 server:/export/home

# Restart autofs
sudo systemctl restart autofs
sudo systemctl enable --now autofs

# Test autofs
touch /mnt/nfs/data/test.txt
ls /mnt/nfs/data/
```

### Extending Logical Volumes

#### Extending LV and Filesystem

```bash
# Extend LV
sudo lvextend -L +10G /dev/vg0/lv_home

# Resize ext4 filesystem
sudo resize2fs /dev/vg0/lv_home

# Resize XFS filesystem
sudo xfs_growfs /mount/point

# View changes
df -h /home
```

### File Permission Troubleshooting

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

# Fix directory permissions for access
sudo chmod -R 755 /path/to/directory

# Fix ownership recursively
sudo chown -R user:group /path/to/directory

# Check for missing execute bit
find /path -type f -perm -111 ! -perm /111 -ls
```

---

## Configuration Examples

### GPT Partition Table Configuration

File: `/dev/sda` partition layout

```bash
# Partition layout for RHEL 10 installation
# Part 1: EFI System Partition (512MB)
# Part 2: Root partition (50GB)
# Part 3: Swap partition (8GB)
# Part 4: Home partition (remaining space)

# Using gdisk:
sudo gdisk /dev/sda

# Create EFI partition
n
1
2048
+512M
EFI System

# Create root partition
n
2
+50G
Linux filesystem

# Create swap partition
n
3
+8G
Linux swap

# Create home partition
n
4
remaining
Linux filesystem

# Write changes
w
```

### LVM Volume Group Configuration

File: `/etc/lvm/lvm.conf` (customized)

```conf
# LVM configuration
{
    global {
        umask = 077
        allocation = "linear"
        backup = 1
    }
}
```

### /etc/fstab Configuration

```bash
# /etc/fstab - Filesystem table
# UUID=abc123def4:/boot /boot xfs defaults 0 2
# UUID=def4567890:/ / xfs defaults 0 1
# UUID=7890123456 swap swap defaults 0 0
# UUID=3456789012 /home xfs defaults 0 2
# LABEL=home /home ext4 defaults 0 2
# 192.168.1.10:/export/data /mnt/nfs nfs defaults 0 0
```

### Autofs Configuration

File: `/etc/auto.master`

```
/mnt/nfs /etc/auto.nfs --timeout=60
/mnt/data /etc/auto.data --timeout=300
```

File: `/etc/auto.nfs`

```
data -fstype=nfs,soft,timeo=30,retrans=2,vers=3 server:/export/data
home -fstype=nfs,soft,timeo=30,retrans=2,vers=3 server:/export/home
```

File: `/etc/auto.data`

```
backup -fstype=ext4,defaults server:/mnt/backup
```

### NFS Server Configuration

File: `/etc/exports`

```bash
# NFS exports configuration
# /export/data 192.168.1.0/24(rw,sync,no_root_squash,sec=sys)
# /export/home *(ro,sync,root_squash,sec=sys)
# /export/backup 10.0.0.0/8(rw,sync,root_squash,sec=sys)
```

### SELinux File Context Configuration

File: `/etc/selinux/targeted/contexts/files/file_contexts.local`

```
# SELinux file contexts for custom filesystems
/export/data.*    system_u:object_r:admin_home_t:s0
/export/home.*    system_u:object_r:home_t:s0
```

---

## Real-World Examples

### Scenario 1: Setting Up a Web Server Filesystem

A new web server needs to be configured with separate partitions for web content, logs, and backup.

```bash
# Create partition layout
sudo gdisk /dev/sdb

# Create partitions:
# Part 1: 50GB for /var/www
# Part 2: 20GB for /var/log
# Part 3: Remaining for backup

# Create LVM
sudo pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb3
sudo vgcreate webvg /dev/sdb1 /dev/sdb2 /dev/sdb3

# Create logical volumes
sudo lvcreate -L 50G -n lv_www webvg
sudo lvcreate -L 20G -n lv_logs webvg
sudo lvcreate -l 100%FREE -n lv_backup webvg

# Create filesystems
sudo mkfs.xfs /dev/webvg/lv_www
sudo mkfs.xfs /dev/webvg/lv_logs
sudo mkfs.xfs /dev/webvg/lv_backup

# Create mount points
sudo mkdir -p /var/www /var/log/webserver /backup

# Mount filesystems
sudo mount /dev/webvg/lv_www /var/www
sudo mount /dev/webvg/lv_logs /var/log/webserver
sudo mount /dev/webvg/lv_backup /backup

# Add to /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_www | grep UUID | cut -d= -f2) /var/www xfs defaults 0 2' | sudo tee -a /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_logs | grep UUID | cut -d= -f2) /var/log/webserver xfs defaults 0 2' | sudo tee -a /etc/fstab
echo 'UUID=$(blkid /dev/webvg/lv_backup | grep UUID | cut -d= -f2) /backup xfs defaults 0 2' | sudo tee -a /etc/fstab
```

### Scenario 2: Expanding Storage Without Downtime

A database server needs additional storage, but the application cannot be stopped.

```bash
# Check current LV size
sudo lvdisplay /var/log/lv_database

# Extend logical volume
sudo lvextend -L +50G /dev/vg_database/lv_database

# Resize XFS filesystem (online)
sudo xfs_growfs /mnt/database

# Verify new size
df -h /mnt/database

# Application continues running during resize
```

### Scenario 3: Setting Up Shared Storage with NFS

Multiple servers need to access shared files.

```bash
# Server 1: Set up NFS server
sudo dnf install -y nfs-utils
sudo mkdir -p /shared/data
sudo chown -R nobody:nogroup /shared/data
sudo chmod -R 755 /shared/data

# Configure exports
echo '/shared/data 192.168.1.0/24(rw,sync,no_root_squash)' | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-server

# Server 2: Mount NFS share
sudo mkdir -p /mnt/shared
sudo mount -t nfs 192.168.1.10:/shared/data /mnt/shared

# Add to /etc/fstab
echo '192.168.1.10:/shared/data /mnt/shared nfs defaults 0 0' | sudo tee -a /etc/fstab

# Test autofs for on-demand mounting
sudo dnf install -y autofs
echo '/mnt/shared /etc/auto.shared --timeout=60' | sudo tee -a /etc/auto.master
echo 'data -fstype=nfs,soft,timeo=30 192.168.1.10:/shared/data' | sudo tee -a /etc/auto.shared
sudo systemctl restart autofs
```

### Scenario 4: Troubleshooting Mount Issues

A system administrator needs to fix a filesystem that won't mount.

```bash
# Check if filesystem is mounted
mount | grep /data

# Check /etc/fstab
cat /etc/fstab

# Check filesystem type
sudo blkid /dev/vg0/lv_data

# Test filesystem
sudo xfs_repair /dev/vg0/lv_data

# Try mounting
sudo mount /dev/vg0/lv_data /data

# Check permissions
ls -la /data

# Fix ownership if needed
sudo chown -R root:root /data

# Verify mount
df -h /data
```

---

## Troubleshooting

### Partition Issues

**Problem**: Cannot create partition

```bash
# Check disk is not in use
lsblk
fdisk -l /dev/sda

# Check disk is not mounted
mount | grep /dev/sda

# Unmount if needed
sudo umount /dev/sda1

# Check disk is writable
sudo test -w /dev/sda
```

**Problem**: Partition not recognized

```bash
# Rescan SCSI devices
echo 1 > /sys/class/scsi_device/*/device/rescan

# Refresh partition table
sudo partprobe /dev/sda
# or
sudo partx -u /dev/sda

# Reload kernel modules
sudo modprobe -r sd_mod
sudo modprobe sd_mod
```

### LVM Issues

**Problem**: Cannot create physical volume

```bash
# Check PV status
sudo pvdisplay

# Check if PV already exists
sudo pvck

# Remove existing PV
sudo pvremove /dev/sda1

# Create new PV
sudo pvcreate /dev/sda1
```

**Problem**: Cannot extend volume group

```bash
# Check available PVs
sudo pvs

# Check VG status
sudo vgdisplay

# Check if PV is part of VG
sudo vgs -o vg_name,pv_count

# Add missing PV
sudo pvcreate /dev/sdb1
sudo vgextend vg0 /dev/sdb1
```

**Problem**: Cannot resize logical volume

```bash
# Check LV status
sudo lvs

# Check if LV is mounted
mount | grep /dev/vg0/lv_home

# Unmount if needed
sudo umount /dev/vg0/lv_home

# Extend LV
sudo lvextend -L +10G /dev/vg0/lv_home

# Resize filesystem
sudo resize2fs /dev/vg0/lv_home
# or
sudo xfs_growfs /mnt/home
```

### Filesystem Issues

**Problem**: Filesystem won't mount

```bash
# Check filesystem type
sudo blkid /dev/vg0/lv_home

# Check for errors
sudo xfs_repair /dev/vg0/lv_home
# or
sudo fsck.ext4 /dev/vg0/lv_home

# Check /etc/fstab
cat /etc/fstab

# Try mounting with verbose
sudo mount -v /dev/vg0/lv_home /mnt

# Check SELinux
getenforce

# Set SELinux to permissive for testing
sudo setenforce 0

# Try mounting again
sudo mount /dev/vg0/lv_home /mnt

# Restore SELinux
sudo setenforce 1
```

**Problem**: NFS mount fails

```bash
# Check NFS server is running
ssh server "systemctl status nfs-server"

# Check exports configuration
ssh server "cat /etc/exports"

# Check firewall
ssh server "firewall-cmd --list-all"

# Check client mount options
mount | grep nfs

# Check network connectivity
ping server

# Test NFS mount
sudo mount -t nfs server:/export/data /mnt/nfs

# Check NFS statistics
nfsstat
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

# Check for setuid/setgid bits
stat /path/to/file

# Fix permissions
sudo chmod 755 /path/to/directory
sudo chown user:group /path/to/directory

# Fix SELinux context
sudo restorecon -Rv /path/to/directory
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
```

---

## Verification Procedures

### Partition Verification

```bash
# Verify partition table
sudo gdisk -l /dev/sda
sudo parted -l

# Verify partitions are accessible
lsblk -f

# Verify partition UUIDs
sudo blkid
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

### Filesystem Verification

```bash
# Verify filesystems
sudo df -T
sudo blkid

# Verify XFS filesystem
xfs_info /mnt/point

# Verify ext4 filesystem
dumpe2fs /dev/vg0/lv_home

# Verify filesystem integrity
sudo xfs_repair -n /dev/vg0/lv_home
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

---

## RHCSA Exam Notes

### Exam Relevance

Configuring local storage is a heavily tested area on the RHCSA. Expect 3-4 questions covering partitioning, LVM, filesystem creation, mounting, and NFS configuration.

### Key Exam Points

1. **GPT Partitioning**: Know how to create and manage GPT partitions with `gdisk` and `parted`.

2. **LVM Commands**: Know `pvcreate`, `vgcreate`, `lvcreate`, `lvextend`, `lvremove`, `vgextend`, `vgremove`.

3. **Filesystem Creation**: Know `mkfs.ext4`, `mkfs.xfs`, `mkfs.vfat` and their options.

4. **Mounting**: Know how to mount by UUID, label, and device name. Know how to add to `/etc/fstab`.

5. **NFS**: Know how to configure NFS server and client, and mount NFS shares.

6. **Autofs**: Know how to configure autofs for on-demand mounting.

7. **File Permissions**: Know how to diagnose and fix permission problems using `chmod`, `chown`, `chattr`.

### Common Exam Traps

- Using `fdisk` instead of `gdisk` for GPT disks
- Forgetting to add new partition to LVM after creation
- Not resizing the filesystem after extending the LV
- Using device names in `/etc/fstab` instead of UUIDs
- Forgetting to enable autofs service
- Not setting correct NFS export permissions

### Time Management Tips

- Use `lsblk` to quickly identify disk structure
- Use `blkid` to get UUIDs for /etc/fstab
- Use `pvdisplay`, `vgdisplay`, `lvdisplay` to check LVM status
- Use `df -T` to check mounted filesystems
- Use `mount | grep` to find specific mounts

### Exam Environment Notes

- You will have access to physical disks for partitioning
- LVM tools are pre-installed
- NFS utilities may need to be installed
- UUIDs can be obtained with `blkid`
- Autofs may require manual configuration

---

## Chapter Summary

This chapter covered local storage configuration including partitioning with GPT, LVM management, filesystem creation and mounting, NFS configuration, and autofs setup. You learned how to create and manage partitions on GPT disks, configure LVM for flexible storage management, create and mount various filesystem types, set up network file sharing with NFS, and configure on-demand mounting with autofs.

These skills are essential for system administration tasks including disk expansion, storage optimization, and shared file access. Understanding storage configuration also helps diagnose and resolve common issues related to filesystem mounting, permissions, and data access.

---

## Quick Reference

### Partition Commands

```bash
gdisk /dev/sda           # GPT partition editor
parted /dev/sda          # Partition editor
fdisk /dev/sda           # Partition editor (MBR)
lsblk                    # List block devices
blkid                    # List UUIDs
```

### LVM Commands

```bash
pvcreate /dev/sda1       # Create physical volume
pvremove /dev/sda1       # Remove physical volume
pvdisplay                # Display PV info
pvs                      # List PVs

vgcreate vg0 /dev/sda1   # Create volume group
vgremove vg0             # Remove volume group
vgdisplay                # Display VG info
vgs                      # List VGs

lvcreate -L 10G -n lv1 vg0  # Create LV
lvremove /dev/vg0/lv1    # Remove LV
lvdisplay                # Display LV info
lvs                      # List LVs

lvextend -L +10G /dev/vg0/lv1  # Extend LV
lvreduce -L -5G /dev/vg0/lv1   # Shrink LV
```

### Filesystem Commands

```bash
mkfs.ext4 /dev/vg0/lv1   # Create ext4
mkfs.xfs /dev/vg0/lv1    # Create XFS
mkfs.vfat -F 32 /dev/sdb1  # Create VFAT

df -T                    # List mounted filesystems
df -h                    # Human-readable
```

### Mount Commands

```bash
mount /dev/vg0/lv1 /mnt      # Mount by device
mount -a                   # Mount from /etc/fstab
umount /mnt                 # Unmount
umount -l /mnt              # Lazy unmount
mount | grep /mnt           # Check mount
```

### UUID and Label

```bash
blkid                     # List UUIDs
e2label /dev/sda1 label   # Set ext4 label
xfs_admin -L label /dev/sda1  # Set XFS label
```

### NFS Commands

```bash
exportfs -a               # Export all
exportfs -v               # List exports
showmount -e server       # View exports
mount -t nfs server:/export /mnt  # Mount NFS
```

### Autofs Commands

```bash
vi /etc/auto.master       # Edit master config
vi /etc/auto.nfs          # Edit mount points
systemctl restart autofs  # Restart autofs
```

### Permission Commands

```bash
chmod 755 /path           # Change permissions
chown user:group /path    # Change ownership
ls -la /path              # List permissions
stat /path                # Detailed info
```

---

## Review Questions

1. What is the difference between MBR and GPT partition tables?

2. How do you create a new partition on a GPT disk using gdisk?

3. What command would you use to create a physical volume from a partition?

4. How do you extend a logical volume without unmounting it?

5. What is the advantage of using UUIDs instead of device names in /etc/fstab?

6. How do you create an ext4 filesystem on a logical volume?

7. What command would you use to mount a filesystem by UUID?

8. How do you configure a directory to be exported via NFS?

9. What is autofs and when would you use it?

10. How do you extend a logical volume and resize the filesystem at the same time?

11. What command would you use to check if a logical volume is mounted?

12. How do you set a filesystem label for mounting purposes?

13. What is the difference between soft and hard NFS mounts?

14. How do you diagnose a permission problem where a user cannot access a directory?

15. What command would you use to remove a physical volume from LVM use?

---

## Answers

1. MBR (Master Boot Record) supports up to 4 primary partitions and disks up to 2 TB. GPT (GUID Partition Table) supports up to 128 partitions and disks up to 9.4 ZB. GPT uses UUIDs for partition identification and includes a backup partition table for recovery.

2. Use `sudo gdisk /dev/sda`, then type `n` to create a new partition. Specify partition number, start sector, end sector (or size), and partition type. Press `w` to write changes to disk.

3. Use `pvcreate /dev/sda1` to create a physical volume from a partition. This marks the partition as usable by LVM and writes LVM metadata.

4. Use `lvextend -L +10G /dev/vg0/lv1` to extend the logical volume. Then resize the filesystem: use `resize2fs` for ext4 or `xfs_growfs /mount/point` for XFS. XFS can be resized online without unmounting.

5. UUIDs are persistent identifiers that don't change even if device names change (e.g., `/dev/sda1` might become `/dev/sdb1` after adding a disk). Using UUIDs ensures the correct filesystem is mounted regardless of device naming.

6. Use `mkfs.ext4 /dev/vg0/lv1` to create an ext4 filesystem. For a labeled filesystem, use `mkfs.ext4 -L mylabel /dev/vg0/lv1`.

7. Use `mount -a` to mount all filesystems listed in /etc/fstab. To mount a specific UUID, use `mount UUID=abc123 /mount/point` or add the UUID to /etc/fstab and run `mount -a`.

8. Add an entry to /etc/exports: `/export/path user_or_network(options)`. For example: `/export/data 192.168.1.0/24(rw,sync,no_root_squash)`. Then run `exportfs -a` and restart the NFS service.

9. Autofs automatically mounts filesystems when they are first accessed and unmounts them when idle. It's useful for network filesystems that are accessed infrequently, as it saves resources and improves security by not keeping unnecessary mounts active.

10. Use `lvextend -L +10G /dev/vg0/lv1` to extend the logical volume, then use `resize2fs /dev/vg0/lv1` for ext4 or `xfs_growfs /mount/point` for XFS. XFS can be grown online without unmounting.

11. Use `mount | grep /dev/vg0/lv1` or `df /mount/point` to check if a logical volume is mounted. You can also check `/proc/mounts`.

12. Use `e2label /dev/vg0/lv1 mylabel` for ext4 or `xfs_admin -L mylabel /dev/vg0/lv1` for XFS. Then use `LABEL=mylabel` in /etc/fstab to mount by label.

13. Soft mounts use a timeout and will fail gracefully if the server is unavailable. Hard mounts will block and wait for the server to respond. Soft mounts are safer but may cause data inconsistency; hard mounts are safer for data integrity.

14. Check directory permissions with `ls -ld /path`, check ownership with `ls -ln /path`, verify the directory is mounted correctly, and check SELinux context with `ls -Z /path`. Use `chmod` to fix permissions and `chown` to fix ownership.

15. Use `pvremove /dev/sda1` to remove the physical volume. This removes LVM metadata but not the data on the partition. Use `pvremove -f` to force removal without confirmation.
