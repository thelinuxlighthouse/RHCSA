# Chapter 2: Managing Software

## Objectives

After this chapter, you should be able to:

- explain the relationship among an RPM package, the RPM database, DNF, and a repository;
- identify which package owns a file and which package provides a missing command;
- configure and verify RPM repositories;
- install, update, reinstall, downgrade, and remove RPM packages;
- install a local RPM without bypassing dependency checks;
- inspect DNF transactions and verify installed files;
- configure a Flatpak remote; and
- search, install, update, inspect, and remove Flatpak applications.

The current RHEL 10 EX200 objectives explicitly include both RPM and Flatpak repositories and packages.

## 1. The software layers

An RPM file is a package containing files and metadata. The RPM database records installed packages. A repository stores packages plus searchable dependency metadata. DNF is the normal high-level tool that reads repository metadata, resolves dependencies, verifies signatures, and performs RPM transactions.

Use this rule:

- Use `dnf` for normal installation and removal, including local RPM files.
- Use `rpm` mainly to query, verify, or inspect packages.
- Do not routinely install with `rpm -i`; it does not obtain missing dependencies.

Package identity is often described as NEVRA:

~~~text
Name-Epoch:Version-Release.Architecture
~~~

The epoch is normally omitted when it is zero.

## 2. Query installed and available RPM content

### DNF discovery

~~~bash
# dnf repolist
# dnf repolist --all
# dnf search 'web server'
# dnf list installed
# dnf list available
# dnf info httpd
# dnf repoquery httpd
# dnf provides /usr/bin/semanage
~~~

Use `dnf search` when you know a concept, `dnf info` when you know a package, and `dnf provides` when you know a file or command path.

### RPM queries

~~~bash
$ rpm -q bash
$ rpm -qi bash
$ rpm -ql bash
$ rpm -qc openssh-server
$ rpm -qd openssh-server
$ rpm -qf /usr/bin/ssh
$ rpm -q --whatprovides /usr/bin/ssh
$ rpm -qa | sort
~~~

For an RPM file that is not installed:

~~~bash
$ rpm -qpi ./package.rpm
$ rpm -qpl ./package.rpm
$ rpm -K ./package.rpm
~~~

The `-p` means “query this package file,” not the installed database.

## 3. Configure RPM repositories

Repository files are normally stored in `/etc/yum.repos.d/` and end with `.repo`. A minimal repository section is:

~~~ini
[training-baseos]
name=Training BaseOS
baseurl=https://repo.example.com/rhel10/BaseOS/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
~~~

Meaning:

- the name in brackets is the unique repository ID;
- `name` is a human-readable label;
- `baseurl` points to the directory containing `repodata`;
- `enabled=1` makes the repository available;
- `gpgcheck=1` requires package signature verification;
- `gpgkey` identifies the trusted signing key.

Do not solve a signature problem by setting `gpgcheck=0` unless a lab explicitly requires an unsigned training repository. In real administration, confirm that the repository and key are trusted.

### Safely add and test a remote repository

~~~bash
# vim /etc/yum.repos.d/training.repo
# dnf clean metadata
# dnf repolist
# dnf --disablerepo='*' --enablerepo=training-baseos makecache
# dnf --disablerepo='*' --enablerepo=training-baseos list available
~~~

Testing with all other repositories disabled proves that the new repository works by itself.

### Enable or disable for one command

~~~bash
# dnf --enablerepo=training-baseos install package-name
# dnf --disablerepo=training-baseos update
~~~

To change the persistent `enabled` value, edit the `.repo` file. If the `config-manager` plugin is available, it can also manage repository state; use its local man page because DNF plugin syntax can differ by version.

### Local repository from installation media

First inspect the media:

~~~bash
# mkdir -p /mnt/rhel10
# mount -o loop,ro /root/rhel-10.iso /mnt/rhel10
# find /mnt/rhel10 -maxdepth 2 -type d -name repodata
~~~

Then configure the repositories that actually exist on that media:

~~~ini
[rhel10-media-baseos]
name=RHEL 10 Media BaseOS
baseurl=file:///mnt/rhel10/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel10-media-appstream]
name=RHEL 10 Media AppStream
baseurl=file:///mnt/rhel10/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
~~~

Verify:

~~~bash
# dnf clean all
# dnf repolist
# dnf --disablerepo='*' --enablerepo='rhel10-media-*' makecache
~~~

