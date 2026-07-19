# Chapter 4: Operating Running Systems

## Objectives

After this chapter, you should be able to:

- boot, reboot, and shut down normally;
- change the current target and choose the default boot target;
- interrupt boot and reset the root password;
- identify CPU- or memory-heavy processes and end them safely;
- adjust process niceness;
- select and verify a TuneD profile;
- search the system journal and traditional log files;
- make the journal persistent;
- start, stop, restart, reload, and inspect services; and
- transfer files securely between systems.

## 1. Normal power operations

Use systemd-aware commands:

~~~bash
# systemctl reboot
# systemctl poweroff
# systemctl halt
~~~

`poweroff` stops the system and requests power removal. `halt` stops the operating system but is not normally the best choice for a VM or physical server.

The `shutdown` command can schedule an operation:

~~~bash
# shutdown -r now
# shutdown -h +10 "Maintenance begins in ten minutes"
# shutdown -c
~~~

Before rebooting a remote system:

1. confirm that you are connected to the intended host with `hostnamectl`;
2. inspect logged-in users with `w`;
3. check failed services with `systemctl --failed`;
4. validate boot-critical configuration; and
5. ensure that you have console access or a recovery path.

## 2. Understand systemd targets

A target groups units into a desired system state.

| Target | Purpose |
| --- | --- |
| `poweroff.target` | power off |
| `rescue.target` | single-user recovery with basic services |
| `emergency.target` | minimal emergency shell |
| `multi-user.target` | non-graphical multi-user system |
| `graphical.target` | multi-user system with graphical login |
| `reboot.target` | reboot |

Inspect:

~~~bash
$ systemctl get-default
$ systemctl list-units --type=target
$ systemctl list-unit-files --type=target
$ systemctl status multi-user.target
~~~

Change the current system state:

~~~bash
# systemctl isolate multi-user.target
# systemctl isolate rescue.target
~~~

`isolate` stops units that are not required by the target. It can terminate your graphical or network session. Do not isolate a recovery target remotely unless you have console access.

Choose the next normal boot state:

~~~bash
# systemctl set-default multi-user.target
# systemctl get-default
~~~

This changes the persistent default but does not immediately isolate the target.

### One-time target from GRUB

At the GRUB menu:

1. select the normal boot entry;
2. press `e`;
3. find the line that starts with `linux`, `linuxefi`, or `linux` followed by the kernel path;
4. append `systemd.unit=rescue.target`;
5. press Ctrl+x to boot.

The edit applies only to that boot.

## 3. Interrupt boot and reset the root password

This procedure assumes authorized console access.

1. Reboot and stop at the GRUB menu.
2. Select the normal kernel entry and press `e`.
3. Append `rd.break` to the kernel command line.
4. Press Ctrl+x to boot.
5. At the switch-root prompt, run:

~~~bash
switch_root:/# mount -o remount,rw /sysroot
switch_root:/# chroot /sysroot
sh-5.2# passwd root
sh-5.2# touch /.autorelabel
sh-5.2# exit
switch_root:/# exit
~~~

Why every step matters:

- the installed root file system is initially mounted read-only;
- `chroot /sysroot` makes the installed system appear as `/`;
- `passwd` updates the installed password database;
- `/.autorelabel` requests SELinux relabeling because the password file changed outside the normal booted SELinux environment;
- the next boot may take longer while relabeling completes.

Do not omit the relabel request on an enforcing SELinux system.

If the task is to repair `/etc/fstab` rather than reset a password, remount and chroot in the same way, edit the file, and run `findmnt --verify`.

## 4. Processes and jobs

A process has a process ID, owner, state, priority, memory use, CPU use, and parent process.

### Inspect processes

~~~bash
$ ps -ef
$ ps aux
$ ps -eo pid,ppid,user,ni,stat,%cpu,%mem,cmd --sort=-%cpu | head
$ ps -eo pid,user,%mem,rss,cmd --sort=-rss | head
$ pgrep -a sshd
$ pidof sshd
$ top
~~~

In `top`:

- press `P` to sort by CPU;
- press `M` to sort by memory;
- press `k` to send a signal;
- press `r` to renice;
- press `q` to quit.

Resident memory, RSS, is the portion currently in RAM. Virtual size is not the same as physical RAM consumption.

### Foreground and background jobs

~~~bash
$ long-command &
$ jobs -l
$ fg %1
$ bg %1
$ Ctrl+z
~~~

