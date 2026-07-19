# Chapter 1: Understanding and Using Essential Tools

## Objectives

After this chapter, you should be able to:

- enter commands with correct syntax and find commands in the search path;
- redirect standard input, standard output, and standard error;
- build useful pipelines and use grep with basic and extended regular expressions;
- create, inspect, copy, move, and remove files and directories;
- create hard links and symbolic links;
- archive and compress data with tar, gzip, and bzip2;
- edit text safely with Vim;
- connect through SSH and switch users;
- read and change ordinary user, group, and other permissions; and
- find answers in man, info, and /usr/share/doc.

## 1. Think in paths, commands, and state

A shell is a text interface to the operating system. Bash reads a command, expands variables and wildcards, applies redirections, finds the program, and waits for the program to finish.

The usual syntax is:

~~~text
command [options] [arguments]
~~~

Linux is case-sensitive. The files Report and report are different. Options usually begin with a hyphen. A double hyphen normally introduces a long option. Use `--` to end option processing when a filename begins with a hyphen:

~~~bash
$ touch -- -notes
$ rm -- -notes
~~~

Useful orientation commands:

~~~bash
$ whoami
$ id
$ hostname
$ pwd
$ date
$ echo "$SHELL"
$ type cd
$ type ls
$ command -v ssh
$ printenv PATH
~~~

The shell searches directories in PATH from left to right. A command in the current directory is not run merely by typing its name unless the current directory is in PATH. Use an explicit path:

~~~bash
$ ./check-system
~~~

### Quoting and expansion

Use quoting deliberately:

| Form | Meaning |
| --- | --- |
| unquoted text | whitespace separates words; variables and wildcards expand |
| "double quotes" | keeps the result as one word but expands variables and command substitution |
| 'single quotes' | treats nearly every character literally |
| backslash | protects the next character |

~~~bash
$ name="Ahmed Elsers"
$ printf '%s\n' "$name"
$ printf '%s\n' 'The value of $name is not expanded here'
$ printf 'Today is %s\n' "$(date +%F)"
~~~

Always quote a variable that may contain spaces:

~~~bash
$ cp -- "$source_file" "$destination_dir/"
~~~

### Wildcards are not regular expressions

The shell expands filename wildcards before the command runs:

| Pattern | Matches |
| --- | --- |
| `*` | zero or more characters |
| `?` | exactly one character |
| `[abc]` | one listed character |
| `[0-9]` | one character in the range |
| `[!0-9]` | one character not in the range |

