# Chapter 10: Managing Security

## Objectives

After this chapter, you should be able to:

- calculate and configure default file permissions with umask;
- use set-user-ID, set-group-ID, and sticky permissions appropriately;
- configure and verify SSH public-key authentication;
- manage firewalld zones, services, ports, and persistent rules;
- switch SELinux between enforcing and permissive modes now and at boot;
- identify SELinux process and file contexts;
- restore and define persistent file-context mappings;
- manage SELinux network port labels;
- inspect and persist SELinux booleans; and
- troubleshoot a denial without disabling security controls.

## 1. Security is layered

When access fails, several independent controls may be involved:

1. account identity and groups;
2. Unix mode bits and ACLs;
3. mount options;
4. service configuration;
5. firewalld;
6. SELinux policy and labels.

Passing one layer does not bypass the others. A world-writable file can still be blocked by SELinux. An allowed firewall port does not make a service listen.

## 2. Default permissions and umask

New regular files start from a maximum of `0666`; new directories start from `0777`. The umask removes requested bits.

Example:

~~~text
file:      0666 minus mask 0027 -> 0640
directory: 0777 minus mask 0027 -> 0750
~~~

Inspect:

~~~bash
$ umask
$ umask -S
~~~

Set for the current shell:

~~~bash
$ umask 0027
$ touch test-file
$ mkdir test-directory
$ stat -c '%a %n' test-file test-directory
~~~

A common mistake is to subtract as ordinary decimal arithmetic. Think “the mask removes permission bits.”

### Persistent interactive-shell umask

For one Bash user, place it in an appropriate login or interactive startup file, commonly `~/.bashrc` according to the environment:

~~~bash
umask 0027
~~~

For a site-wide login-shell policy, create a clearly named profile fragment:

~~~bash
# vim /etc/profile.d/site-umask.sh
~~~

Contents:

~~~bash
umask 0027
~~~

Test with a new login session. Existing shells keep their current umask.

### systemd service umask

Services do not normally read interactive shell profiles. Create a drop-in:

~~~bash
# systemctl edit SERVICE
~~~

~~~ini
[Service]
UMask=0027
~~~

Then:

~~~bash
# systemctl daemon-reload
# systemctl restart SERVICE
# systemctl show SERVICE -p UMask
~~~

## 3. Special mode bits

### set-user-ID on a file

An executable can run with the file owner's effective UID:

~~~bash
$ ls -l /usr/bin/passwd
~~~

Do not add set-user-ID to arbitrary programs or scripts. It is a strong privilege mechanism.

Set symbolically or numerically:

~~~bash
# chmod u+s PROGRAM
# chmod 4755 PROGRAM
~~~

### set-group-ID

On an executable, it selects the file group's effective GID. On a directory, it makes new entries inherit the directory's group:

~~~bash
# chown root:project /srv/project
# chmod 2770 /srv/project
~~~

### sticky bit

On a shared writable directory, users can normally remove only entries they own, entries owned by the directory owner, or entries removed by root:

~~~bash
# chmod 1777 /srv/dropbox
$ ls -ld /tmp /srv/dropbox
~~~

Sticky does not make file contents private. It mainly controls removal and rename within the directory.

## 4. SSH public-key authentication

The private key stays with the client. The public key is copied to the target account.

### Generate a key

As the client user:

~~~bash
$ ssh-keygen -t ed25519
~~~

Accept the default path unless the task requires another. Protect the private key with a passphrase in real administration. If the system is in a mode whose crypto policy does not permit Ed25519, use an allowed key type such as an appropriately sized RSA key according to local policy.

Files:

~~~text
~/.ssh/id_ed25519       private key
~/.ssh/id_ed25519.pub   public key
~~~

Never copy or publish the private key.

### Install the public key

When password login is temporarily available:

~~~bash
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@serverb
~~~

Test:

~~~bash
$ ssh -i ~/.ssh/id_ed25519 alice@serverb
~~~

### Manual server-side installation

As or for the target account:

~~~bash
$ mkdir -p ~/.ssh
$ chmod 700 ~/.ssh
$ vim ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ restorecon -RFv ~/.ssh
~~~

If root created the files for Alice:

~~~bash
# chown -R alice:alice /home/alice/.ssh
# chmod 700 /home/alice/.ssh
# chmod 600 /home/alice/.ssh/authorized_keys
# restorecon -RFv /home/alice/.ssh
~~~

The home directory and every parent path must also permit Alice and `sshd` to traverse appropriately.

### Server configuration

Inspect effective settings:

~~~bash
# sshd -T | grep -E 'pubkeyauthentication|authorizedkeysfile|passwordauthentication'
~~~

Use a drop-in such as `/etc/ssh/sshd_config.d/60-local.conf`:

~~~sshdconfig
PubkeyAuthentication yes
~~~

Before reload:

~~~bash
# sshd -t
# systemctl reload sshd
~~~

Test key login in a second terminal before disabling password authentication or closing the working session.

### Troubleshoot key login

Client detail:

~~~bash
$ ssh -vv alice@serverb
~~~

Server checks:

