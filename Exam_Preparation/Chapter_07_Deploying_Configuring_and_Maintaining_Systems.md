# Chapter 7: Deploying, Configuring, and Maintaining Systems

## Objectives

After this chapter, you should be able to:

- schedule one-time work with `at`;
- schedule recurring work with cron and systemd timers;
- start, stop, enable, and disable services;
- select the default boot target;
- configure and verify a chrony time client;
- install and update packages from a repository or local RPM;
- inspect and modify persistent kernel command-line settings; and
- make boot-loader changes without confusing BIOS and UEFI paths.

## 1. Choose the right scheduler

| Need | Tool |
| --- | --- |
| one execution at a human-friendly time | at |
| simple recurring calendar schedule | cron |
| recurring job with systemd dependencies, logging, and missed-run catch-up | systemd timer |

## 2. One-time jobs with at

Install and start:

~~~bash
# dnf install at
# systemctl enable --now atd
~~~

Create a job interactively:

~~~bash
$ at 23:00
at> /usr/local/bin/nightly-check >> /var/tmp/nightly-check.log 2>&1
at> Ctrl+d
~~~

Other accepted time forms include:

~~~bash
$ at now + 20 minutes
$ at 04:00 tomorrow
~~~

List and inspect:

~~~bash
$ atq
$ at -c JOB_NUMBER
~~~

Remove:

~~~bash
$ atrm JOB_NUMBER
~~~

An at job receives a limited noninteractive environment. Use absolute command paths when the environment is uncertain, redirect output explicitly, and make the called script executable.

## 3. Recurring jobs with cron

Ensure the daemon runs:

~~~bash
# systemctl enable --now crond
~~~

### User crontab

~~~bash
$ crontab -e
$ crontab -l
$ crontab -r
~~~

Five time fields:

~~~text
minute hour day-of-month month day-of-week command
~~~

Examples:

~~~cron
# Every day at 02:15
15 2 * * * /home/student/bin/report >> /home/student/report.log 2>&1

# Every Monday through Friday at 18:00
0 18 * * 1-5 /home/student/bin/close-day

# Every fifteen minutes
0,15,30,45 * * * * /home/student/bin/check
~~~

### System crontab files

Files in `/etc/cron.d/` contain an extra user field:

~~~cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

30 3 * * * root /usr/local/sbin/system-report
~~~

Set sensible ownership and mode:

~~~bash
# chown root:root /etc/cron.d/system-report
# chmod 644 /etc/cron.d/system-report
~~~

Do not add the user field to a personal `crontab -e`, and do not omit it from `/etc/cron.d/`.

Troubleshoot:

~~~bash
# systemctl status crond
# journalctl -u crond --since today
# tail -f /var/log/cron
$ env
~~~

The command working in an interactive shell does not prove it will work in cron. Cron may have a smaller PATH, no terminal, and a different current directory.

## 4. systemd service and timer units

Create `/etc/systemd/system/system-report.service`:

~~~ini
[Unit]
Description=Create a local system report

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/system-report
~~~

Create `/etc/systemd/system/system-report.timer`:

~~~ini
[Unit]
Description=Run the system report every day

