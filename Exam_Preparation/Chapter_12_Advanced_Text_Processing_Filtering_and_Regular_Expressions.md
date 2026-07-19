# Chapter 12: Text Processing, Verification, and Final EX200 Practice

## Scope

The official objective is to use grep and regular expressions to analyze text. `sed`, `awk`, `cut`, `sort`, `uniq`, `tr`, `wc`, and `xargs` go beyond the minimum wording but make system administration and verification much faster. This chapter also combines all current exam areas into a final practice workflow.

## Objectives

After this chapter, you should be able to:

- choose fixed-string, basic-regex, or extended-regex grep;
- filter comments and blank lines safely;
- extract fields from colon- and whitespace-delimited data;
- transform text without accidentally corrupting configuration;
- combine commands into readable pipelines;
- process filenames safely;
- analyze journal and authentication events;
- verify exam tasks through machine-readable command output; and
- complete a cumulative RHEL 10 EX200 mock lab.

## 1. Text streams are interfaces

Linux tools are useful together because many read stdin and write stdout.

~~~bash
$ producer | filter | formatter > result
~~~

stderr is separate unless redirected:

~~~bash
$ command > normal.out 2> errors.out
~~~

Show and save:

~~~bash
$ journalctl -u sshd --since today | tee sshd-today.txt
~~~

Append as root:

~~~bash
$ printf '%s\n' 'new line' | sudo tee -a /etc/example.conf
~~~

Do not use a long pipeline only to look clever. Each stage should have one clear purpose and be easy to verify.

## 2. Choose the correct grep mode

### Fixed text

Use `-F` when characters should be literal:

~~~bash
$ grep -F '192.0.2.10' /etc/hosts
$ grep -F '[Service]' unit-file
~~~

This avoids interpreting dots or brackets as regex.

### Basic regular expression

Default grep supports anchors, character classes, dot, and repetition:

~~~bash
$ grep '^root:' /etc/passwd
$ grep 'bash$' /etc/passwd
$ grep '[[:digit:]]' file
~~~

### Extended regular expression

Use `-E` for clean alternation, grouping, plus, question mark, and counted repetition:

~~~bash
$ grep -E 'error|warning|critical' app.log
$ grep -E '^(alice|bob):' /etc/passwd
$ grep -E '^[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2}$' dates
~~~

### Essential selection options