~~~bash
# getent passwd alice
# namei -om /home/alice/.ssh/authorized_keys
# ls -ldZ /home/alice /home/alice/.ssh
# ls -lZ /home/alice/.ssh/authorized_keys
# sshd -t
# journalctl -u sshd --since today
# tail -n 50 /var/log/secure
~~~

Typical causes are wrong ownership, overly open key-file permissions, wrong account, wrong public key, invalid server configuration, wrong SELinux context, or crypto-policy restrictions.

## 5. firewalld security

Inspect the zone that actually handles the connection:

~~~bash
$ firewall-cmd --state
$ firewall-cmd --get-active-zones
$ firewall-cmd --zone=public --list-all
~~~

Persist a service:

~~~bash
# firewall-cmd --permanent --zone=public --add-service=ssh
# firewall-cmd --reload
$ firewall-cmd --zone=public --query-service=ssh
~~~

Persist a port:

~~~bash
# firewall-cmd --permanent --zone=public --add-port=8080/tcp
# firewall-cmd --reload
$ firewall-cmd --zone=public --query-port=8080/tcp
~~~

Remove:

~~~bash
# firewall-cmd --permanent --zone=public --remove-port=8080/tcp
# firewall-cmd --reload
~~~

Security habit:

- allow only the required service or port;
- apply it to the correct zone;
- verify permanent intent and runtime result;
- do not stop firewalld to make an application work.

Chapter 8 covers source restrictions and profile-to-zone assignment.

## 6. SELinux mental model

SELinux makes policy decisions using labels and rules in addition to ordinary Unix permissions.

A context appears as:

~~~text
SELinux-user:role:type:level
~~~

For routine targeted-policy administration, the **type** is usually the most important part:

- a web process can run as `httpd_t`;
- ordinary static web content can be `httpd_sys_content_t`;
- policy allows particular operations between defined types.

## 7. SELinux modes

Inspect:

~~~bash
$ getenforce
$ sestatus
~~~

Change current runtime mode:

~~~bash
# setenforce 0
# setenforce 1
~~~

- `0` means permissive;
- `1` means enforcing.

This does not change the next boot. Persistent setting is in `/etc/selinux/config`:

~~~ini
SELINUX=enforcing
SELINUXTYPE=targeted
~~~

or:

~~~ini
SELINUX=permissive
SELINUXTYPE=targeted
~~~

The exam objective requires enforcing and permissive management. Do not disable SELinux to solve an ordinary configuration problem.

Permissive mode logs denials without enforcing them. It is a diagnostic state, not a substitute for a correct policy-aligned configuration.

## 8. View file and process contexts

~~~bash
$ ls -lZ /var/www/html
$ stat -c '%C %n' /var/www/html/index.html
$ ps -eZ
$ ps -eZ | grep httpd
$ id -Z
~~~

Compare a suspect path with a known-good path serving the same purpose.

View the default context policy expects:

~~~bash
$ matchpathcon /web/index.html
~~~

## 9. Restore default file contexts

If a file has the wrong current label but the policy already defines the correct default:

~~~bash
# restorecon -v /var/www/html/index.html
# restorecon -Rv /var/www/html
~~~

`restorecon` reads the policy's file-context mappings.

`chcon` changes only the current label:

~~~bash
# chcon -t httpd_sys_content_t /web/index.html
~~~

That can be overwritten by relabeling or `restorecon`. It is useful for temporary diagnosis, not the normal persistent solution.

## 10. Define persistent custom file contexts

Suppose static web content is stored below `/web`.

Install the management command when needed:

~~~bash
# dnf install policycoreutils-python-utils
~~~

Add a persistent mapping:

~~~bash
# semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'
# restorecon -Rv /web
# ls -ldZ /web
# ls -lZ /web
~~~

Reasoning:

1. `semanage fcontext` records what labels the path should have.
2. `restorecon` applies those expected labels to existing files.
3. Future relabeling preserves the mapping.

Add a more specific mapping for a writable subdirectory:

~~~bash
# semanage fcontext -a -t httpd_sys_rw_content_t '/web/uploads(/.*)?'
# restorecon -Rv /web/uploads
~~~

Use `-m` only when that exact regular-expression mapping already exists and must be modified.

List local customizations:

~~~bash
# semanage fcontext -C -l
~~~

Remove a local mapping, then recalculate labels:

~~~bash
# semanage fcontext -d '/web(/.*)?'
# restorecon -Rv /web
~~~

## 11. Manage SELinux port labels

A daemon may be configured to listen on a nondefault port, but SELinux must allow its process type to bind a compatible port type.

Inspect:

~~~bash
# semanage port -l | grep '^http_port_t'
~~~

Add TCP 8080 to the web port type:

~~~bash
# semanage port -a -t http_port_t -p tcp 8080
~~~

If the port already has another SELinux definition, `-a` fails. Inspect first, then modify only when the task requires reassignment:

~~~bash
# semanage port -m -t http_port_t -p tcp 8080
~~~

Remove a local addition:

~~~bash
# semanage port -d -p tcp 8080
~~~

The complete network-service chain is:

