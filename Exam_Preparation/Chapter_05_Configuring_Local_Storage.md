# Chapter 5: Configuring Local Storage

## Objectives

After this chapter, you should be able to:

- identify disks, partitions, LVM objects, file systems, mount points, and swap;
- create and remove partitions on a GPT disk;
- create and remove LVM physical volumes, volume groups, and logical volumes;
- add storage without damaging existing data;
- configure persistent mounts by UUID or label;
- create and enable persistent swap; and
- remove lab storage in the correct dependency order.

This chapter changes block devices. Use only disposable lab disks until the procedure is automatic.

## 1. The storage stack

Two common paths are:

~~~text
disk -> GPT partition -> file system -> mount point
~~~

and:

~~~text
disk or partition -> LVM PV -> VG -> LV -> file system -> mount point
~~~

Definitions:

- a **partition table** describes partitions on a disk;
- a **partition** is a bounded block-device region;
- an LVM **physical volume**, PV, makes a device usable by LVM;
- a **volume group**, VG, pools one or more PVs;
- a **logical volume**, LV, allocates usable block storage from a VG;
- a **file system** organizes files and directories;
- a **mount point** attaches a file system to the directory tree;
- **swap** supplies disk-backed virtual memory and is not mounted as a file system.

## 2. Inventory before change

Run several views because no single command tells the whole story:

~~~bash
# lsblk -e7 -o NAME,PATH,SIZE,TYPE,FSTYPE,FSVER,LABEL,UUID,MOUNTPOINTS
# blkid
# findmnt
# findmnt --fstab
# swapon --show
# pvs
# vgs
# lvs -a -o +devices
# parted -l
~~~

Before touching `/dev/vdb`, prove:

- it is the correct size;
- it is not mounted;
- it is not active swap;
- it is not already a PV used by a VG;
- it does not contain needed signatures or data.

Never select a disk only because it “looks like the second disk.” Device names can change across systems.

## 3. GPT partitioning with parted

The current objective specifically names GPT.

### Create a GPT table

This destroys the existing partition map on the selected device:

~~~bash
# parted /dev/vdb --script mklabel gpt
~~~

Confirm first with `parted /dev/vdb print`. Do not run `mklabel` on a disk containing needed partitions.

### Create partitions

Use MiB or GiB boundaries:

~~~bash
# parted /dev/vdb --script mkpart primary 1MiB 2049MiB
# parted /dev/vdb --script mkpart primary 2049MiB 4097MiB
# parted /dev/vdb --script name 1 data
# parted /dev/vdb --script name 2 lvm
# parted /dev/vdb --script set 2 lvm on
# partprobe /dev/vdb
# udevadm settle
# parted /dev/vdb unit MiB print
# lsblk /dev/vdb
~~~

The first MiB is left unused for alignment and metadata. The `lvm` flag documents the intended use; `pvcreate` is what actually initializes LVM metadata.

### Create a partition using the remaining space

~~~bash
# parted /dev/vdb --script mkpart primary 4097MiB 100%
# partprobe /dev/vdb
~~~

### Remove a partition

Before deletion, unmount it, disable swap if applicable, and remove any LVM dependency.

~~~bash
# parted /dev/vdb print
# parted /dev/vdb --script rm 3
# partprobe /dev/vdb
# lsblk /dev/vdb
~~~

Removing a partition deletes the partition definition; it does not safely migrate its data elsewhere.

### If the kernel does not see the change

~~~bash
# partprobe /dev/vdb
# udevadm settle
# lsblk /dev/vdb
~~~

If a partition is busy, the kernel may refuse to reread the table. Stop using the device rather than forcing a dangerous live change.

## 4. LVM mental model

Think of:

- PV as a labeled building block;
- VG as a storage pool;
- LV as a flexible virtual partition from the pool.

Inspect:

~~~bash
# pvs -o +pv_used
# vgs -o +vg_free
# lvs -a -o +devices
# pvdisplay /dev/vdb2
# vgdisplay vgdata
# lvdisplay /dev/vgdata/lvfiles
~~~

LVM provides two equivalent device paths:

~~~text
/dev/vgdata/lvfiles
/dev/mapper/vgdata-lvfiles
~~~

Use `/dev/VG/LV` in commands because it is easy to read.

## 5. Create LVM storage

### Physical volume

~~~bash
# pvcreate /dev/vdb2
# pvs
~~~