[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true

[Install]
WantedBy=timers.target
~~~

`Persistent=true` causes a missed calendar run to execute after the system becomes available again.

Validate the calendar:

~~~bash
$ systemd-analyze calendar '*-*-* 02:15:00'
~~~

Load, test, and enable:

~~~bash
# systemctl daemon-reload
# systemctl start system-report.service
# systemctl status system-report.service
# journalctl -u system-report.service
# systemctl enable --now system-report.timer
$ systemctl list-timers --all
~~~

Enable the timer, not the oneshot service. The timer activates the service.

After editing a unit file:

~~~bash
# systemctl daemon-reload
# systemctl restart system-report.timer
~~~

Useful timer forms:

~~~ini
OnBootSec=10min
OnUnitActiveSec=1h
~~~

Monotonic timers such as these count from a system or unit event. `OnCalendar` uses wall-clock calendar time.

## 5. Service boot state

~~~bash
# systemctl start SERVICE
# systemctl stop SERVICE
# systemctl restart SERVICE
# systemctl reload SERVICE
# systemctl enable SERVICE
# systemctl disable SERVICE
# systemctl enable --now SERVICE
# systemctl disable --now SERVICE
$ systemctl is-active SERVICE
$ systemctl is-enabled SERVICE
~~~

`enable --now` is a convenient way to satisfy “start now and at boot.”

Masking is stronger than disabling:

~~~bash
# systemctl mask SERVICE
# systemctl unmask SERVICE
~~~

A masked unit cannot be started manually or as a dependency until unmasked. Use it only when the task requires prevention.

## 6. Default boot target

~~~bash
$ systemctl get-default
# systemctl set-default multi-user.target
$ systemctl get-default
~~~

To change only the current state:

~~~bash
# systemctl isolate multi-user.target
~~~

Changing the default and isolating are different actions. Chapter 4 explains recovery targets and one-time kernel parameters.

## 7. Configure a chrony time client

Accurate time is essential for logs, certificates, authentication, and distributed applications.

Inspect:

~~~bash
$ timedatectl
$ systemctl status chronyd
$ chronyc tracking
$ chronyc sources -v
~~~

Install and enable:

~~~bash
# dnf install chrony
# systemctl enable --now chronyd
~~~

Set time zone:

~~~bash
$ timedatectl list-timezones
# timedatectl set-timezone Asia/Bahrain
$ timedatectl
~~~

Configure `/etc/chrony.conf` with the server values supplied by the task:

~~~conf
server ntp1.example.com iburst
server ntp2.example.com iburst
~~~

`iburst` speeds initial measurements when the server becomes reachable.

After editing:

~~~bash
# chronyd -p
# systemctl restart chronyd
$ chronyc sources -v
$ chronyc tracking
~~~

In `chronyc sources -v`, `^*` marks the currently selected source and `^+` marks an acceptable combined source. Synchronization may take time; first confirm DNS, routing, firewall, service state, and source reachability.

Request an immediate correction when appropriate:

~~~bash
# chronyc makestep
~~~

Use it deliberately because stepping the clock can affect applications.

`timedatectl set-ntp true` enables the system's network time mechanism, but the exam task may require direct chrony configuration. Verify `chronyd` and its actual sources.

## 8. Install and update software

The complete procedure is in Chapter 2. Core deployment commands:

~~~bash
# dnf repolist
# dnf info PACKAGE
# dnf install PACKAGE
# dnf install ./local-package.rpm
# dnf check-update
# dnf upgrade PACKAGE
# rpm -q PACKAGE
~~~

Use DNF for a local RPM so that dependencies can be resolved from repositories.

When a package supplies a service:

~~~bash
# dnf install httpd
# systemctl enable --now httpd
$ rpm -q httpd
$ systemctl is-enabled httpd
$ systemctl is-active httpd
~~~

Installing the package does not necessarily enable or start its service.

## 9. Understand the RHEL boot-loader layout

RHEL uses GRUB 2 and Boot Loader Specification entries. Kernel-specific information is commonly represented under `/boot/loader/entries/`, while GRUB configuration is generated rather than maintained as a hand-edited main file.

Inspect:

~~~bash
# grubby --default-kernel
# grubby --default-title
# grubby --info=ALL
# grub2-editenv list
# ls -l /boot/loader/entries
~~~

Use tools rather than manually changing every kernel entry.

## 10. Modify persistent kernel arguments

Add an argument to all installed kernel entries:

~~~bash
# grubby --update-kernel=ALL --args="ARGUMENT"
~~~

Remove it:

~~~bash
# grubby --update-kernel=ALL --remove-args="ARGUMENT"
~~~

Verify:

~~~bash
# grubby --info=ALL
$ cat /proc/cmdline
~~~

`grubby --info=ALL` verifies the saved boot entries. `/proc/cmdline` shows the arguments used for the current boot, so a newly saved argument appears there only after reboot.

Set a particular installed kernel as default:

~~~bash
# grubby --set-default /boot/vmlinuz-KERNEL_VERSION
# grubby --default-kernel
~~~

Use the exact path displayed by `grubby --info=ALL`.

## 11. Regenerate GRUB configuration when required

General GRUB defaults are in `/etc/default/grub`. Back up and edit:

~~~bash
# cp -a /etc/default/grub /etc/default/grub.before-change
# vim /etc/default/grub
# grub2-mkconfig -o /boot/grub2/grub.cfg
~~~

On modern RHEL UEFI installations, `/boot/efi/EFI/redhat/grub.cfg` can be a small forwarding configuration. Do not overwrite it with a full generated configuration unless official platform-specific instructions explicitly require that path. `/boot/grub2/grub.cfg` is the normal generated target on current RHEL.

For kernel arguments, prefer `grubby`; it updates the appropriate boot entries and avoids fragile manual editing.

### One-time GRUB edit

At the menu, press `e`, change the kernel line, and boot with Ctrl+x. That edit is temporary and useful for recovery or testing.

## 12. Boot safety checklist

Before reboot:

~~~bash
# findmnt --verify
# systemctl --failed
# systemctl get-default
# grubby --default-kernel
# grubby --info=ALL
# nmcli connection show --active
# getenforce
~~~

Then record the expected console or recovery path.

## 13. Real-life example: nightly maintenance report

Requirement: run `/usr/local/sbin/maintenance-report` daily at 01:30, catch a missed run, log through the journal, and ensure chrony is enabled.

Create the executable script, a oneshot service, and a timer:

~~~ini
[Unit]
Description=Generate maintenance report
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/maintenance-report
~~~

~~~ini
[Unit]
Description=Nightly maintenance report

[Timer]
OnCalendar=*-*-* 01:30:00
Persistent=true

[Install]
WantedBy=timers.target
~~~

Apply:

~~~bash
# chmod 755 /usr/local/sbin/maintenance-report
# systemctl daemon-reload
# systemctl start maintenance-report.service
# systemctl enable --now maintenance-report.timer
# systemctl enable --now chronyd
$ systemctl list-timers --all
$ journalctl -u maintenance-report.service
$ chronyc tracking
~~~

Testing the service first separates script failure from timer failure.

## 14. Exam lab

Tasks:

1. Schedule a supplied command to run once in 15 minutes.
2. Create a root cron task that runs at 04:10 every Sunday.
3. Create a systemd timer that runs a supplied script 5 minutes after boot and then hourly.
4. Configure `chronyd` to use the supplied server and start at boot.
5. Make `multi-user.target` the default.
6. Add the supplied persistent kernel argument to all kernels.

Solution pattern:

~~~bash
# systemctl enable --now atd crond
$ at now + 15 minutes
# vim /etc/cron.d/sunday-task
# vim /etc/systemd/system/hourly-check.service
# vim /etc/systemd/system/hourly-check.timer
# systemctl daemon-reload
# systemctl start hourly-check.service
# systemctl enable --now hourly-check.timer
# vim /etc/chrony.conf
# chronyd -p
# systemctl enable --now chronyd
# systemctl restart chronyd
# systemctl set-default multi-user.target
# grubby --update-kernel=ALL --args="SUPPLIED_ARGUMENT"
~~~

Timer section for the requested relative schedule:

~~~ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h

[Install]
WantedBy=timers.target
~~~

Verify all saved state:

~~~bash
$ atq
# cat /etc/cron.d/sunday-task
$ systemctl list-timers --all
$ systemctl is-enabled chronyd
$ chronyc sources -v
$ systemctl get-default
# grubby --info=ALL
~~~

## Quick check

- Can you distinguish user crontab syntax from `/etc/cron.d/` syntax?
- Can you test a oneshot service before enabling its timer?
- Can you explain `Persistent=true`?
- Can you prove which chrony source is selected?
- Can you distinguish an installed service from an enabled service?
- Can you add and remove a persistent kernel argument safely?
- Can you explain saved boot entries versus the current `/proc/cmdline`?

## Official references

- [RHEL 10 time synchronization](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_time_synchronization/using-chrony)
- [RHEL 10 software management](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/index)
- Local pages: `man at`, `man 5 crontab`, `man systemd.timer`, `man systemd.time`, `man chrony.conf`, `man grubby`, and `man grub2-mkconfig`

