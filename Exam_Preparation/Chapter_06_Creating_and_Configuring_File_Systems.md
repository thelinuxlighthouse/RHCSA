# Chapter 6: Creating and Configuring File Systems

## Objectives

After this chapter, you should be able to:

- create, mount, unmount, and persist VFAT, ext4, and XFS file systems;
- identify a file system by type, UUID, and label;
- mount an NFS export temporarily and at boot;
- configure indirect and direct autofs maps;
- extend an LV and grow its XFS or ext4 file system; and
- diagnose permission, mount, and file-system problems methodically.

## 1. File system choice

| File system | Typical RHEL use | Important facts |
| --- | --- | --- |
| XFS | general RHEL data and large file systems | grows online; does not shrink |
| ext4 | general Linux data | grows online; can shrink only offline with care |
| VFAT | firmware, EFI, and cross-platform exchange | no normal Unix ownership and mode metadata |
| NFS | remote Unix file sharing | depends on network and server export |

The exam may specify the type. Use exactly the requested file system.

## 2. Identify current file systems

~~~bash
# lsblk -f
# blkid
# findmnt
# findmnt -t xfs,ext4,vfat,nfs,nfs4
# df -hT
# file -s /dev/vdb1
~~~

Differences:

- `lsblk -f` connects block devices to file systems and mount points;
- `blkid` reports signatures, UUIDs, and labels;
- `findmnt` reports the mounted view and options;
- `df` reports capacity from mounted file systems;
- `file -s` inspects a block device even when it is not mounted.

## 3. Create file systems

Confirm the device before every `mkfs`. Creating a file system overwrites existing file-system metadata.

### XFS

~~~bash
# mkfs.xfs -L appdata /dev/vgapp/lvdata
# blkid /dev/vgapp/lvdata
~~~

If an existing signature is detected, stop and investigate. Do not add `-f` merely to suppress the warning.

### ext4

~~~bash
# mkfs.ext4 -L archive /dev/vdb1
# blkid /dev/vdb1
~~~

### VFAT

The command is supplied by the `dosfstools` package:

~~~bash
# dnf install dosfstools
# mkfs.vfat -n EXCHANGE /dev/vdb2
# blkid /dev/vdb2
~~~

VFAT labels have format-specific length and character limitations. Use a short, simple label.

### Change a label

~~~bash
# xfs_admin -L newxfslabel /dev/vgapp/lvdata
# e2label /dev/vdb1 newextlabel
# fatlabel /dev/vdb2 NEWVFAT
~~~

Unmount before changing labels unless the tool's documentation explicitly supports the mounted state. After changing a label, update any fstab entry that uses the old label.

## 4. Mount and unmount

### Temporary mount

~~~bash
# mkdir -p /srv/appdata
# mount /dev/vgapp/lvdata /srv/appdata
# findmnt /srv/appdata
# df -hT /srv/appdata
~~~

When a recognizable file-system signature exists, `mount` can normally detect the type. It is also valid to be explicit:

~~~bash
# mount -t ext4 /dev/vdb1 /archive
~~~

Mount by label or UUID:

~~~bash
# mount LABEL=appdata /srv/appdata
# mount UUID=ACTUAL_UUID /srv/appdata
~~~

### What mounting does to a nonempty directory

Mounting does not delete existing directory contents, but it hides them until unmount. Therefore, use an empty mount point unless you intentionally understand the overlay.

### Unmount

~~~bash
# umount /srv/appdata
~~~

If it is busy:

~~~bash
# findmnt /srv/appdata
# fuser -vm /srv/appdata
# lsof +D /srv/appdata
~~~

Common causes include a shell whose current directory is inside the mount, an open file, or a running service. Leave the directory or stop the consumer, then unmount. Avoid lazy or forced unmount as a first response.

## 5. Persistent local mounts

Example `/etc/fstab`:

~~~fstab
UUID=ACTUAL_XFS_UUID /srv/appdata xfs defaults 0 0
LABEL=archive /archive ext4 defaults 0 2
LABEL=EXCHANGE /exchange vfat defaults,umask=0022 0 0
~~~

For VFAT, mount options such as `uid`, `gid`, `umask`, `fmask`, and `dmask` present synthetic Linux ownership and permissions because VFAT does not store normal Unix mode bits.

Safe workflow:

~~~bash
# mkdir -p /srv/appdata /archive /exchange
# cp -a /etc/fstab /etc/fstab.before-filesystems
# vim /etc/fstab
# findmnt --verify
# mount -a
# findmnt /srv/appdata
# findmnt /archive
# findmnt /exchange
~~~

`mount -a` does not enable swap; `swapon -a` handles swap.

### Useful mount options