Do not run `pvcreate -ff` to silence a warning. A warning often means the device contains a signature or belongs to existing storage. Investigate.

### Volume group

~~~bash
# vgcreate vgdata /dev/vdb2
# vgs
~~~

Add another PV later:

~~~bash
# pvcreate /dev/vdc1
# vgextend vgdata /dev/vdc1
# vgs -o vg_name,vg_size,vg_free,pv_count
~~~

### Logical volume

Create by size:

~~~bash
# lvcreate -n lvfiles -L 1G vgdata
~~~

Create by extents:

~~~bash
# lvcreate -n lvlogs -l 50%FREE vgdata
~~~

Verify actual allocation:

~~~bash
# lvs -o lv_name,vg_name,lv_size,devices
~~~

Avoid `100%FREE` when the task expects space to remain for later LVs or extensions.

File-system creation and mounting are covered fully in Chapter 6. A typical continuation is:

~~~bash
# mkfs.xfs -L files /dev/vgdata/lvfiles
# mkdir -p /srv/files
# mount /dev/vgdata/lvfiles /srv/files
~~~

## 6. Persistent mounts by UUID or label

Device names such as `/dev/vdb1` can change. A UUID is created with the file system and is normally unique. A label is human-readable but must be kept unique by the administrator.

Discover both:

~~~bash
# blkid /dev/vdb1
# lsblk -f /dev/vdb1
~~~

Example `/etc/fstab` entries:

~~~fstab
UUID=11111111-2222-3333-4444-555555555555 /srv/archive xfs defaults 0 0
LABEL=files /srv/files xfs defaults 0 0
~~~

Fields are:

1. source;
2. mount point;
3. file-system type;
4. options;
5. dump flag, normally 0;
6. fsck order.

Common final field choices:

- XFS: `0`;
- ext4 root: usually `1`;
- other ext4 file systems: usually `2`;
- VFAT: often `0` in simple lab configuration;
- swap: `0`.

### Safe `/etc/fstab` workflow

~~~bash
# cp -a /etc/fstab /etc/fstab.before-storage
# mkdir -p /srv/archive
# vim /etc/fstab
# findmnt --verify
# mount -a
# findmnt /srv/archive
~~~

Do not test a new entry by rebooting first. `findmnt --verify` catches many errors, and `mount -a` tests mountable entries now.

If the file system is already mounted manually, unmount it before a clean `mount -a` test:

~~~bash
# umount /srv/archive
# mount -a
# findmnt /srv/archive
~~~

## 7. Add and persist swap

Swap can be a partition or LV.

### Swap partition

~~~bash
# mkswap -L swap-extra /dev/vdb3
# swapon /dev/vdb3
# swapon --show
# free -h
~~~

Use the UUID from `blkid` in `/etc/fstab`:

~~~fstab
UUID=SWAP_UUID none swap defaults 0 0
~~~

Test:

~~~bash
# swapoff /dev/vdb3
# swapon -a
# swapon --show
~~~

### Swap logical volume

~~~bash
# lvcreate -n lvswap -L 1G vgdata
# mkswap -L swap-lv /dev/vgdata/lvswap
# swapon /dev/vgdata/lvswap
# blkid /dev/vgdata/lvswap
~~~

Persistent entry:

~~~fstab
UUID=SWAP_LV_UUID none swap defaults 0 0
~~~

If `swapon` reports an invalid argument, confirm that the target has a swap signature with `blkid` and that it is not already in use for a file system.

## 8. Non-destructive expansion mindset

“Add storage non-destructively” means use unused space or new devices without recreating existing file systems or partition tables.

Safe pattern:

1. inventory disks and LVM;
2. identify genuinely unallocated space or a new device;
3. create a new partition without changing the boundaries of existing ones;
4. create a new PV or file system;
5. add the resource;
6. verify data and persistence.

For LVM:

~~~bash
# pvcreate /dev/vdc1
# vgextend vgdata /dev/vdc1
# vgs
~~~

Then create a new LV or extend an existing LV as Chapter 6 explains.

Do not “fix” a size mistake by running `mkfs` again. `mkfs` creates a new file system and destroys access to the old one.

## 9. Remove LVM storage

Dependencies must be removed from the top down.

For a mounted LV:

~~~bash
# findmnt /srv/files
# umount /srv/files
# vim /etc/fstab
# findmnt --verify
# lvremove /dev/vgdata/lvfiles
# lvs
~~~