~~~bash
$ ls /etc/*.conf
$ ls report?.txt
~~~

Use `printf '%s\n' pattern` before a risky operation to see what a wildcard expands to.

## 2. Files and directories

### Absolute and relative paths

An absolute path begins at `/`. A relative path begins at the current directory.

~~~bash
$ pwd
/home/student
$ cd /etc
$ cd ssh
$ cd ..
$ cd -
$ cd
~~~

Special names:

- `.` is the current directory.
- `..` is the parent directory.
- `~` is the current user's home.
- `~alice` is Alice's home when the account exists.

### List and inspect

~~~bash
$ ls
$ ls -lah /var/log
$ stat /etc/passwd
$ file /usr/bin/ls
$ du -sh /var/log
$ df -hT
~~~

`ls -l` shows, in order, the file type and permissions, link count, owner, group, size, modification time, and name.

### Create, copy, move, and remove

~~~bash
$ touch notes.txt
$ mkdir reports
$ mkdir -p project/{docs,logs,scripts}
$ cp notes.txt reports/
$ cp -a project project.backup
$ mv notes.txt notes.old
$ rm notes.old
$ rm -r project.backup
$ rmdir empty-directory
~~~

Important choices:

- `cp -a` preserves metadata and recursively copies directories.
- `cp -i`, `mv -i`, and `rm -i` ask before overwriting or removing.
- `rm -r` recursively removes a tree. Inspect the exact path first.
- `rm -f` does not mean “more correct”; it suppresses prompts and some errors.

### Safe real-life example: prepare an application tree

The application team needs separate directories for configuration, logs, and data.

~~~bash
# mkdir -p /srv/inventory/{config,logs,data}
# touch /srv/inventory/config/inventory.conf
# cp -a /srv/inventory /root/inventory.initial
# find /srv/inventory -maxdepth 2 -printf '%M %u:%g %p\n'
~~~

Why this feels natural:

1. `mkdir -p` creates the whole shape in one operation.
2. `touch` creates the required configuration file without changing it if it already exists.
3. `cp -a` records a metadata-preserving starting copy.
4. `find` verifies the tree instead of assuming it exists.

## 3. Standard streams, redirection, and pipelines

Every process starts with three standard file descriptors:

| Number | Name | Normal destination |
| --- | --- | --- |
| 0 | standard input, stdin | keyboard |
| 1 | standard output, stdout | terminal |
| 2 | standard error, stderr | terminal |

### Redirection

~~~bash
$ date > report.txt
$ uptime >> report.txt
$ find /etc -name '*.conf' > found.txt 2> denied.txt
$ find /etc -name '*.conf' &> all-output.txt
$ sort < names.txt
$ command > /dev/null 2>&1
~~~

The order of redirections matters:

~~~bash
$ command > all.txt 2>&1
~~~

First stdout is sent to all.txt, and then stderr is sent to the same place as stdout.

~~~bash
$ command 2>&1 > all.txt
~~~

Here stderr is first copied to the terminal, and only stdout is later sent to all.txt. This is usually not what the administrator intended.

Use `tee` when output must be visible and saved:

~~~bash
$ ip address show | tee interface-report.txt
$ printf '%s\n' '192.0.2.10 servera.example.com servera' | sudo tee -a /etc/hosts
~~~

The second example matters because shell redirection is performed by your current shell. `sudo echo ... >> /etc/hosts` does not make the redirection privileged; `sudo tee` does.

### Pipelines

A pipe connects stdout on the left to stdin on the right:

~~~bash
$ ps -ef | grep '[s]shd'
$ journalctl -u sshd --since today | grep -iE 'failed|error'
$ cut -d: -f1 /etc/passwd | sort
~~~

Avoid an unnecessary cat:

~~~bash
$ grep bash /etc/passwd
~~~

is simpler than:

~~~bash
$ cat /etc/passwd | grep bash
~~~

In scripts, use `set -o pipefail` when failure of any command in a pipeline should make the pipeline fail.

## 4. grep and regular expressions

`grep` prints lines that match a pattern.

~~~bash
$ grep root /etc/passwd
$ grep -i error application.log
$ grep -n '^server' /etc/hosts
$ grep -vE '^[[:space:]]*(#|$)' /etc/ssh/sshd_config
$ grep -R --include='*.conf' 'PermitRootLogin' /etc/ssh
~~~

Essential options:

| Option | Purpose |
| --- | --- |
| `-i` | ignore case |
| `-v` | select nonmatching lines |
| `-n` | show line numbers |
| `-c` | count matching lines |
| `-l` | print names of files with matches |
| `-r` or `-R` | search recursively |
| `-w` | match a whole word |
| `-F` | treat the pattern as a fixed string |
| `-E` | use extended regular expressions |
| `-A`, `-B`, `-C` | show after, before, or surrounding context |

### Core patterns

| Pattern | Meaning |
| --- | --- |
| `^root` | root at the beginning of a line |
| `bash$` | bash at the end of a line |
| `.` | any single character |
| `[abc]` | one character from the set |
| `[^abc]` | one character outside the set |
| `[[:digit:]]` | a digit |
| `*` | zero or more of the previous item |
| `+` with grep -E | one or more |
| `?` with grep -E | zero or one |
| `{3}` with grep -E | exactly three |
| `a|b` with grep -E | a or b |
| `(ab)+` with grep -E | a repeated group |

Keep regex in single quotes so Bash does not interpret it:

~~~bash
$ grep -E '^(root|student):' /etc/passwd
$ grep -E '^[[:digit:]]{1,3}(\.[[:digit:]]{1,3}){3}$' addresses.txt
~~~

Regex validates the shape, not necessarily the meaning; the second pattern would also accept 999.999.999.999.

## 5. Text files and Vim

You may use any available editor, but Vim is common in minimal RHEL installations.

Open a file:

~~~bash
$ vim notes.txt
~~~

Essential Vim actions:

| Action | Keys |
| --- | --- |
| enter insert mode | `i` |
| return to normal mode | Esc |
| save | `:w` |
| save and exit | `:wq` |
| exit when unchanged | `:q` |
| abandon changes | `:q!` |
| search forward | `/pattern`, then Enter |
| next match | `n` |
| show line numbers | `:set number` |
| go to line 20 | `:20` |
| delete current line | `dd` |
| copy current line | `yy` |
| paste after cursor | `p` |
| undo | `u` |

For structured configuration, use a syntax checker before replacing the working file. Examples include `visudo -c`, `sshd -t`, `chronyd -p`, and `findmnt --verify`.

## 6. Links

### Hard links

~~~bash
$ printf '%s\n' 'important data' > original
$ ln original second-name
$ ls -li original second-name
~~~

Both names refer to the same inode. Changing either changes the same file data. A hard link normally cannot cross file systems and should not be created for directories.

### Symbolic links

~~~bash
$ ln -s /var/log/messages current-log
$ readlink current-log
$ readlink -f current-log
~~~

A symbolic link stores a path. It can cross file systems and point to a directory, but it breaks if its target path no longer exists.

Real-life choice:

- Use a symbolic link for `/opt/app/current` pointing to a versioned release directory.
- Use a hard link when two file names on the same file system must continue to reach the same data even if one name is removed.

## 7. Permissions and ownership

### Read the mode

~~~bash
$ ls -ld shared report.txt
-rw-r-----. 1 alice finance 1200 Jul 19 09:00 report.txt
~~~

The first character is the file type. The next nine characters are owner, group, and other permissions.

For an ordinary file:

- read means read content;
- write means change content;
- execute means run it as a program.

For a directory:

- read means list names;
- write means create, rename, or remove entries;
- execute means traverse the directory and access known names.

Directory write without execute is usually not useful. A user can delete a file from a writable directory even without write permission on the file, because deletion changes the directory entry.

### Change mode

~~~bash
$ chmod u+x backup.sh
$ chmod g=rw,o= report.txt
$ chmod 640 report.txt
$ chmod -R g+rX project
~~~

`X` adds execute only to directories and to files that already have an execute bit. It is safer than recursively making every file executable.

Octal values:

~~~text
read 4 + write 2 + execute 1
750 = rwxr-x---
640 = rw-r-----
~~~

### Change owner and group

~~~bash
# chown alice report.txt
# chown alice:finance report.txt
# chgrp finance report.txt
# chown -R alice:finance /srv/reports
~~~

Verify from the perspective of the target user:

~~~bash
# sudo -u alice test -r /srv/reports/monthly.txt && echo readable
# namei -om /srv/reports/monthly.txt
~~~

`namei -om` is excellent for finding a missing directory traversal permission in a long path.

Special permissions and default permissions are covered in Chapter 10.

## 8. Archives and compression

An archive combines file names, data, and metadata. Compression reduces size. `tar` can do both.

~~~bash
$ tar -cf etc-backup.tar /etc/hosts /etc/fstab
$ tar -tf etc-backup.tar
$ mkdir restore
$ tar -xf etc-backup.tar -C restore
~~~

Compressed tar archives:

~~~bash
$ tar -czf logs.tar.gz /var/log
$ tar -cjf logs.tar.bz2 /var/log
$ tar -tzf logs.tar.gz
$ tar -xjf logs.tar.bz2 -C restore
~~~

Remember:

- `c` create;
- `t` list;
- `x` extract;
- `f` archive file follows;
- `z` gzip;
- `j` bzip2;
- `v` verbose, optional;
- `C` change directory before the operation.

Standalone compression:

~~~bash
$ gzip large.log
$ gzip -d large.log.gz
$ bzip2 large.log
$ bzip2 -d large.log.bz2
~~~

Do not extract an untrusted archive as root without inspecting its member names first.

## 9. SSH, switching users, and secure copies

Connect:

~~~bash
$ ssh student@serverb
$ ssh -p 2222 student@serverb
$ ssh serverb hostname
~~~

On the first connection, compare the displayed host fingerprint through a trusted channel before accepting it. The known host key is stored in `~/.ssh/known_hosts`.

Copy securely:

~~~bash
$ scp report.txt student@serverb:/tmp/
$ scp student@serverb:/etc/hosts ./serverb.hosts
$ sftp student@serverb
~~~

Chapter 4 covers remote transfers further, and Chapter 10 covers key-based authentication.

Switch identity:

~~~bash
$ su - alice
$ sudo -i
$ sudo -u alice id
~~~

`su - alice` starts a login-like environment for Alice. Plain `su alice` retains more of the current environment and is more likely to produce confusing results.

See who is logged in:

~~~bash
$ who
$ w
$ loginctl list-sessions
~~~

## 10. Find files and documentation

### Find files by properties

~~~bash
$ find /etc -type f -name '*.conf'
$ find /var/log -type f -size +10M
$ find /home -xdev -user alice
$ find /tmp -type f -mtime +7 -print
~~~

Use `-xdev` when you do not want to cross into other mounted file systems.

`locate` searches a database and can be faster, but the database may be outdated:

~~~bash
# dnf install plocate
# updatedb
$ locate sshd_config
~~~

### Use local documentation

~~~bash
$ man passwd
$ man 5 passwd
$ man -k 'logical volume'
$ apropos 'copy files'
$ info coreutils
$ ls /usr/share/doc
$ rpm -qd openssh-server
$ command --help
~~~

Man-page sections commonly needed:

| Section | Content |
| --- | --- |
| 1 | user commands |
| 5 | file formats and configuration |
| 8 | administrative commands |

Examples:

~~~bash
$ man 1 passwd
$ man 5 passwd
$ man 8 useradd
~~~

Search inside a man page with `/pattern`, move to the next match with `n`, and quit with `q`.

## 11. Troubleshooting method

When “permission denied” appears:

1. Run `id` to confirm the effective user and groups.
2. Run `namei -om path` to inspect every directory.
3. Run `ls -l` and `getfacl` on the object.
4. Check whether the file system is mounted read-only with `findmnt`.
5. Check SELinux context and denials as described in Chapter 10.

When “command not found” appears:

1. Check spelling and case.
2. Run `type -a command` and `command -v command`.
3. Inspect PATH.
4. Search the owning package with `dnf provides /usr/bin/command`.

## 12. Guided lab: prepare and transfer an incident bundle

Task: create a bundle containing host identity, disk usage, failed services, and recent SSH messages. The bundle must be readable only by its owner and then copied to serverb.

~~~bash
$ mkdir -p ~/incident
$ hostnamectl > ~/incident/host.txt
$ df -hT > ~/incident/filesystems.txt
$ systemctl --failed > ~/incident/failed-units.txt
$ journalctl -u sshd --since today > ~/incident/sshd-today.txt
$ grep -iE 'failed|error|denied' ~/incident/sshd-today.txt > ~/incident/important.txt
$ chmod -R go-rwx ~/incident
$ tar -czf ~/incident-$(date +%F).tar.gz -C ~ incident
$ tar -tzf ~/incident-$(date +%F).tar.gz
$ scp ~/incident-$(date +%F).tar.gz student@serverb:/tmp/
~~~

Why each step exists:

- Redirection makes a repeatable evidence file.
- grep reduces the log to likely problems without destroying the full copy.
- permissions protect potentially sensitive information.
- `tar -C ~ incident` stores a clean relative path instead of `/home/user/incident`.
- listing the archive verifies it before transfer.

## 13. Exam drills

Perform without looking at the solution:

1. Create `/root/review/day1` and place copies of `/etc/hosts` and `/etc/fstab` inside it.
2. Create a symbolic link `/root/current-review` pointing to that directory.
3. Create `/root/bash-users` containing only account names whose login shell is Bash.
4. Create a gzip-compressed tar archive at `/root/day1.tar.gz`.
5. Set the directory so only root can access it.

One correct solution:

~~~bash
# mkdir -p /root/review/day1
# cp /etc/hosts /etc/fstab /root/review/day1/
# ln -s /root/review/day1 /root/current-review
# awk -F: '$7 == "/bin/bash" { print $1 }' /etc/passwd > /root/bash-users
# tar -czf /root/day1.tar.gz -C /root/review day1
# chmod 700 /root/review/day1
# ls -ld /root/review/day1 /root/current-review
# cat /root/bash-users
# tar -tzf /root/day1.tar.gz
~~~

## Quick check

- Can you explain the difference between `>` and `>>`?
- Can you send stdout and stderr to different files?
- Can you explain why `sudo echo text >> /root/file` fails?
- Can you distinguish a shell wildcard from a regular expression?
- Can you explain directory read, write, and execute permissions?
- Can you create and inspect both link types?
- Can you create, list, and extract `.tar.gz` and `.tar.bz2` archives?
- Can you find the file-format man page for `/etc/fstab`?

## Official references

- [EX200 objectives](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)
- [RHEL 10 documentation collection](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)