| Option | Meaning |
| --- | --- |
| `defaults` | standard default set |
| `ro` | read-only |
| `rw` | read-write |
| `noexec` | do not directly execute binaries from this mount |
| `nosuid` | ignore set-user-ID and set-group-ID bits |
| `nodev` | do not interpret device special files |
| `_netdev` | identify a network-dependent mount |

Apply changed options to a mounted file system:

~~~bash
# mount -o remount,ro /srv/appdata
# findmnt -no OPTIONS /srv/appdata
~~~

## 6. NFS client mounts

Install client tools:

~~~bash
# dnf install nfs-utils
~~~

Discovering exports can be helpful when supported by the server:

~~~bash
$ showmount -e nfsserver.example.com
~~~

An NFSv4-only server may not expose every export through `showmount`; use the exact server path supplied by the task.

### Temporary NFS mount

~~~bash
# mkdir -p /mnt/projects
# mount -t nfs nfsserver.example.com:/exports/projects /mnt/projects
# findmnt /mnt/projects
# touch /mnt/projects/client-test
~~~

The touch test succeeds only if server export policy, network identity mapping, and directory permissions allow writing.

### Persistent NFS mount

~~~fstab
nfsserver.example.com:/exports/projects /mnt/projects nfs defaults,_netdev 0 0
~~~

Test:

~~~bash
# umount /mnt/projects
# findmnt --verify
# mount -a
# findmnt /mnt/projects
~~~

Name resolution and network readiness must work at boot. Use the server name only when hostname resolution is correctly configured.

## 7. autofs

autofs mounts a file system when its path is accessed and unmounts it after inactivity. It is useful for many NFS shares or intermittently used shares.

Install and start:

~~~bash
# dnf install autofs nfs-utils
# systemctl enable --now autofs
~~~

### Indirect map

Goal: accessing `/shares/projects` mounts `nfsserver:/exports/projects`.

Create master-map fragment `/etc/auto.master.d/shares.autofs`:

~~~text
/shares /etc/auto.shares
~~~

Create `/etc/auto.shares`:

~~~text
projects -rw,sync nfsserver.example.com:/exports/projects
public   -ro      nfsserver.example.com:/exports/public
~~~

Apply:

~~~bash
# systemctl reload autofs
$ ls /shares/projects
$ findmnt /shares/projects
~~~

The key `projects` becomes a child below `/shares`.

### Indirect wildcard map

If export names and requested keys are the same:

~~~text
* -rw,sync nfsserver.example.com:/exports/&
~~~

Accessing `/shares/team1` substitutes `team1` for `&`.

### Direct map

Master fragment:

~~~text
/- /etc/auto.direct
~~~

Direct map:

~~~text
/mnt/company-data -rw nfsserver.example.com:/exports/company-data
~~~

Reload and access:

~~~bash
# systemctl reload autofs
$ ls /mnt/company-data
$ findmnt /mnt/company-data
~~~

Use an indirect map when many related keys live below one managed base. Use a direct map when the exact absolute mount point is required.

### Troubleshoot autofs

~~~bash
# automount -m
# systemctl status autofs
# journalctl -xeu autofs
$ getent hosts nfsserver.example.com
$ showmount -e nfsserver.example.com
~~~

Do not create or manually mount the final indirect key directory. Let autofs manage it.

## 8. Extend LVM and grow the file system

There are two separate layers:

1. enlarge the LV block device;
2. enlarge the file system to use that space.

Record the current state:

~~~bash
# lvs /dev/vgapp/lvdata
# vgs vgapp
# findmnt /srv/appdata
# df -hT /srv/appdata
~~~

### One-step extension

When supported for the file-system type:

~~~bash
# lvextend -L +1G -r /dev/vgapp/lvdata
~~~

`+1G` means add 1 GiB. Without the plus sign, `-L 1G` means set the final LV size to 1 GiB.

`-r` asks LVM to resize the file system as well. Verify both:

~~~bash
# lvs /dev/vgapp/lvdata
# df -hT /srv/appdata
~~~

### Separate XFS procedure

~~~bash
# lvextend -L +1G /dev/vgapp/lvdata
# xfs_growfs /srv/appdata
~~~

`xfs_growfs` takes the mounted XFS mount point. XFS grows online and cannot be shrunk.

### Separate ext4 procedure

~~~bash
# lvextend -L +1G /dev/vgapp/lvarchive
# resize2fs /dev/vgapp/lvarchive
~~~

An ext4 file system can normally grow while mounted. Verify the actual mount and type first.

### Use all free extents only when requested

~~~bash
# lvextend -l +100%FREE -r /dev/vgapp/lvdata
~~~

This leaves no free space in the VG. It is correct only when the task asks for all remaining space.