The mount must exist whenever DNF uses the file-based repository. Chapter 6 explains persistent mounts.

### Red Hat Content Delivery Network

A subscribed RHEL system normally uses `subscription-manager`:

~~~bash
# subscription-manager status
# subscription-manager identity
# subscription-manager repos --list-enabled
# subscription-manager repos --list
~~~

Registration requires credentials or an organization and activation key supplied by the organization:

~~~bash
# subscription-manager register
~~~

Repository IDs vary with architecture and subscription. List them instead of guessing:

~~~bash
# subscription-manager repos --list | less
# subscription-manager repos --enable=REPOSITORY_ID
~~~

Never place real credentials or activation keys in a study script or shell history.

## 4. Install, update, and remove RPM software

### Install

~~~bash
# dnf install httpd
# dnf install -y httpd curl
# dnf install /usr/bin/semanage
# dnf group list
# dnf group info 'Development Tools'
# dnf group install 'Development Tools'
~~~

Installing a command path is useful when the package name is unknown and DNF can resolve the provider.

### Install a local RPM correctly

~~~bash
# dnf install ./vendor-agent.rpm
~~~

The `./` makes it clear that this is a file. DNF can obtain dependencies from enabled repositories.

### Update

~~~bash
# dnf check-update
# dnf updateinfo
# dnf upgrade
# dnf upgrade httpd
# dnf upgrade --security
~~~

`dnf check-update` can return exit status 100 when updates are available; that is not the same as a fatal command failure.

### Remove, reinstall, and downgrade

~~~bash
# dnf remove httpd
# dnf autoremove
# dnf reinstall openssh-server
# dnf list --showduplicates package-name
# dnf downgrade package-name
~~~

Read the transaction summary before answering yes. Removing a package may remove dependent packages as well.

### Package groups and RHEL 10 modularity note

Package groups remain useful. RHEL 10 deprecates DNF modularity, and the current EX200 objectives do not require module-stream management. Do not build your exam workflow around old RHEL 8 module commands.

## 5. DNF history, cache, and repair

~~~bash
# dnf history
# dnf history info TRANSACTION_ID
# dnf history undo TRANSACTION_ID
# dnf history rollback TRANSACTION_ID
# dnf clean metadata
# dnf clean all
# dnf makecache
~~~

An undo or rollback is another package transaction; it is not a filesystem snapshot. It can fail if the required old packages are no longer available.

### Verify installed package files

~~~bash
$ rpm -V openssh-server
$ rpm -Va
~~~

No output means the checked metadata matches the RPM database. Output does not automatically mean compromise: administrators legitimately modify configuration files.

Common verification letters include:

| Marker | Changed attribute |
| --- | --- |
| `S` | size |
| `M` | mode or file type |
| `5` | digest |
| `U` | user owner |
| `G` | group owner |
| `T` | modification time |

If a packaged binary was damaged, reinstall its package:

~~~bash
# rpm -qf /usr/bin/ssh
# dnf reinstall openssh-clients
~~~

## 6. Troubleshoot repositories and packages

### “Cannot download metadata”

Check in this order:

~~~bash
# dnf repolist -v
# getent hosts repo.example.com
# curl -I https://repo.example.com/rhel10/BaseOS/repodata/repomd.xml
# grep -R '^\(baseurl\|metalink\|mirrorlist\|enabled\|gpgcheck\)' /etc/yum.repos.d/
# dnf clean metadata
# dnf makecache
~~~

Possible causes include a wrong URL, missing `repodata`, DNS failure, proxy failure, a certificate problem, or an inaccessible local mount.

### GPG failure

Do not disable checking as the first response. Confirm:

~~~bash
# rpm -qa 'gpg-pubkey*'
# rpm -K ./package.rpm
# grep -R '^gpgkey' /etc/yum.repos.d/
~~~

Import only a verified key:

~~~bash
# rpm --import /path/to/verified-key
~~~

### Locked transaction

First determine whether another package manager is genuinely running:

~~~bash
# ps -ef | grep '[d]nf'
# ps -ef | grep '[r]pm'
~~~

Do not delete lock files blindly while a real transaction is active.

## 7. Flatpak concepts

Flatpak is separate from RPM:

- a Flatpak **remote** is a repository;
- a Flatpak **application ID** identifies an application;
- a **runtime** supplies shared libraries to applications;
- installations can be system-wide or per-user;
- applications run with sandbox permissions.