Ctrl+z suspends the foreground job; it does not terminate it. `bg` continues it in the background.

To keep a command running after logout:

~~~bash
$ nohup long-command > long-command.log 2>&1 &
~~~

For managed long-running work, a systemd service is more reliable than `nohup`.

## 5. End processes safely

Signals request a process action:

| Signal | Number | Usual purpose |
| --- | ---: | --- |
| HUP | 1 | reload or terminal hangup, application-dependent |
| INT | 2 | interrupt, like Ctrl+c |
| TERM | 15 | request clean termination; default for `kill` |
| KILL | 9 | kernel-enforced termination; cannot be caught |
| STOP | 19 | suspend; cannot be caught |
| CONT | 18 | continue a stopped process |

Use a progressive approach:

~~~bash
$ kill PID
$ sleep 2
$ ps -p PID
$ kill -KILL PID
~~~

Use KILL only when the process does not respond to TERM and immediate termination is necessary. KILL prevents cleanup.

Select by name or other criteria:

~~~bash
$ pkill -TERM process-name
$ pgrep -u alice -a process-name
# pkill -u alice -TERM process-name
~~~

Before ending multiple processes, run the matching `pgrep` command to confirm the exact targets.

For a service, prefer systemd:

~~~bash
# systemctl stop httpd
~~~

This follows the unit's designed shutdown procedure.

## 6. Process scheduling with nice and renice

Linux niceness normally ranges from -20 to 19:

- lower numeric niceness requests more CPU scheduling preference;
- higher niceness yields more readily to other work;
- ordinary users can increase niceness for their processes but normally need privilege to decrease it.

Start with a niceness value:

~~~bash
$ nice -n 10 tar -czf backup.tar.gz /srv/data
~~~

Adjust an existing process:

~~~bash
$ ps -o pid,ni,cmd -p PID
# renice -n 5 -p PID
$ ps -o pid,ni,cmd -p PID
~~~

Niceness affects CPU scheduling, not a hard CPU percentage, and not memory limits.

## 7. TuneD profiles

TuneD applies a coordinated performance profile.

~~~bash
# dnf install tuned
# systemctl enable --now tuned
$ tuned-adm list
$ tuned-adm recommend
$ tuned-adm active
# tuned-adm profile throughput-performance
$ tuned-adm active
~~~

Use the profile requested by the task; do not assume that “performance” is always correct. A virtual guest, a latency-sensitive database, and a power-saving laptop have different goals.

Revert to no TuneD profile:

~~~bash
# tuned-adm off
~~~

Verify both the service and selected profile:

~~~bash
$ systemctl is-enabled tuned
$ systemctl is-active tuned
$ tuned-adm active
~~~

## 8. Services with systemd

Common operations:

~~~bash
$ systemctl status sshd
$ systemctl is-active sshd
$ systemctl is-enabled sshd
# systemctl start sshd
# systemctl stop sshd
# systemctl restart sshd
# systemctl reload sshd
# systemctl reload-or-restart sshd
# systemctl enable sshd
# systemctl disable sshd
# systemctl enable --now sshd
# systemctl disable --now sshd
~~~

Active and enabled are different:

- active describes the current runtime state;
- enabled describes whether systemd is arranged to start it at boot.

Inspect failures and dependencies:

~~~bash
$ systemctl --failed
$ systemctl list-dependencies sshd
$ systemctl cat sshd
$ systemctl show sshd -p ActiveState -p SubState -p UnitFileState
~~~

If unit files were created or changed:

~~~bash
# systemctl daemon-reload
~~~

A reload asks the service to reread configuration. A restart stops and starts it. Not every service supports reload.

## 9. Journals and log files

### Query the journal

~~~bash
$ journalctl
$ journalctl -b
$ journalctl -b -1
$ journalctl -u sshd
$ journalctl -u sshd --since today
$ journalctl --since '2026-07-19 08:00' --until '2026-07-19 09:00'
$ journalctl -p warning..alert
$ journalctl -f
$ journalctl -k
$ journalctl _PID=1234
$ journalctl -xeu sshd
~~~

Useful meanings:

- `-b` selects a boot;
- `-u` selects a unit;
- `-p` selects priorities;
- `-f` follows new entries;
- `-k` selects kernel messages;
- `-e` jumps near the end;
- `-x` adds catalog explanations where available.