## 9. Diagnose permission problems

When a process cannot reach a file, check every control layer rather than repeatedly running `chmod`.

### Identity

~~~bash
$ id
# sudo -u alice id
~~~

Group membership in an existing login session may be stale. A new login normally obtains the new group list.

### Every path component

~~~bash
# namei -om /srv/appdata/team/report.txt
# ls -ld / /srv /srv/appdata /srv/appdata/team
# ls -l /srv/appdata/team/report.txt
~~~

The user needs execute permission on every directory in the path.

### ACL and mount state

~~~bash
# getfacl -p /srv/appdata/team/report.txt
# findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /srv/appdata
~~~

An ACL can grant or restrict effective access. A read-only mount blocks writes even when mode bits appear writable.

### Test as the affected user

~~~bash
# sudo -u alice test -r /srv/appdata/team/report.txt && echo readable
# sudo -u alice test -w /srv/appdata/team/report.txt && echo writable
~~~

### SELinux

~~~bash
# ls -lZ /srv/appdata/team/report.txt
# ps -eZ | grep process-name
# ausearch -m AVC -ts recent
~~~

Chapter 10 teaches correct SELinux repair. Do not disable SELinux or apply random labels to hide a denial.

### NFS-specific permissions

Client root may be mapped to an anonymous identity by server-side root squashing. Fix server export and ownership design; do not assume client root can write every export.

## 10. File-system checking and repair

Check the correct file-system type and unmount before offline repair.

For ext4:

~~~bash
# umount /archive
# e2fsck -f /dev/vgapp/lvarchive
~~~

For XFS:

~~~bash
# umount /srv/appdata
# xfs_repair /dev/vgapp/lvdata
~~~

For VFAT:

~~~bash
# umount /exchange
# fsck.vfat /dev/vdb2
~~~

Do not run a repair tool for the wrong file-system family. Do not run `xfs_repair` on a mounted file system.

## 11. Real-life example: expandable project storage

Requirement: create an ext4 LV, persist it at `/projects`, then add 512 MiB while it is in use.

~~~bash
# lvcreate -n lvprojects -L 1G vgdata
# mkfs.ext4 -L projects /dev/vgdata/lvprojects
# mkdir -p /projects
# blkid /dev/vgdata/lvprojects
# vim /etc/fstab
# findmnt --verify
# mount -a
# findmnt /projects
# df -hT /projects
# lvextend -L +512M -r /dev/vgdata/lvprojects
# lvs /dev/vgdata/lvprojects
# df -hT /projects
~~~

The final two commands prove both layers grew.

## 12. Exam lab

Tasks:

1. Create a 1 GiB XFS LV named `lvteam` in existing VG `vgexam`.
2. Label it `teamdata` and mount it persistently at `/teamdata`.
3. Extend it by 512 MiB, including the file system.
4. Configure autofs so `/shares/docs` mounts `nfsserver.example.com:/exports/docs` read-only.
5. Diagnose why user `alice` cannot read `/teamdata/readme`.

Solution pattern:

~~~bash
# lvcreate -n lvteam -L 1G vgexam
# mkfs.xfs -L teamdata /dev/vgexam/lvteam
# mkdir -p /teamdata
# blkid /dev/vgexam/lvteam
# vim /etc/fstab
# findmnt --verify
# mount -a
# lvextend -L +512M -r /dev/vgexam/lvteam
# lvs /dev/vgexam/lvteam
# df -hT /teamdata
# dnf install autofs nfs-utils
# vim /etc/auto.master.d/shares.autofs
# vim /etc/auto.shares
# systemctl enable --now autofs
# systemctl reload autofs
$ ls /shares/docs
$ findmnt /shares/docs
# sudo -u alice id
# namei -om /teamdata/readme
# getfacl -p /teamdata/readme
# findmnt -no OPTIONS /teamdata
# ls -lZ /teamdata/readme
# ausearch -m AVC -ts recent
~~~

## Quick check

- Can you create and identify XFS, ext4, and VFAT?
- Can you explain why VFAT permissions are mount options?
- Can you validate local and NFS fstab entries without rebooting?
- Can you build both an indirect and a direct autofs map?
- Can you explain LV size versus file-system size?
- Can you grow XFS and ext4 correctly?
- Can you troubleshoot access through identity, path permissions, ACL, mount state, and SELinux?

## Official references

- [RHEL 10 product documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)
- [Configuring and managing LVM](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_logical_volumes/index)
- Local pages: `man mkfs.xfs`, `man mkfs.ext4`, `man mkfs.vfat`, `man mount`, `man fstab`, `man autofs`, `man auto.master`, `man lvextend`, `man xfs_growfs`, and `man resize2fs`

