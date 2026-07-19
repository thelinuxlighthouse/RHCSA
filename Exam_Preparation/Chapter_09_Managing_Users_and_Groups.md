# Chapter 9: Managing Users and Groups

## Objectives

After this chapter, you should be able to:

- create, inspect, modify, lock, and delete local user accounts;
- set passwords and password-aging rules;
- create, rename, renumber, and delete local groups;
- manage primary and supplementary membership without losing existing groups;
- understand the account databases and default files;
- configure privileged access through `sudo`; and
- verify access as the target user.

## 1. Local account databases

### `/etc/passwd`

Each line contains:

~~~text
name:password-placeholder:UID:primary-GID:comment:home:shell
~~~

Example:

~~~text
alice:x:1001:1001:Alice Admin:/home/alice:/bin/bash
~~~

The `x` means the password hash is stored in protected `/etc/shadow`.

### `/etc/shadow`

Contains password hashes and aging fields. Do not edit it directly in normal administration.

### `/etc/group`

~~~text
group-name:password-placeholder:GID:comma-separated-supplementary-members
~~~

A user's primary group relationship is recorded by GID in `/etc/passwd`, so the user may not appear in the member list of `/etc/group`.

### `/etc/gshadow`

Contains protected group security information.

Query through the name service layer:

~~~bash
$ getent passwd alice
$ getent group developers
$ id alice
$ groups alice
~~~

`getent` is preferable to grepping the local files when the system may also use directory services.

Use safe database editors when direct repair is required:

~~~bash
# vipw
# vigr
# pwck -r
# grpck -r
~~~

## 2. Account defaults

Inspect:

~~~bash
# useradd -D
# cat /etc/default/useradd
# grep -E '^(UID_MIN|UID_MAX|GID_MIN|GID_MAX|CREATE_HOME|USERGROUPS_ENAB|PASS_)' /etc/login.defs
# ls -la /etc/skel
~~~

Important distinction:

- `/etc/default/useradd` and `/etc/login.defs` influence new accounts;
- `/etc/skel` supplies initial home-directory files;
- changing defaults does not retroactively modify existing users.

## 3. Create users

### Basic interactive user

~~~bash
# useradd -m alice
# passwd alice
# id alice
# getent passwd alice
# ls -ld /home/alice
~~~

Useful options:

~~~bash
# useradd -m -c "Alice Admin" -s /bin/bash alice
# useradd -m -u 1500 alice
# useradd -m -g developers -G wheel,project alice
# useradd -m -e 2026-12-31 -f 7 alice
# useradd -m -d /srv/homes/alice alice
~~~

Meanings:

- `-m` creates the home;
- `-c` sets the comment;
- `-s` sets the login shell;
- `-u` chooses a UID;
- `-g` selects the primary group, which must exist;
- `-G` sets supplementary groups;
- `-e` sets account expiration as YYYY-MM-DD;
- `-f` sets inactive days after password expiration;
- `-d` selects a home path.

Use a unique UID unless the task explicitly requires a particular existing identity mapping.

### System or service account

~~~bash
# useradd --system --no-create-home \
    --shell /usr/sbin/nologin \
    --user-group inventorysvc
~~~

Then verify instead of assuming a particular numeric UID:

~~~bash
# id inventorysvc
# getent passwd inventorysvc
~~~

A service account normally has no interactive shell and may not need a home.

## 4. Modify users

~~~bash
# usermod -c "Alice Operations" alice
# usermod -s /bin/bash alice
# usermod -d /srv/homes/alice -m alice
# usermod -u 1600 alice
# usermod -g operations alice
# usermod -aG wheel,project alice
~~~

Critical warning:

~~~bash
# usermod -G project alice
~~~

sets the supplementary group list to only `project`. Use `-aG` to append without removing existing memberships.

If you change a UID, files on other file systems can retain the old numeric owner. Locate carefully:

~~~bash
# find / -xdev -uid OLD_UID -ls
~~~

Repeat on other intended file systems rather than crossing every network and virtual mount.

### Rename an account

~~~bash
# usermod -l alicia alice
# usermod -d /home/alicia -m alicia
~~~

Renaming the login does not automatically rename every related object such as a same-named group, cron entry, mail alias, sudo rule, or application configuration. Audit those separately.

### Lock and unlock password authentication

~~~bash
# usermod -L alice
# usermod -U alice
# passwd -l alice
# passwd -u alice
# passwd -S alice
~~~

Password locking does not necessarily block SSH public-key login or every other authentication method. If the requirement is complete account expiration:

~~~bash
# chage -E 0 alice
~~~

To restore no account expiration:

~~~bash
# chage -E -1 alice
~~~

Use the exact control the task asks for: password lock, account expiry, login shell, or removal.

## 5. Delete users

~~~bash
# userdel alice
# userdel -r alice
~~~

`-r` removes the home directory and mail spool known to the account tools. It does not safely find and delete every file owned by the UID across all storage.

Before deletion:

~~~bash
# pgrep -u alice -a
# find /home /srv -xdev -user alice -ls
~~~

After deletion, files can display only the old numeric UID. Decide whether to archive, reassign, or remove them according to the task.

## 6. Passwords and aging

Set or change:

~~~bash
# passwd alice
$ passwd
~~~

Never place a plaintext password in a command line or script. It can appear in history, process listings, logs, or saved files.

Inspect:

~~~bash
# passwd -S alice
# chage -l alice
~~~

Set aging:

~~~bash
# chage -m 1 -M 90 -W 7 -I 14 alice
# chage -E 2026-12-31 alice
~~~

Meanings:

- minimum 1 day between password changes;
- maximum 90 days before expiration;
- warn for 7 days;
- disable after 14 inactive days following password expiration;
- expire the account on the specified date.

Force password change at next login:

~~~bash
# chage -d 0 alice
~~~

Remove maximum-age enforcement:

~~~bash
# chage -M -1 alice
~~~

Password expiry and account expiry are different. Always read `chage -l`.

Defaults in `/etc/login.defs` generally apply when new accounts are created. Use `chage` for existing accounts.

## 7. Create and manage groups

Create:

~~~bash
# groupadd developers
# groupadd -g 2500 project
# getent group developers
~~~

Modify:

~~~bash
# groupmod -n engineering developers
# groupmod -g 2600 project
~~~

Changing a GID does not automatically fix all files that store the old numeric GID. Locate and reassign required files.

Delete:

~~~bash
# groupdel engineering
~~~

A group cannot normally be deleted while it is the primary group of a user. Move those users to another primary group first.

## 8. Membership

Add supplementary membership:

~~~bash
# usermod -aG project alice
# gpasswd -a bob project
~~~

Remove one supplementary membership:

~~~bash
# gpasswd -d alice project
~~~

Change the primary group:

~~~bash
# usermod -g project alice
~~~

Verify:

~~~bash
# id alice
# getent group project
~~~

Existing login sessions keep their group list. The user should log out and back in, or a new process can be tested:

~~~bash
# su - alice
$ id
~~~

## 9. Configure privileged access

RHEL normally grants broad sudo access to members of `wheel` through the packaged sudoers configuration.

~~~bash
# usermod -aG wheel alice
# sudo -l -U alice
~~~

For task-specific access, create a drop-in rather than editing many unrelated lines.

Open safely:

~~~bash
# visudo -f /etc/sudoers.d/operations
~~~

Examples:

~~~sudoers
# One user can run all commands after authenticating
alice ALL=(ALL) ALL

# Members of operations can manage selected services as root
%operations ALL=(root) /usr/bin/systemctl status httpd, /usr/bin/systemctl restart httpd
~~~

Then:

~~~bash
# chmod 440 /etc/sudoers.d/operations
# visudo -c
# sudo -l -U alice
~~~

Important points:

- `%operations` refers to a group;
- command paths must be absolute;
- arguments in a sudoers command specification matter;
- `NOPASSWD` removes password prompting and should be used only when explicitly required;
- `visudo -c` validates the complete configuration.

Test as the user:

~~~bash
# su - alice
$ sudo -l
$ sudo systemctl status httpd
~~~

Keep another root shell open while changing sudo.

## 10. Shared team directory

Create a directory where team files remain group-owned:

~~~bash
# groupadd project
# usermod -aG project alice
# usermod -aG project bob
# mkdir -p /srv/project
# chown root:project /srv/project
# chmod 2770 /srv/project
~~~

The leading `2` sets set-group-ID on the directory. New items inherit group `project`.

For group write on newly created files, members also need a cooperative umask such as `0002`, or a default ACL where required. Chapter 10 covers default permissions.

Verify as both users:

~~~bash
# sudo -u alice touch /srv/project/alice-file
# sudo -u bob touch /srv/project/bob-file
# ls -l /srv/project
~~~

## 11. Troubleshooting

### User cannot log in

~~~bash
# getent passwd alice
# passwd -S alice
# chage -l alice
# id alice
# getent passwd alice | cut -d: -f7
# ls -ld /home/alice
# journalctl -u sshd --since today
# tail -n 50 /var/log/secure
~~~

Check whether the account exists, is locked or expired, has a valid shell, has a reachable home, and is being denied by the relevant authentication service.

### Group access does not work

~~~bash
# id alice
# namei -om /srv/project/file
# getfacl -p /srv/project/file
~~~

The most common causes are an old login session, missing directory execute permission, wrong ownership, a restrictive ACL, a read-only mount, or SELinux.

### sudo fails

~~~bash
# visudo -c
# sudo -l -U alice
# journalctl _COMM=sudo --since today
~~~

Check the exact command path and arguments, not only the user name.

## 12. Exam lab

Tasks:

1. Create group `analysts` with the supplied GID.
2. Create `alice` with a home, Bash shell, requested UID, primary group `analysts`, and supplementary group `wheel`.
3. Set maximum password age to 60 days, warning to 7 days, and force a change at first login.
4. Create `/srv/analysts` so new files inherit group `analysts`.
5. Permit group `analysts` to run only `systemctl status chronyd` through sudo.

Solution pattern:

~~~bash
# groupadd -g SUPPLIED_GID analysts
# useradd -m -u SUPPLIED_UID -g analysts -G wheel -s /bin/bash alice
# passwd alice
# chage -M 60 -W 7 -d 0 alice
# mkdir -p /srv/analysts
# chown root:analysts /srv/analysts
# chmod 2770 /srv/analysts
# visudo -f /etc/sudoers.d/analysts
~~~

sudoers content:

~~~sudoers
%analysts ALL=(root) /usr/bin/systemctl status chronyd
~~~

Verify:

~~~bash
# chmod 440 /etc/sudoers.d/analysts
# visudo -c
# id alice
# chage -l alice
# ls -ld /srv/analysts
# sudo -l -U alice
# sudo -u alice touch /srv/analysts/test
# ls -l /srv/analysts/test
~~~

## Quick check

- Can you explain primary versus supplementary group storage?
- Can you append membership without deleting existing groups?
- Can you distinguish password lock from account expiration?
- Can you apply password aging to an existing account?
- Can you rename a user without assuming every related object is renamed?
- Can you create a sudo drop-in, validate it, and test the exact allowed command?
- Can you make a shared directory inherit its group?

## Official references

- [RHEL 10 documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)
- Local pages: `man 5 passwd`, `man 5 shadow`, `man useradd`, `man usermod`, `man userdel`, `man groupadd`, `man chage`, `man sudoers`, and `man visudo`