List recorded boots:

~~~bash
$ journalctl --list-boots
~~~

### Traditional logs

Depending on installed and enabled services, useful files include:

- `/var/log/messages` for general messages;
- `/var/log/secure` for authentication and authorization;
- `/var/log/cron` for cron activity;
- `/var/log/audit/audit.log` for audit and SELinux AVC records.

Use both sources when appropriate:

~~~bash
# tail -f /var/log/secure
# grep -i 'failed password' /var/log/secure
# ausearch -m AVC -ts recent
~~~

The absence of a traditional file can mean that only the journal is configured; it does not mean the event was never logged.

## 10. Preserve the system journal

Persistent journal data is stored below `/var/log/journal`. A clear configuration is a drop-in:

~~~bash
# mkdir -p /etc/systemd/journald.conf.d
# vim /etc/systemd/journald.conf.d/persistent.conf
~~~

Contents:

~~~ini
[Journal]
Storage=persistent
~~~

Apply and verify:

~~~bash
# systemctl restart systemd-journald
# journalctl --flush
# test -d /var/log/journal && echo persistent-directory-present
# journalctl --disk-usage
~~~

After a reboot:

~~~bash
$ journalctl --list-boots
$ journalctl -b -1
~~~

If the previous boot can be queried, persistence is working.

## 11. Secure file transfer

### scp

~~~bash
$ scp report.txt student@serverb:/tmp/
$ scp student@serverb:/etc/hosts ./serverb.hosts
$ scp -r reports student@serverb:/srv/
$ scp -P 2222 report.txt student@serverb:/tmp/
~~~

For `scp`, uppercase `-P` selects the SSH port.

### sftp

~~~bash
$ sftp student@serverb
sftp> pwd
sftp> lpwd
sftp> put report.txt
sftp> get remote.txt
sftp> exit
~~~

Commands beginning with `l` operate on the local side.

### rsync over SSH

When installed:

~~~bash
$ rsync -av --progress reports/ student@serverb:/srv/reports/
~~~

A trailing slash on `reports/` means copy its contents. Without the slash, rsync copies the directory itself.

## 12. Real-life troubleshooting sequence

A web service is reported unavailable.

~~~bash
$ systemctl is-active httpd
$ systemctl status httpd --no-pager
$ journalctl -xeu httpd
$ ss -lntp
$ ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
~~~

Reasoning:

1. Test whether systemd believes the service is active.
2. Read the immediate status and last messages.
3. Query the full unit-specific journal.
4. Confirm whether a process is listening.
5. Check whether resource pressure explains the symptom.

Networking, firewalld, and SELinux checks are covered in Chapters 8 and 10.

## 13. Exam lab

Tasks:

1. Make the journal persistent.
2. Configure the system to boot normally to `multi-user.target`.
3. Enable and immediately start `chronyd`.
4. Find the top three processes by resident memory.
5. Start a CPU-intensive lab command at niceness 10, then change it to 15.
6. Copy `/etc/hosts` securely to the supplied remote host.

Solution pattern:

~~~bash
# mkdir -p /etc/systemd/journald.conf.d
# vim /etc/systemd/journald.conf.d/persistent.conf
# systemctl restart systemd-journald
# journalctl --flush
# systemctl set-default multi-user.target
# systemctl enable --now chronyd
$ ps -eo pid,user,rss,%mem,cmd --sort=-rss | head -n 4
$ nice -n 10 LAB_COMMAND &
$ jobs -l
# renice -n 15 -p PID
$ scp /etc/hosts student@REMOTE_HOST:/tmp/servera.hosts
~~~

Verify:

~~~bash
$ systemctl get-default
$ systemctl is-enabled chronyd
$ systemctl is-active chronyd
$ ps -o pid,ni,cmd -p PID
$ journalctl --disk-usage
~~~

## Quick check

- Can you explain current target versus default target?
- Can you perform the full authorized root-password recovery procedure?
- Can you identify high CPU and high resident-memory processes?
- Can you explain why TERM should normally precede KILL?
- Can you interpret niceness direction correctly?
- Can you query a previous boot and a single service?
- Can you prove that the journal survives reboot?
- Can you distinguish an active service from an enabled service?

## Official references

- [RHEL 10 documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)
- Local pages: `man systemctl`, `man journalctl`, `man systemd.special`, `man nice`, `man renice`, `man scp`, and `man sftp`