1. application configured to listen on the port;
2. SELinux port type permits that daemon domain;
3. firewalld permits required clients;
4. service is active and listening.

Verify:

~~~bash
$ ss -lntp
$ firewall-cmd --zone=public --list-ports
$ semanage port -l | grep 8080
~~~

## 12. SELinux booleans

Booleans expose supported policy choices.

List and search:

~~~bash
$ getsebool -a
$ getsebool -a | grep httpd
$ semanage boolean -l | grep httpd
~~~

Change runtime only:

~~~bash
# setsebool httpd_can_network_connect on
~~~

Change runtime and persistent policy state:

~~~bash
# setsebool -P httpd_can_network_connect on
~~~

Verify:

~~~bash
$ getsebool httpd_can_network_connect
~~~

Enable only a boolean that matches the required behavior. Do not enable every web-related boolean.

## 13. Diagnose SELinux denials

First confirm ordinary configuration:

~~~bash
$ getenforce
$ ls -lZ AFFECTED_PATH
$ ps -eZ | grep PROCESS
# ausearch -m AVC,USER_AVC -ts recent
# journalctl -t setroubleshoot --since today
~~~

Then classify the cause:

- wrong file location or label -> use `semanage fcontext` and `restorecon`;
- nondefault service port -> use `semanage port`;
- supported optional behavior -> find the specific boolean;
- ordinary mode or ownership failure -> fix Unix permissions;
- application misconfiguration -> fix the application;
- genuine missing policy after correct labeling and configuration -> investigate policy as an advanced task.

Do not begin with `audit2allow`. Most RHCSA-level denials are caused by a wrong label, port type, boolean, or ordinary permission.

Temporary permissive testing can show whether SELinux is involved, but return to enforcing and implement the correct fix:

~~~bash
# setenforce 0
# reproduce-the-test
# setenforce 1
~~~

## 14. Integrated real-life example: web service on custom path and port

Requirement: Apache serves `/web` on TCP 8080 and starts at boot.

~~~bash
# dnf install httpd policycoreutils-python-utils
# mkdir -p /web
# printf '%s\n' 'RHEL 10 web service' > /web/index.html
# semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'
# restorecon -Rv /web
# semanage port -a -t http_port_t -p tcp 8080
# vim /etc/httpd/conf/httpd.conf
# apachectl configtest
# firewall-cmd --permanent --zone=public --add-port=8080/tcp
# firewall-cmd --reload
# systemctl enable --now httpd
~~~

The Apache configuration must include the requested document root and listen port with appropriate directory access. Verify each layer:

~~~bash
$ ls -lZ /web
$ semanage port -l | grep '^http_port_t'
$ firewall-cmd --zone=public --query-port=8080/tcp
$ systemctl is-enabled httpd
$ systemctl is-active httpd
$ ss -lntp | grep ':8080'
$ curl http://localhost:8080/
~~~

If `semanage port -a` reports that 8080 is already defined, inspect its existing type and use `-m` only if reassignment is correct.

## 15. Exam lab

Tasks:

1. Set an interactive default umask that creates files as 640 and directories as 750.
2. Configure SSH key authentication for the supplied account.
3. Make `/srv/site` valid static Apache content persistently.
4. Allow Apache to bind supplied TCP port 8088.
5. Permit that port through the active firewalld zone.
6. Ensure SELinux is enforcing now and at boot.
7. Enable a supplied SELinux boolean persistently.

Solution pattern:

~~~bash
$ umask 0027
$ ssh-keygen -t ed25519
$ ssh-copy-id ACCOUNT@HOST
# semanage fcontext -a -t httpd_sys_content_t '/srv/site(/.*)?'
# restorecon -Rv /srv/site
# semanage port -a -t http_port_t -p tcp 8088
# firewall-cmd --permanent --zone=ZONE --add-port=8088/tcp
# firewall-cmd --reload
# setenforce 1
# vim /etc/selinux/config
# setsebool -P SUPPLIED_BOOLEAN on
~~~

Verify:

~~~bash
$ umask
$ ssh ACCOUNT@HOST
$ ls -ldZ /srv/site
$ semanage port -l | grep 8088
$ firewall-cmd --zone=ZONE --query-port=8088/tcp
$ getenforce
$ grep '^SELINUX=' /etc/selinux/config
$ getsebool SUPPLIED_BOOLEAN
~~~

## Quick check

- Can you calculate file and directory results from a umask?
- Can you distinguish shell umask from systemd `UMask=`?
- Can you configure SSH keys without moving the private key?
- Can you verify key-file ownership, mode, and SELinux context?
- Can you explain runtime and persistent firewalld changes?
- Can you distinguish current SELinux mode from boot mode?
- Can you identify types on files and processes?
- Can you make custom path labels persistent?
- Can you distinguish an application port, firewall rule, and SELinux port type?
- Can you choose a boolean based on the required behavior?

## Official references

- [RHEL 10 documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10)
- Local pages: `man umask`, `man ssh-keygen`, `man ssh-copy-id`, `man sshd_config`, `man firewall-cmd`, `man selinux`, `man semanage-fcontext`, `man restorecon`, `man semanage-port`, and `man setsebool`