Check availability:

~~~bash
# dnf install flatpak
$ flatpak --version
~~~

System operations are the default when run as root. A regular user can explicitly use `--user`.

## 8. Configure and use Flatpak repositories

### List and add remotes

~~~bash
$ flatpak remotes --show-details
# flatpak remote-add --if-not-exists rhel https://flatpaks.redhat.io/rhel.flatpakrepo
# flatpak remotes --system --show-details
~~~

On a subscribed RHEL system, the `redhat-flatpak-repo` package may configure access. On an unsubscribed system, access to Red Hat catalog content can require Red Hat credentials. Follow the environment supplied in the exam or lab.

To add a per-user remote:

~~~bash
$ flatpak remote-add --user --if-not-exists REMOTE_NAME REMOTE_URL
~~~

To remove one:

~~~bash
$ flatpak remote-delete --user REMOTE_NAME
# flatpak remote-delete --system REMOTE_NAME
~~~

### Search, install, and run

~~~bash
$ flatpak search APPLICATION_NAME
# flatpak install rhel APPLICATION_ID
$ flatpak list
$ flatpak info APPLICATION_ID
$ flatpak run APPLICATION_ID
~~~

Use the exact application ID returned by search. Names are not always unique.

### Update and remove

~~~bash
# flatpak update
# flatpak uninstall APPLICATION_ID
# flatpak uninstall --unused
~~~

For a per-user application:

~~~bash
$ flatpak install --user REMOTE_NAME APPLICATION_ID
$ flatpak uninstall --user APPLICATION_ID
~~~

Do not mix system and user scope while troubleshooting. Check both:

~~~bash
$ flatpak list --user
$ flatpak list --system
~~~

## 9. Real-life scenario: build a controlled utility host

Requirement:

- the `httpd` RPM must come only from `training-appstream`;
- the repository must stay enabled;
- a graphical Flatpak application must be installed system-wide from the `rhel` remote.

Workflow:

~~~bash
# dnf --disablerepo='*' --enablerepo=training-appstream info httpd
# dnf --disablerepo='*' --enablerepo=training-appstream install httpd
# rpm -q httpd
# dnf repoquery --installed httpd
# flatpak remote-add --if-not-exists rhel https://flatpaks.redhat.io/rhel.flatpakrepo
# flatpak search APPLICATION_NAME
# flatpak install rhel APPLICATION_ID
# flatpak info APPLICATION_ID
~~~

This is natural because discovery comes before change, source restriction proves where the RPM comes from, and separate verification is used for the separate packaging systems.

## 10. Exam lab

Tasks:

1. Create an enabled repository named `exam-baseos` that uses the supplied URL and performs GPG checking with the supplied key.
2. Prove that metadata can be read from only that repository.
3. Install the package requested by the examiner.
4. Find the RPM package that supplies a requested command.
5. Configure the supplied Flatpak remote and install the requested application system-wide.

Generic solution pattern:

~~~ini
[exam-baseos]
name=Exam BaseOS
baseurl=SUPPLIED_URL
enabled=1
gpgcheck=1
gpgkey=file:///SUPPLIED_KEY_PATH
~~~

~~~bash
# dnf clean metadata
# dnf --disablerepo='*' --enablerepo=exam-baseos makecache
# dnf --disablerepo='*' --enablerepo=exam-baseos install REQUESTED_PACKAGE
# dnf provides /usr/bin/REQUESTED_COMMAND
# flatpak remote-add --if-not-exists EXAM_REMOTE SUPPLIED_FLATPAK_URL
# flatpak search REQUESTED_APPLICATION
# flatpak install EXAM_REMOTE APPLICATION_ID
# rpm -q REQUESTED_PACKAGE
# flatpak info APPLICATION_ID
~~~

## Quick check

- Can you explain why DNF is safer than `rpm -i` for installation?
- Can you query an installed package, a package file, and the owner of an installed file?
- Can you write a valid `.repo` file without disabling GPG checks?
- Can you prove that a particular repository works alone?
- Can you distinguish system-wide from per-user Flatpak state?
- Can you install and remove a Flatpak by exact application ID?

## Official references

- [Managing software with DNF in RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/index)
- [RHEL 10 DNF command list](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/dnf-commands-list)
- [Installing applications with Flatpak](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/administering_rhel_by_using_the_gnome_desktop_environment/installing-applications-by-using-flatpak)