~~~bash
$ grep -i 'permission denied' file
$ grep -n 'Listen' /etc/httpd/conf/httpd.conf
$ grep -v 'DEBUG' app.log
$ grep -c 'Failed password' /var/log/secure
$ grep -l 'PermitRootLogin' /etc/ssh/sshd_config.d/*.conf
$ grep -R --include='*.conf' 'AllowUsers' /etc/ssh
$ grep -A 3 -B 2 'error' app.log
$ grep -q '^alice:' /etc/passwd
~~~

`grep -q` prints nothing and communicates through its exit status:

~~~bash
if grep -q '^alice:' /etc/passwd; then
    echo "local entry found"
fi
~~~

For account lookup, `getent passwd alice` is normally better because it includes configured name services.

## 3. Regex that administrators actually use

| Pattern | Meaning |
| --- | --- |
| `^` | beginning of line |
| `$` | end of line |
| `.` | any single character |
| `[abc]` | one character in a set |
| `[^abc]` | one character outside a set |
| `[[:digit:]]` | digit |
| `[[:space:]]` | whitespace |
| `*` | zero or more of previous item |
| `+` in ERE | one or more |
| `?` in ERE | zero or one |
| `{n,m}` in ERE | between n and m |
| `a|b` in ERE | alternative |
| `(group)` in ERE | group |

### Ignore comments and empty lines

~~~bash
$ grep -vE '^[[:space:]]*(#|$)' /etc/ssh/sshd_config
~~~

Read it from inside out:

- `[[:space:]]*` permits indentation;
- `#|$` means a comment marker or the end of an otherwise empty line;
- `^` anchors the pattern at the line start;
- `-v` prints everything that is not such a line.

### Match an exact configuration directive

~~~bash
$ grep -Ei '^[[:space:]]*PermitRootLogin[[:space:]]+' /etc/ssh/sshd_config
~~~

This avoids matching a commented directive or an unrelated word in the middle of a line.

### Limits of regex

Regex can check shape without proving meaning. A simple dotted-quad pattern can accept values above 255. Use a purpose-built command when one exists:

- `ip address` for configured addresses;
- `getent` for names and accounts;
- `systemctl is-active` for service state;
- `findmnt` for mounts.

## 4. cut

`cut` extracts delimited fields or character ranges:

~~~bash
$ cut -d: -f1 /etc/passwd
$ cut -d: -f1,3,7 /etc/passwd
$ printf '%s\n' 'servera.example.com' | cut -d. -f1
~~~

`cut` expects a single-character delimiter. It is excellent for predictable records such as `/etc/passwd`, but not for columns separated by variable whitespace.

## 5. sort and uniq

~~~bash
$ sort names
$ sort -u names
$ sort -n numbers
$ sort -r names
$ sort -t: -k3,3n /etc/passwd
~~~

Count duplicate adjacent lines:

~~~bash
$ sort values | uniq -c | sort -nr
~~~

`uniq` only combines adjacent duplicates, which is why `sort` usually comes first.

Use a predictable locale when bytewise ordering matters:

~~~bash
$ LC_ALL=C sort names
~~~

## 6. wc, head, tail, and tee

~~~bash
$ wc -l /etc/passwd
$ wc -w notes
$ head -n 5 file
$ tail -n 20 /var/log/secure
$ tail -f /var/log/messages
$ command | tee report
$ command | tee -a report
~~~

Count real records rather than display lines when a specialized command exists. For active processes named sshd:

~~~bash
$ pgrep -c sshd
~~~

This is more reliable than counting `ps | grep`.

## 7. tr

`tr` translates or deletes individual characters:

~~~bash
$ printf '%s\n' 'RHEL TEN' | tr '[:upper:]' '[:lower:]'
$ printf '%s\n' '12 34 56' | tr -d ' '
$ printf '%s\n' 'a:::b' | tr -s ':'
~~~

It does not understand words or regex groups. Use `sed` for structured substitutions.

## 8. sed

By default, sed prints transformed output and does not edit the file.

~~~bash
$ sed -n '1,10p' file
$ sed -n '/error/p' app.log
$ sed 's/old/new/' file
$ sed 's/old/new/g' file
$ sed '/^[[:space:]]*#/d; /^[[:space:]]*$/d' config
~~~

Safe editing pattern:

1. print the proposed result;
2. compare it;
3. use an in-place backup only after it is correct.

~~~bash
$ sed 's|^Option=.*|Option=new|' app.conf > app.conf.proposed
$ diff -u app.conf app.conf.proposed
$ sed -i.bak 's|^Option=.*|Option=new|' app.conf
~~~

For critical configuration, a purpose-built command or a careful editor plus syntax checker is usually safer than a broad substitution.

## 9. awk

awk processes records and fields. Its default field separator is runs of whitespace.

~~~bash
$ awk '{print $1, $NF}' file
$ awk -F: '{print $1, $3, $7}' /etc/passwd
$ awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd
$ awk '$5 + 0 >= 80 {print}' report
~~~

Useful built-ins:

- `NR` is the current record number;
- `NF` is the number of fields;
- `$1` is the first field;
- `$NF` is the last field;
- `BEGIN` runs before input;
- `END` runs after input.

Example sum:

~~~bash
$ awk '{sum += $1} END {print sum}' numbers
~~~

Parse portable `df -P` output:

~~~bash
$ df -P / | awk 'NR == 2 {gsub(/%/, "", $5); print $5}'
~~~

Do not parse decorative, unstable output when a command offers selected fields or a quiet status mode.

## 10. paste and formatted reporting

Combine corresponding lines:

~~~bash
$ paste users shells
$ paste -d: users uids
~~~

Format with `printf` in Bash or awk:

~~~bash
$ awk -F: 'BEGIN {printf "%-18s %8s\n", "USER", "UID"}
  {printf "%-18s %8s\n", $1, $3}' /etc/passwd
~~~

## 11. Safe filename processing

Filenames can contain spaces, tabs, wildcard characters, and newlines.

Unsafe:

~~~bash
for file in $(find /srv -type f); do
    command "$file"
done
~~~

Safe null-delimited pipeline:

~~~bash
$ find /srv -type f -print0 | xargs -0 -r stat --
~~~

Or use `find -exec`:

~~~bash
$ find /srv -type f -exec chmod g+r {} +
~~~

Preview before any changing operation:

~~~bash
$ find /srv -type f -print
~~~

Do not parse `ls` output in scripts.

## 12. Journal analysis

Use journal selectors before adding grep:

~~~bash
$ journalctl -u sshd --since today
$ journalctl -b -p warning..alert
$ journalctl _COMM=sudo --since '-1 hour'
~~~

Then filter message content:

~~~bash
$ journalctl -u sshd --since today --no-pager |
    grep -iE 'failed|invalid|accepted'
~~~

Count by message category:

~~~bash
$ journalctl -u sshd --since today --no-pager |
    awk '
      /Failed password/ {failed++}
      /Accepted/ {accepted++}
      END {print "failed=" failed, "accepted=" accepted}
    '
~~~

Use time, boot, unit, process, and priority selectors to reduce data at the source. A smaller, purposeful pipeline is easier to trust.

## 13. Verification patterns for EX200

Use commands designed to answer a state question:

| Question | Verification |
| --- | --- |
| Is service running? | `systemctl is-active SERVICE` |
| Will it start at boot? | `systemctl is-enabled SERVICE` |
| What is default target? | `systemctl get-default` |
| Is a mount active and what options? | `findmnt TARGET` |
| Is fstab syntactically consistent? | `findmnt --verify` |
| Is swap active? | `swapon --show` |
| What owns a file? | `rpm -qf PATH` |
| Is an RPM installed? | `rpm -q PACKAGE` |
| Is a Flatpak installed? | `flatpak info APP_ID` |
| Which profile is active? | `nmcli connection show --active` |
| What routes are used? | `ip route` and `ip -6 route` |
| Does name resolution work? | `getent hosts NAME` |
| Is a port listening? | `ss -lntup` |
| Is a firewall rule active? | `firewall-cmd --query-service` or `--query-port` |
| What is SELinux mode? | `getenforce` |
| What label is present? | `ls -Z` or `ps -eZ` |
| Is sudo valid? | `visudo -c` and `sudo -l -U USER` |
| Is time synchronized? | `chronyc tracking` and `chronyc sources -v` |
| What kernel args are saved? | `grubby --info=ALL` |

Avoid “verification” that simply repeats the changing command.

## 14. Common text-processing mistakes

- Using grep regex when the target is literal; use `grep -F`.
- Forgetting quotes around a regex, allowing shell expansion.
- Counting `ps | grep`, which can count grep itself.
- Using `uniq` without sorting when duplicates are not adjacent.
- Editing in place before previewing.
- Parsing human-formatted output when a quiet or selected-field command exists.
- Splitting filenames on whitespace.
- Treating stderr as normal data without intentional redirection.
- Using a regex to validate a semantic value that has a dedicated parser.

## 15. Cumulative EX200 mock lab

Use fresh RHEL 10 virtual machines and values supplied by your lab. The following is a study mock, not an actual Red Hat exam.

### Rules

- Target time: 2.5 hours.
- Work from the console for boot and network changes.
- Do not use external internet resources while attempting it.
- Record supplied values before starting.
- Every required configuration must persist after reboot.
- Do not look at the solution pattern until the attempt is complete.

### Tasks

1. Create `/root/evidence` and save the hostname, active connections, file systems, and failed units in separate text files.
2. Create group `operators` with a supplied GID.
3. Create user `sam` with a supplied UID, Bash shell, home directory, primary group `operators`, and supplementary group `wheel`.
4. Set Sam's maximum password age to 45 days, warning to 5 days, and force a password change at first login.
5. Create `/srv/operators` with group inheritance and no access for others.
6. Permit `operators` to run only `systemctl status chronyd` through sudo.
7. Configure the supplied RPM repository with GPG checking and install the requested package.
8. Configure the supplied Flatpak remote and install the requested application system-wide.
9. Write `/usr/local/bin/service-report` that accepts one or more service names, prints ACTIVE or INACTIVE for each, prints usage when empty, and returns nonzero if any service is inactive.
10. Make the journal persistent.
11. Enable and start `chronyd`, then configure the supplied NTP server.
12. Select the TuneD profile supplied by the task.
13. Create a GPT partition on an unused disk, initialize it as an LVM PV, and create VG `vgexam` from it.
14. Create LV `lvwork`, format it with the requested XFS size, label it `work`, and mount by UUID at `/work`.
15. Extend `lvwork` and its XFS file system by the supplied amount.
16. Create and persist the requested swap LV.
17. Mount the supplied NFS export persistently at `/mnt/reference`.
18. Configure autofs so `/shares/team` maps to the supplied NFS export.
19. Create a root cron task at the supplied schedule.
20. Create a systemd timer that runs the supplied script daily and catches missed runs.
21. Configure a persistent dual-stack NetworkManager profile, DNS, hostname, autoconnect, and the supplied firewalld zone.
22. Allow the requested service and custom port through that zone.
23. Configure key-based SSH authentication for Sam.
24. Configure `/srv/site` with the correct persistent Apache content context, the requested SELinux web port, one supplied boolean, and enforcing mode now and at boot.
25. Set the default boot target and add the supplied kernel argument to all installed kernels.
26. Reboot and verify every persistent requirement.

## 16. Cumulative solution pattern

This is a workflow, not a substitute for the actual values in the task.

### Evidence and accounts

~~~bash
# mkdir -p /root/evidence
# hostnamectl > /root/evidence/hostname
# nmcli connection show --active > /root/evidence/connections
# lsblk -f > /root/evidence/filesystems
# systemctl --failed > /root/evidence/failed-units
# groupadd -g SUPPLIED_GID operators
# useradd -m -u SUPPLIED_UID -g operators -G wheel -s /bin/bash sam
# passwd sam
# chage -M 45 -W 5 -d 0 sam
# mkdir -p /srv/operators
# chown root:operators /srv/operators
# chmod 2770 /srv/operators
# visudo -f /etc/sudoers.d/operators
# chmod 440 /etc/sudoers.d/operators
# visudo -c
~~~

sudoers line:

~~~sudoers
%operators ALL=(root) /usr/bin/systemctl status chronyd
~~~

### Software

Create the supplied `.repo` file, then:

~~~bash
# dnf clean metadata
# dnf --disablerepo='*' --enablerepo=EXAM_REPO makecache
# dnf --disablerepo='*' --enablerepo=EXAM_REPO install REQUESTED_PACKAGE
# rpm -q REQUESTED_PACKAGE
# flatpak remote-add --if-not-exists EXAM_FLATPAK SUPPLIED_URL
# flatpak search REQUESTED_APP
# flatpak install EXAM_FLATPAK APPLICATION_ID
# flatpak info APPLICATION_ID
~~~

### Script

~~~bash
#!/usr/bin/bash

if [ "$#" -eq 0 ]; then
    printf 'Usage: %s SERVICE...\n' "$0" >&2
    exit 2
fi

result=0
for service in "$@"; do
    if systemctl is-active --quiet "$service"; then
        printf 'ACTIVE %s\n' "$service"
    else
        printf 'INACTIVE %s\n' "$service"
        result=1
    fi
done
exit "$result"
~~~

~~~bash
# chmod 755 /usr/local/bin/service-report
# bash -n /usr/local/bin/service-report
# /usr/local/bin/service-report sshd chronyd nonexistent
~~~

### Journal, time, and tuning

~~~bash
# mkdir -p /etc/systemd/journald.conf.d
# vim /etc/systemd/journald.conf.d/persistent.conf
# systemctl restart systemd-journald
# journalctl --flush
# vim /etc/chrony.conf
# chronyd -p
# dnf install tuned
# systemctl enable --now chronyd tuned
# systemctl restart chronyd
# tuned-adm profile SUPPLIED_PROFILE
$ chronyc sources -v
$ tuned-adm active
~~~

### Storage

~~~bash
# lsblk -e7 -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
# parted UNUSED_DISK print
# parted UNUSED_DISK --script mklabel gpt
# parted UNUSED_DISK --script mkpart primary 1MiB 100%
# parted UNUSED_DISK --script set 1 lvm on
# partprobe UNUSED_DISK
# pvcreate NEW_PARTITION
# vgcreate vgexam NEW_PARTITION
# lvcreate -n lvwork -L SUPPLIED_SIZE vgexam
# mkfs.xfs -L work /dev/vgexam/lvwork
# mkdir -p /work
# blkid /dev/vgexam/lvwork
# vim /etc/fstab
# findmnt --verify
# mount -a
# lvextend -L +SUPPLIED_GROWTH -r /dev/vgexam/lvwork
# lvcreate -n lvswap -L SUPPLIED_SWAP_SIZE vgexam
# mkswap -L examswap /dev/vgexam/lvswap
# blkid /dev/vgexam/lvswap
# vim /etc/fstab
# swapon -a
~~~

### NFS and autofs

~~~bash
# dnf install nfs-utils autofs
# mkdir -p /mnt/reference
# vim /etc/fstab
# findmnt --verify
# mount -a
# vim /etc/auto.master.d/shares.autofs
# vim /etc/auto.shares
# systemctl enable --now autofs
# systemctl reload autofs
$ ls /shares/team
~~~

### Scheduling

Create the cron file with its required user field. Create a oneshot `.service` and a `.timer` with:

~~~ini
[Timer]
OnCalendar=SUPPLIED_CALENDAR
Persistent=true

[Install]
WantedBy=timers.target
~~~

Then:

~~~bash
# systemctl enable --now crond
# systemctl daemon-reload
# systemctl start EXAM_SERVICE.service
# systemctl enable --now EXAM_TIMER.timer
$ systemctl list-timers --all
~~~

### Networking and firewall

~~~bash
# nmcli connection add type ethernet ifname INTERFACE con-name exam-net \
    ipv4.method manual ipv4.addresses IPV4/PREFIX ipv4.gateway IPV4_GW \
    ipv4.dns "DNS1 DNS2" ipv4.dns-search DOMAIN \
    ipv6.method manual ipv6.addresses IPV6/PREFIX ipv6.gateway IPV6_GW \
    connection.autoconnect yes
# nmcli connection modify exam-net connection.zone ZONE
# nmcli connection up exam-net
# hostnamectl set-hostname FQDN
# vim /etc/hosts
# systemctl enable --now firewalld sshd
# firewall-cmd --permanent --zone=ZONE --add-service=REQUESTED_SERVICE
# firewall-cmd --permanent --zone=ZONE --add-port=PORT/tcp
# firewall-cmd --reload
~~~

### SSH and SELinux

~~~bash
$ ssh-keygen -t ed25519
$ ssh-copy-id sam@TARGET
$ ssh sam@TARGET
# dnf install policycoreutils-python-utils
# semanage fcontext -a -t httpd_sys_content_t '/srv/site(/.*)?'
# restorecon -Rv /srv/site
# semanage port -a -t http_port_t -p tcp SUPPLIED_PORT
# setsebool -P SUPPLIED_BOOLEAN on
# setenforce 1
# vim /etc/selinux/config
~~~

### Target and kernel argument

~~~bash
# systemctl set-default SUPPLIED_TARGET
# grubby --update-kernel=ALL --args="SUPPLIED_ARGUMENT"
# grubby --info=ALL
~~~

## 17. Pre-reboot validation

Run before the mock's final reboot:

~~~bash
# findmnt --verify
# mount -a
# swapon -a
# visudo -c
# sshd -t
# chronyd -p
# systemctl --failed
# systemctl get-default
# systemctl list-timers --all
# nmcli connection show
# firewall-cmd --get-active-zones
# firewall-cmd --zone=ZONE --list-all
# getenforce
# semanage fcontext -C -l
# semanage port -l
# grubby --info=ALL
~~~

Also verify:

~~~bash
# pvs
# vgs
# lvs
# findmnt /work
# findmnt /mnt/reference
# swapon --show
# systemctl is-enabled chronyd
# systemctl is-enabled autofs
# sudo -l -U sam
# rpm -q REQUESTED_PACKAGE
# flatpak info APPLICATION_ID
~~~

Only then reboot:

~~~bash
# systemctl reboot
~~~

After boot, repeat the state checks and inspect:

~~~bash
$ journalctl --list-boots
$ journalctl -b -p err
$ systemctl --failed
$ cat /proc/cmdline
~~~

## 18. Final objective checklist

You are ready for a serious practice attempt when you can perform each item without copying:

- shell syntax, redirection, grep, SSH, archives, files, links, modes, documentation;
- RPM repositories and packages;
- Flatpak remotes and applications;
- script arguments, conditions, loops, and command output;
- boot targets, recovery, processes, niceness, TuneD, logs, persistent journal, secure copy;
- GPT, PV, VG, LV, UUID/label mounts, partitions, LVs, and swap;
- VFAT, ext4, XFS, NFS, autofs, LV and file-system growth, permission diagnosis;
- at, cron, timers, services, time client, package deployment, boot-loader changes;
- IPv4, IPv6, hostname resolution, autoconnect, firewalld;
- local users, groups, password aging, membership, sudo;
- umask, SSH keys, firewalld, SELinux modes, contexts, restorecon, ports, and booleans.

## Official references

- [Current EX200 objectives](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)
- [RHEL 10 documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)
- Local pages: `man grep`, `man 7 regex`, `man sed`, `man awk`, `man cut`, `man sort`, `man uniq`, `man tr`, `man xargs`, and `man journalctl`