Remove an empty VG and then its PV:

~~~bash
# vgremove vgdata
# pvremove /dev/vdb2
# pvs
~~~

For swap on an LV:

~~~bash
# swapoff /dev/vgdata/lvswap
# vim /etc/fstab
# lvremove /dev/vgdata/lvswap
~~~

If `lvremove` says the LV is in use, find the consumer instead of forcing removal:

~~~bash
# findmnt -S /dev/vgdata/lvfiles
# swapon --show
# lsof /dev/vgdata/lvfiles
~~~

## 10. Troubleshooting

### Device not visible

~~~bash
# lsblk
# dmesg | tail
# udevadm settle
~~~

For a VM, confirm that the virtual disk is actually attached.

### Mount fails

~~~bash
# blkid DEVICE
# file -s DEVICE
# findmnt --verify
# journalctl -b -p err
~~~

Common causes are a wrong file-system type, wrong UUID, missing mount point, corrupted file system, or an already-mounted device.

### VG or LV not active

~~~bash
# pvs
# vgs
# lvs
# vgchange -ay VG_NAME
~~~

Do not activate unknown VGs from cloned disks without understanding duplicate names and UUIDs.

### Duplicate label

Labels are convenient, not inherently unique:

~~~bash
# blkid -L files
# lsblk -o NAME,LABEL,UUID
~~~

Use UUID when labels are ambiguous.

## 11. Real-life example: add application data without touching the OS disk

Requirement: add a 2 GiB XFS-backed LV mounted at `/srv/appdata`, leaving free VG space for growth.

~~~bash
# lsblk -e7 -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
# parted /dev/vdb --script mklabel gpt
# parted /dev/vdb --script mkpart primary 1MiB 100%
# parted /dev/vdb --script set 1 lvm on
# partprobe /dev/vdb
# pvcreate /dev/vdb1
# vgcreate vgapp /dev/vdb1
# lvcreate -n lvdata -L 2G vgapp
# mkfs.xfs -L appdata /dev/vgapp/lvdata
# mkdir -p /srv/appdata
# blkid /dev/vgapp/lvdata
# vim /etc/fstab
# findmnt --verify
# mount -a
# findmnt /srv/appdata
# vgs
~~~

The natural order follows the storage stack. Each layer is verified before building the next one.

## 12. Exam lab

Use a disposable `/dev/vdb` of at least 6 GiB:

1. Create a GPT table.
2. Create a 1 GiB partition for an ext4 file system.
3. Create a 1 GiB swap partition.
4. Use the remaining space as an LVM PV.
5. Create VG `vgexam` and a 1 GiB LV `lvexam`.
6. Do not format the LV yet.
7. Make the ext4 file system mount by UUID at `/exam`.
8. Make swap persist.

Solution pattern:

~~~bash
# parted /dev/vdb --script mklabel gpt
# parted /dev/vdb --script mkpart primary 1MiB 1025MiB
# parted /dev/vdb --script mkpart primary 1025MiB 2049MiB
# parted /dev/vdb --script mkpart primary 2049MiB 100%
# parted /dev/vdb --script set 3 lvm on
# partprobe /dev/vdb
# udevadm settle
# mkfs.ext4 -L examfs /dev/vdb1
# mkswap -L examswap /dev/vdb2
# pvcreate /dev/vdb3
# vgcreate vgexam /dev/vdb3
# lvcreate -n lvexam -L 1G vgexam
# mkdir -p /exam
# blkid /dev/vdb1 /dev/vdb2
# vim /etc/fstab
# findmnt --verify
# mount -a
# swapon -a
# findmnt /exam
# swapon --show
# pvs
# vgs
# lvs
~~~

Do not copy sample UUIDs. Use the values displayed by `blkid`.

## Quick check

- Can you draw the direct-partition and LVM storage stacks?
- Can you prove a device is unused before changing it?
- Can you create and remove GPT partitions?
- Can you create and remove PVs, VGs, and LVs in the correct order?
- Can you explain why a UUID is safer than `/dev/vdb1` in fstab?
- Can you validate fstab without rebooting?
- Can you enable swap now and after reboot?

## Official references

- [Configuring and managing LVM in RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes/index)
- Local pages: `man parted`, `man lsblk`, `man pvcreate`, `man vgcreate`, `man lvcreate`, `man fstab`, and `man swapon`

