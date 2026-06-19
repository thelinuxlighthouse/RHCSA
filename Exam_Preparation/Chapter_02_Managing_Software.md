# Chapter 2: Managing Software

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand RPM package management architecture in RHEL 10
- Configure access to RPM repositories (local, remote, and CDN)
- Install, update, remove, and query RPM software packages using `dnf`
- Manage package groups and modules
- Troubleshoot dependency issues
- Configure access to Flatpak repositories
- Install, update, remove, and manage Flatpak software packages
- Set Flatpak permissions and runtime environments

---

## Concepts and Theory

### Package Management Overview

Package management is the process of installing, upgrading, configuring, and removing software on a Linux system in a standardized, reproducible way. Without package management, administrators would need to manually download source code, compile it, resolve dependencies, and track file installations—a tedious and error-prone process.

RHEL 10 uses a layered package management architecture:

1. **RPM (Red Hat Package Manager)**: The low-level package format and database. RPM tracks every file installed, their permissions, ownership, and cryptographic signatures. It handles individual `.rpm` files but does not resolve dependencies automatically.

2. **DNF (Dandified YUM)**: The high-level package manager built on top of RPM. DNF resolves dependencies, manages repositories, handles transactions with rollback support, and provides a rich command-line interface. DNF 5 is the default in RHEL 10, written in C++ for improved performance.

3. **Flatpak**: A universal package format for end-user applications that runs in sandboxed environments. Flatpak packages include their own dependencies and runtimes, making them distribution-agnostic.

### RPM Package Structure

An RPM package (`.rpm` file) is a CPIO archive containing:

- **Payload**: The actual files to be installed, compressed with xz by default in RHEL 10
- **Metadata**: Package name, version, release, description, dependencies, scripts
- **Signature**: GPG signature for integrity and authenticity verification

Each RPM package is identified by a **NEVRA** string:

- **N**ame: Package name (e.g., `nginx`)
- **E**poch: Optional version qualifier for complex versioning (rarely used)
- **V**ersion: Software version (e.g., `1.24.0`)
- **R**elease: Package release number (e.g., `1.el10`)
- **A**rchitecture: Target architecture (`x86_64`, `aarch64`, `noarch`)

Example NEVRA: `nginx-1:1.24.0-1.el10.x86_64`

### RPM Database

RPM maintains a database at `/var/lib/rpm/` that tracks every installed package. This database stores:

- Package name, version, release, and architecture
- File list with checksums for each installed file
- File permissions, ownership, and SELinux contexts
- Pre-installation and post-installation script results
- Configuration file modification status

The RPM database enables verification of installed packages, file ownership lookups, and dependency resolution.

### DNF and Repositories

DNF retrieves packages from configured repositories. A repository is a collection of RPM packages along with metadata describing available packages, their versions, dependencies, and relationships.

**Repository Types:**

- **BaseOS**: Core operating system packages, kernel, essential utilities
- **AppStream**: Application stream packages, development tools, servers
- **Custom/Third-party**: Vendor-specific or community repositories
- **Local**: Packages stored on the local filesystem
- **CDN**: Red Hat Content Delivery Network (subscription-based)

**Repository Metadata:**

Each repository contains a `repodata/` directory with XML files describing available packages. DNF downloads this metadata before performing any package operation, caching it locally for efficiency.

### DNF 5 in RHEL 10

RHEL 10 ships with DNF 5, a complete rewrite of the package manager with significant improvements:

- **Performance**: Written in C++ with a libdnf backend, dramatically faster dependency resolution
- **LibrePM Integration**: Uses LibrePM for low-level RPM operations
- **Improved Caching**: Better metadata and package caching
- **Plugin Architecture**: Extensible through Python plugins
- **Backward Compatibility**: Most DNF 4 commands work identically

### Flatpak Architecture

Flatpak is a universal package format designed for desktop applications. Key concepts:

**Sandboxing**: Each Flatpak application runs in an isolated sandbox with limited access to the system. Applications must explicitly request permissions for network access, device access, and filesystem access.

**Runtimes**: Flatpak applications depend on runtimes (formerly called SDKs) that provide shared libraries and frameworks. Common runtimes include Freedesktop and GNOME runtimes.

**Remotes**: Flatpak remotes are repositories that host Flatpak applications and runtimes. The default remote in RHEL 10 is **Flathub**, the community-maintained repository.

**Directory Structure**:
- `/var/lib/flatpak/`: System-wide Flatpak installations
- `~/.local/share/flatpak/`: User-level Flatpak installations
- `/var/lib/flatpak/repo/`: Downloaded Flatpak packages

### Dependency Management

Dependencies are relationships between packages. When you install a package, DNF automatically resolves and installs all required dependencies. Types of dependencies include:

- **Requires**: Packages that must be installed for this package to function
- **Recommends**: Suggested packages that enhance functionality
- **Conflicts**: Packages that cannot coexist with this package
- **Obsoletes**: Older packages that this package replaces
- **Provides**: Virtual capabilities this package offers

DNF uses a sophisticated solver algorithm to find a consistent set of package transactions that satisfy all dependency constraints.

---

## How It Works Internally

### RPM Installation Process

When you install an RPM package, the following steps occur:

1. **GPG Verification**: RPM verifies the package signature against trusted GPG keys stored in `/etc/pki/rpm-gpg/`
2. **Dependency Check**: RPM checks if all required dependencies are satisfied
3. **Script Execution**: Pre-installation scripts (`prein`) run before file installation
4. **File Installation**: Package files are extracted to their target locations
5. **Database Update**: RPM database is updated with package metadata
6. **Script Execution**: Post-installation scripts (`postin`) run after file installation
7. **Transaction Recording**: The transaction is logged for potential rollback

### DNF Transaction Flow

When you run `dnf install package`, DNF performs these steps:

1. **Metadata Refresh**: DNF checks if repository metadata is stale and refreshes if needed
2. **Solve**: The dependency solver calculates the complete transaction, including all required and recommended packages
3. **Download**: Packages are downloaded from repository mirrors, using the fastest available mirror
4. **GPG Check**: Each package is verified against repository GPG keys
5. **Transaction Test**: DNF performs a test transaction to verify feasibility
6. **User Confirmation**: DNF displays the transaction summary and asks for confirmation
7. **Execution**: RPM installs packages in dependency order
8. **Cleanup**: Downloaded packages may be cached or removed based on configuration

### Repository Mirror Selection

DNF uses the `metalink` or `mirrorlist` URL in repository configuration to discover available mirrors. It selects mirrors based on:

- Geographic proximity
- Historical response time
- Current load status
- Protocol support (HTTPS preferred)

The `dnf mirrorbar` plugin provides visual feedback during mirror selection and package downloads.

### How Flatpak Sandboxing Works

Flatpak uses Linux namespaces and seccomp filters to create sandboxed environments:

1. **Filesystem Isolation**: Applications only see a virtualized filesystem. Access to host files is granted through explicit portal mechanisms (file chooser dialogs)
2. **Network Control**: Network access can be enabled or disabled per application
3. **Device Access**: Access to cameras, printers, and other devices is controlled through portals
4. **Bubblewrap**: The underlying sandboxing technology is `bwrap` (Bubblewrap), a lightweight sandboxing tool
5. **Portals**: Standardized interfaces for applications to request host system access (file dialogs, printing, screenshots)

### DNF Module System

RHEL 10 uses the AppStream module system to manage multiple versions of software. A module defines:

- **Module Name**: Logical grouping (e.g., `nginx`, `python39`, `nodejs`)
- **Stream**: Version stream (e.g., `1.24`, `1.26`)
- **Profile**: Subset of packages within a stream (e.g., `default`, `common`, `minimal`)
- **Context**: Build context for distinguishing variants

Modules enable installing different versions of the same software without conflicts.

---

## Commands and Administration Tasks

### Configuring RPM Repositories

#### Repository Configuration Files

Repository configurations are stored in `/etc/yum.repos.d/` as `.repo` files.

```bash
# List configured repositories
dnf repolist all

# View repository configuration
cat /etc/yum.repos.d/*.repo

# Check repository metadata status
dnf repolist -v
```

#### Creating a Repository Configuration File

File: `/etc/yum.repos.d/custom.repo`

```ini
[custom-repo]
name=Custom Repository
baseurl=https://example.com/repo/rhel-$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://example.com/RPM-GPG-KEY-custom
```

**Repository Configuration Options:**

| Option | Description | Example |
|--------|-------------|---------|
| `name` | Human-readable repository name | `name=My Repo` |
| `baseurl` | Repository URL | `baseurl=https://mirror.example.com/repo/` |
| `mirrorlist` | URL to mirror list | `mirrorlist=https://mirrors.example.com/list` |
| `metalink` | URL to metalink file | `metalink=https://mirrors.example.com/metalink` |
| `enabled` | Enable (1) or disable (0) | `enabled=1` |
| `gpgcheck` | Enable (1) or disable (0) GPG checking | `gpgcheck=1` |
| `gpgkey` | URL or path to GPG key | `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY` |
| `repo_gpgcheck` | Check individual package signatures | `repo_gpgcheck=1` |
| `sslverify` | Verify SSL certificates | `sslverify=1` |
| `exclude` | Packages to exclude | `exclude=kernel*` |
| `includepkgs` | Only include specific packages | `includepkgs=nginx,nginx-mod-http-ssl` |

#### Configuring a Local Repository

```bash
# Mount ISO or copy packages to local directory
mkdir -p /mnt/repo
mount /dev/sr0 /mnt/repo

# Create local repository configuration
cat > /etc/yum.repos.d/local.repo << 'EOF'
[local-repo]
name=Local Repository
baseurl=file:///mnt/repo
enabled=1
gpgcheck=0
EOF

# Refresh repository metadata
dnf makecache
```

#### Configuring a Remote HTTP Repository

```bash
# Create remote repository configuration
cat > /etc/yum.repos.d/remote.repo << 'EOF'
[remote-repo]
name=Remote Repository
baseurl=http://repo.example.com/packages/rhel10/x86_64/
enabled=1
gpgcheck=1
gpgkey=http://repo.example.com/RPM-GPG-KEY
EOF

# Import GPG key manually if needed
rpm --import http://repo.example.com/RPM-GPG-KEY

# Test repository access
dnf repolist
```

#### Managing Repository GPG Keys

```bash
# List imported GPG keys
rpm -qa gpg-pubkey*

# Import a GPG key
rpm --import /path/to/RPM-GPG-KEY

# Import GPG key from URL
rpm --import https://example.com/RPM-GPG-KEY

# View GPG key details
rpm -qi gpg-pubkey-KEYID-DATE

# Remove a GPG key
rpm -e gpg-pubkey-KEYID-DATE
```

### Installing and Managing RPM Packages with DNF

#### Installing Packages

```bash
# Install a single package
dnf install nginx

# Install multiple packages
dnf install nginx php-fpm mariadb-server

# Install without prompting
dnf install -y nginx

# Install specific version
dnf install nginx-1.24.0-1.el10

# Install from a specific repository
dnf install --enablerepo=custom-repo package-name

# Install from local RPM file
dnf install /path/to/package.rpm

# Install from URL
dnf install https://example.com/packages/package.rpm
```

#### Querying Package Information

```bash
# Search for packages by name or description
dnf search keyword

# List installed packages
dnf list installed

# List available packages
dnf list available

# List all packages (installed and available)
dnf list all

# Get detailed package information
dnf info nginx

# List files owned by a package
dnf repoquery -l nginx

# Find which package owns a file
dnf provides /usr/bin/nginx
dnf provides */nginx.conf

# List package dependencies
dnf repoquery --requires nginx

# List packages that depend on a package
dnf repoquery --whatrequires nginx
```

#### Updating Packages

```bash
# Update all packages
dnf update

# Update specific package
dnf update nginx

# Update and show obsoletes
dnf update --best

# Check for available updates without installing
dnf check-update

# List available updates
dnf list updates

# Security updates only
dnf update --security

# Minimal update (no architecture changes)
dnf update --minimal
```

#### Removing Packages

```bash
# Remove a package
dnf remove nginx

# Remove package and unused dependencies
dnf autoremove nginx

# Remove multiple packages
dnf remove nginx php-fpm

# Remove without prompting
dnf remove -y nginx
```

#### Package History and Rollback

```bash
# View transaction history
dnf history

# View details of specific transaction
dnf history info 42

# List packages in a transaction
dnf history list 42

# Undo a transaction
dnf history undo 42

# Redo a transaction
dnf history redo 42
```

#### DNF Cache Management

```bash
# Clear all DNF cache
dnf clean all

# Clear package cache only
dnf clean packages

# Clear metadata cache
dnf clean metadata

# Refresh repository metadata
dnf makecache

# View cache location and size
dnf info dnf
du -sh /var/cache/dnf/
```

### RPM Direct Commands

While DNF is preferred, RPM direct commands are useful for querying and verification:

```bash
# Query installed packages
rpm -qa                          # List all installed packages
rpm -qa | grep nginx             # Search installed packages

# Query package information
rpm -qi nginx                    # Package info
rpm -ql nginx                    # List files
rpm -qc nginx                    # List config files
rpm -qd nginx                    # List doc files

# Query file ownership
rpm -qf /usr/bin/ls              # Which package owns this file
rpm -qf /etc/hosts               # File ownership lookup

# Verify package integrity
rpm -V nginx                     # Verify specific package
rpm -Va                          # Verify all packages

# Verify options
rpm -V --nomtime nginx           # Ignore modification time
rpm -V --nosize nginx            # Ignore file size

# RPM verification flags:
# S - file Size differs
# M - Mode differs (permissions)
# 5 - MD5 sum differs
# D - Device major/minor number mismatch
# L - readLink(2) path mismatch
# U - User ownership differs
# G - Group ownership differs
# T - mTime differs
# P - caPabilities differ
```

### Managing Package Groups

```bash
# List available groups
dnf group list

# List available groups (including hidden)
dnf group list --hidden

# Install a group
dnf group install "Server with GUI"

# Install a group without prompting
dnf group install -y "Minimal Install"

# Update a group
dnf group update "Development Tools"

# Remove a group
dnf group remove "Legacy Software"

# View group details
dnf group info "Development Tools"
```

### Managing DNF Modules

```bash
# List available modules
dnf module list

# List module details
dnf module info nginx

# Install a module profile
dnf module install nginx:1.24/common

# Enable a module stream
dnf module enable nginx:1.26

# Disable a module stream
dnf module disable nginx:1.24

# Reset module state
dnf module reset nginx

# Remove module
dnf module remove nginx
```

### Configuring Flatpak Repositories

#### Adding Flatpak Remotes

```bash
# List configured Flatpak remotes
flatpak remotes

# Add Flathub remote (most common)
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Add a custom remote
flatpak remote-add myremote https://example.com/repo.flatpakrepo

# Remove a remote
flatpak remote-delete myremote

# Modify remote options
flatpak remote-modify flathub --no-automatic-import
```

#### Flatpak Remote Options

```bash
# Set remote to auto-update
flatpak remote-modify flathub --autorefresh

# Disable auto-update
flatpak remote-modify flathub --no-autorefresh

# Set priority (lower number = higher priority)
flatpak remote-modify flathub --priority=5
```

### Installing and Managing Flatpak Packages

#### Installing Flatpak Applications

```bash
# Search for applications
flatpak search vscode

# Install application from Flathub
flatpak install flathub com.github.evernote.yubynote

# Install without confirmation
flatpak install -y flathub com.github.evernote.yubynote

# Install from file
flatpak install /path/to/application.flatpak

# Install specific version
flatpak install flathub com.github.evernote.yubynote --version=2.0

# Install for current user only (no root needed)
flatpak install --user flathub com.github.evernote.yubynote
```

#### Querying Flatpak Applications

```bash
# List installed Flatpak applications
flatpak list

# List installed applications with details
flatpak list --columns=name,version,application-id

# Show application information
flatpak info com.github.evernote.yubynote

# Show all available versions
flatpak info --show-permissions com.github.evernote.yubynote

# List runtimes
flatpak list --runtime
```

#### Updating Flatpak Applications

```bash
# Update all Flatpak applications
flatpak update

# Update specific application
flatpak update com.github.evernote.yubynote

# Update and show details
flatpak update -y

# Refresh remote metadata only
flatpak update --appstream
```

#### Removing Flatpak Applications

```bash
# Remove application
flatpak uninstall com.github.evernote.yubynote

# Remove without confirmation
flatpak uninstall -y com.github.evernote.yubynote

# Remove unused runtimes
flatpak uninstall --unused

# Remove all user installations
flatpak uninstall --user --delete-cache
```

#### Managing Flatpak Permissions

```bash
# Override permissions for an application
flatpak override com.github.evernote.yubynote --device=all
flatpak override com.github.evernote.yubynote --filesystem=home
flatpak override com.github.evernote.yubynote --network
flatpak override com.github.evernote.yubynote --env=GTK_THEME=Adwaita

# Reset overrides
flatpak override --reset com.github.evernote.yubynote

# View overrides
flatpak override com.github.evernote.yubynote

# Common permission options:
# --filesystem=PATH    - Grant filesystem access
# --device=all         - Grant all device access
# --socket=x11         - Grant X11 access
# --socket=wayland     - Grant Wayland access
# --env=KEY=VALUE      - Set environment variable
```

### DNF Configuration

#### Main DNF Configuration

File: `/etc/dnf/dnf.conf`

```ini
[main]
# Number of packages to keep in cache (0 = unlimited)
keepcache=0

# Log file location
logdir=/var/log/dnf

# Maximum number of history records
max_parallel_downloads=10

# GPG check enabled
gpgcheck=1

# Install strong dependencies automatically
installonly_limit=3

# Exclude architectures
# excludepkgs=kernel*

# Automatically remove unused dependencies
clean_requirements_on_remove=True

# Best effort upgrade
best=True
```

#### DNF Plugin Configuration

Plugin configurations are located in `/etc/dnf/plugins/`:

```bash
# List installed DNF plugins
dnf version

# Check plugin status
dnf install dnf-plugin-conflicts
```

---

## Configuration Examples

### RHEL 10 Base Repository Configuration

File: `/etc/yum.repos.d/rhel.repo`

```ini
# RHEL 10 Base Repository Configuration

[rhel-baseos]
name=Red Hat Enterprise Linux $releasever - BaseOS
baseurl=https://cdn.redhat.com/content/dist/rhel10/$releasever/$basearch/baseos/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel-appstream]
name=Red Hat Enterprise Linux $releasever - AppStream
baseurl=https://cdn.redhat.com/content/dist/rhel10/$releasever/$basearch/appstream/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

### Custom Internal Repository Configuration

File: `/etc/yum.repos.d/internal.repo`

```ini
# Internal mirror repository

[internal-baseos]
name=Internal BaseOS Mirror
baseurl=https://mirror.internal.company.com/rhel10/BaseOS/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
metadata_expire=60
sslverify=1
sslcacert=/etc/pki/tls/certs/internal-ca.crt

[internal-appstream]
name=Internal AppStream Mirror
baseurl=https://mirror.internal.company.com/rhel10/AppStream/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
metadata_expire=60
sslverify=1
sslcacert=/etc/pki/tls/certs/internal-ca.crt

[internal-extras]
name=Internal Extras Repository
baseurl=https://mirror.internal.company.com/extras/rhel10/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://mirror.internal.company.com/RPM-GPG-KEY-extras
includepkgs=custom-tool1,custom-tool2,custom-tool3
```

### NFS-Mounted Local Repository

File: `/etc/yum.repos.d/nfs-repo.repo`

```ini
[nfs-repo]
name=NFS Mounted Repository
baseurl=file:///mnt/nfs/repo/rhel10
enabled=1
gpgcheck=1
gpgkey=file:///mnt/nfs/repo/RPM-GPG-KEY
```

Mount configuration in `/etc/fstab`:

```
# NFS repository mount
server.internal:/exports/repo  /mnt/nfs/repo  nfs  defaults,_netdev  0  0
```

### DNF Configuration with Plugins

File: `/etc/dnf/dnf.conf`

```ini
[main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=True
max_parallel_downloads=10
keepcache=0
cachedir=/var/cache/dnf
logdir=/var/log/dnf
history_record=True

# Color output
color_list_available_upgrade=underline/magenta/bold
color_list_installed_extra=underline/blue/bold

# Default install language
default_install_language=en_US
```

### Flatpak System-Wide Configuration

File: `/etc/flatpak/remotes.d/flathub.conf`

```ini
[remote "flathub"]
title=Flathub
url=https://flathub.org/repo
gpg-verify=1
autorefresh=true
no-shell-completion=false
```

---

## Real-Web Examples

### Scenario 1: Setting Up a Development Environment

A developer needs to set up a complete web development environment on a fresh RHEL 10 installation.

```bash
# Step 1: Update system packages
sudo dnf update -y

# Step 2: Install development tools group
sudo dnf group install -y "Development Tools"

# Step 3: Install web server and database
sudo dnf install -y nginx php-fpm php-mysqlnd mariadb-server

# Step 4: Install version control and utilities
sudo dnf install -y git wget curl vim

# Step 5: Install Node.js from module stream
sudo dnf module install -y nodejs:18/common

# Step 6: Install Docker for containerization
sudo dnf install -y docker-ce docker-ce-cli containerd.io

# Step 7: Verify installations
nginx -v
php -v
node --version
git --version
docker --version

# Step 8: Enable services for auto-start
sudo systemctl enable --now nginx php-fpm mariadb docker
```

### Scenario 2: Migrating Repository Configuration

An organization is migrating from CDN to an internal mirror for better performance and air-gapped support.

```bash
# Step 1: Backup existing repository configurations
sudo cp -a /etc/yum.repos.d /etc/yum.repos.d.backup

# Step 2: Disable CDN repositories
for repo in /etc/yum.repos.d/rhel-*.repo; do
    sudo sed -i 's/^enabled=1/enabled=0/' "$repo"
done

# Step 3: Create internal mirror configuration
sudo tee /etc/yum.repos.d/internal-mirror.repo > /dev/null << 'EOF'
[internal-baseos]
name=Internal BaseOS
baseurl=https://mirror.internal.corp/rhel10/BaseOS/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[internal-appstream]
name=Internal AppStream
baseurl=https://mirror.internal.corp/rhel10/AppStream/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

# Step 4: Clear cache and rebuild
sudo dnf clean all
sudo dnf makecache

# Step 5: Verify new repositories
sudo dnf repolist

# Step 6: Test package installation
sudo dnf install -y tree
```

### Scenario 3: Emergency Package Recovery

A critical system package was accidentally removed, breaking multiple services.

```bash
# Step 1: Identify the broken package
rpm -Va | head -20

# Step 2: Check DNF transaction history
sudo dnf history list

# Step 3: View the transaction that removed the package
sudo dnf history info 42

# Step 4: Undo the problematic transaction
sudo dnf history undo 42

# Step 5: If undo is not possible, reinstall the package
sudo dnf reinstall package-name

# Step 6: Reinstall all files from a package (including configs)
sudo dnf reinstall --replacefiles package-name

# Step 7: Verify package integrity
rpm -V package-name

# Step 8: Restart affected services
sudo systemctl restart affected-service
```

### Scenario 4: Deploying Flatpak Applications Organization-Wide

An IT department needs to deploy standardized applications across workstations using Flatpak.

```bash
# Step 1: Ensure Flathub remote is configured (system-wide)
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Step 2: Install required applications system-wide
sudo flatpak install -y flathub org.libreoffice.LibreOffice
sudo flatpak install -y flathub org.gimp.GIMP
sudo flatpak install -y flathub org.videolan.VLC
sudo flatpak install -y flathub com.github.johnfactotum.Foliate

# Step 3: Configure restrictive default permissions
sudo flatpak override --no-network
sudo flatpak override --filesystem=home:ro

# Step 4: Grant network access only to specific applications
sudo flatpak override org.videolan.VLC --network
sudo flatpak override org.libreoffice.LibreOffice --network

# Step 5: Set application-specific filesystem access
sudo flatpak override org.gimp.GIMP --filesystem=home
sudo flatpak override com.github.johnfactotum.Foliate --filesystem=~/Documents/Books

# Step 6: Verify installations
flatpak list

# Step 7: Schedule automatic updates via systemd timer
# (Configure flatpak autorefresh for system-wide updates)
sudo flatpak remote-modify flathub --autorefresh
```

---

## Troubleshooting

### Repository Connection Issues

**Problem**: DNF cannot connect to repository

```bash
# Check repository configuration syntax
sudo dnf repolist -v

# Test network connectivity to repository
curl -I https://repo.example.com/path/

# Check DNS resolution
nslookup mirror.example.com

# Check if firewall blocks repository access
sudo firewall-cmd --list-all

# Test with verbose output
sudo dnf update -d 10

# Check repository metadata expiration
sudo dnf clean metadata
sudo dnf makecache
```

**Problem**: GPG key import fails

```bash
# Check if GPG key is imported
rpm -qa gpg-pubkey*

# Import GPG key manually
sudo rpm --import https://repo.example.com/RPM-GPG-KEY

# Verify GPG key
rpm -qi gpg-pubkey-KEYID

# If GPG check is causing issues temporarily (exam only)
# Disable gpgcheck in repo file: gpgcheck=0
# Or skip for single command: dnf install --nogpgcheck package
```

### Dependency Resolution Failures

**Problem**: DNF cannot resolve dependencies

```bash
# Check for broken dependencies
sudo dnf check

# Update all packages first to resolve version conflicts
sudo dnf update -y

# Try with best effort
sudo dnf install --best --allowerasing package

# Check for module conflicts
sudo dnf module list

# Reset module state and retry
sudo dnf module reset module-name
sudo dnf module install module-name:stream/profile

# Check for conflicting packages
sudo dnf repoquery --whatconflicts package
```

**Problem**: Package not found

```bash
# Verify repository is enabled
sudo dnf repolist

# Search across all repositories
sudo dnf search package-name

# Check if package exists in any repository
sudo dnf --enablerepo='*' list package-name

# Check correct package name
sudo dnf provides */binary-name

# Check if repository metadata is current
sudo dnf makecache
```

### RPM Database Issues

**Problem**: RPM database is corrupted

```bash
# Rebuild RPM database
sudo rpm --rebuilddb

# Verify all packages
sudo rpm -Va

# Check specific package
sudo rpm -V package-name

# Reinstall package to fix missing files
sudo dnf reinstall package-name

# Verify RPM database files exist
ls -la /var/lib/rpm/
```

### DNF Transaction Failures

**Problem**: DNF transaction locked

```bash
# Check for running DNF processes
ps aux | grep dnf

# Remove lock file (only if no DNF process is running)
sudo rm -f /var/run/dnf.pid

# Check for stuck yum processes
sudo rm -f /var/run/yum.pid

# Verify no packagekit is running
sudo systemctl stop packagekit
```

**Problem**: Insufficient disk space

```bash
# Check available disk space
df -h /var/cache/dnf
df -h /

# Clear DNF cache
sudo dnf clean all

# Remove old kernel packages
sudo dnf remove $(dnf repoquery --installonly --latest-extra)

# Reduce cache retention
# Edit /etc/dnf/dnf.conf: keepcache=0
```

### Flatpak Issues

**Problem**: Flatpak application fails to launch

```bash
# Check Flatpak installation status
flatpak list

# Reinstall the application
flatpak uninstall com.example.App
flatpak install flathub com.example.App

# Check for permission issues
flatpak info --show-permissions com.example.App

# Override permissions
flatpak override com.example.App --filesystem=home

# Check Flatpak runtime is installed
flatpak list --runtime

# Reinstall runtime
flatpak install flathub org.freedesktop.Platform//24.08
```

**Problem**: Flatpak remote not accessible

```bash
# Check remote configuration
flatpak remotes

# Remove and re-add remote
sudo flatpak remote-delete flathub
sudo flatpak remote-add flathub https://flathub.org/repo/flathub.flatpakrepo

# Test remote connectivity
flatpak update --appstream

# Check disk space for Flatpak storage
df -h /var/lib/flatpak
```

### Package Conflict Resolution

**Problem**: Two packages conflict

```bash
# Identify conflicting packages
sudo dnf repoquery --whatconflicts package-a
sudo dnf repoquery --whatconflicts package-b

# Check which package provides the conflicting file
sudo dnf provides /path/to/conflicting/file

# Remove conflicting package first
sudo dnf remove conflicting-package
sudo dnf install desired-package

# Use --allowerasing to allow DNF to remove conflicts
sudo dnf install --allowerasing package
```

---

## Verification Procedures

### Repository Configuration Verification

```bash
# Verify repositories are configured and enabled
sudo dnf repolist

# Verify repository metadata is current
sudo dnf repolist -v

# Verify GPG keys are imported
rpm -qa gpg-pubkey* | sort

# Verify repository files exist
ls -la /etc/yum.repos.d/*.repo

# Test repository connectivity
sudo dnf makecache
```

### Package Installation Verification

```bash
# Verify package is installed
dnf list installed nginx

# Verify package version
rpm -qi nginx

# Verify package files are intact
rpm -V nginx

# Verify package is running (for services)
systemctl status nginx

# Verify package binaries work
nginx -v
```

### Repository Functionality Verification

```bash
# Test package search works
dnf search httpd | head -5

# Test package info retrieval
dnf info curl | head -10

# Test package installation
dnf install -y tree
which tree
dnf remove -y tree

# Test update check works
dnf check-update | head -10
```

### Flatpak Verification

```bash
# Verify Flatpak is installed
flatpak --version

# Verify remotes are configured
flatpak remotes

# Verify application installation
flatpak list | grep -i libreoffice

# Verify application launches
flatpak run org.libreoffice.LibreOffice --version

# Verify permissions are set
flatpak override org.libreoffice.LibreOffice

# Verify runtimes are installed
flatpak list --runtime
```

### DNF Cache Verification

```bash
# Verify cache is functional
du -sh /var/cache/dnf/

# Verify cache can be cleared and rebuilt
sudo dnf clean all
sudo dnf makecache

# Verify log files exist
ls -la /var/log/dnf*
```

---

## RHCSA Exam Notes

### Exam Relevance

Software management is a core RHCSA objective. Expect 2-4 questions involving package installation, repository configuration, and potentially Flatpak management.

### Key Exam Points

1. **Repository Configuration**: You may be asked to configure a new repository from a given URL. Know the `.repo` file format and required fields (`baseurl`, `enabled`, `gpgcheck`).

2. **DNF Commands**: Know how to install, remove, update, and query packages. The exam may ask you to install a specific package or find which package provides a specific file.

3. **Package Queries**: Be comfortable with `dnf provides` and `dnf info`. The exam often asks you to identify which package owns a file or provides a capability.

4. **Flatpak**: RHEL 10 exams include Flatpak tasks. Know how to add a remote, install an application, and manage permissions.

5. **GPG Keys**: Know how to import GPG keys and troubleshoot GPG verification failures.

6. **Module Streams**: Understand how to list, enable, and install module streams when multiple versions are available.

### Common Exam Traps

- Forgetting to run `dnf makecache` after adding a new repository
- Using `yum` instead of `dnf` (both work, but DNF is the standard)
- Not enabling a repository (`enabled=1`) in the config file
- Confusing `dnf remove` with `dnf autoremove` (the latter removes dependencies too)
- Forgetting that Flatpak application IDs use reverse domain notation (e.g., `org.gnome.App`)
- Not specifying the remote name when installing Flatpak packages
- Using wrong GPG key path in repository configuration

### Time Management Tips

- Use `dnf install -y` to skip confirmation prompts and save time
- Use `dnf search` quickly to find package names when unsure
- Keep `dnf makecache` in mind when packages are not found
- Use `dnf provides */filename` to find package ownership
- For Flatpak, remember `flatpak install flathub <app-id>` pattern

### Exam Environment Notes

- DNF and Flatpak are pre-installed in the exam environment
- Repository URLs will be provided in exam tasks
- GPG keys may need to be imported manually
- Flatpak remotes may or may not be pre-configured
- Internet access is available for repository operations
- Tasks may require installing packages as prerequisites for other exam objectives

---

## Chapter Summary

This chapter covered the complete software management stack in RHEL 10, from low-level RPM package handling to high-level DNF repository management and Flatpak application deployment. You learned how to configure repositories (local, remote, CDN), manage packages through DNF (install, update, remove, query), handle dependencies, manage module streams, and work with the Flatpak ecosystem including remotes, applications, permissions, and runtimes.

Software management is fundamental to system administration. Whether deploying new services, patching security vulnerabilities, or recovering from accidental package removal, these skills are used daily. The DNF module system in RHEL 10 provides flexible version management, while Flatpak offers distribution-agnostic application deployment with strong sandboxing guarantees.

---

## Quick Reference

### Repository Management

```bash
dnf repolist                        # List enabled repos
dnf repolist all                    # List all repos
dnf makecache                       # Build metadata cache
dnf clean all                       # Clear all caches
```

### DNF Package Operations

```bash
dnf install package                 # Install package
dnf install -y package              # Install without prompt
dnf remove package                  # Remove package
dnf autoremove package              # Remove with dependencies
dnf update                          # Update all packages
dnf update package                  # Update specific package
dnf check-update                    # Check for updates
dnf reinstall package               # Reinstall package
```

### DNF Package Queries

```bash
dnf search keyword                  # Search packages
dnf info package                    # Package details
dnf list installed                  # List installed packages
dnf list available                  # List available packages
dnf provides /path/to/file          # Find package owning file
dnf provides */binary               # Find package providing binary
dnf repoquery -l package            # List package files
```

### DNF History

```bash
dnf history                         # Transaction history
dnf history info N                  # Transaction details
dnf history undo N                  # Undo transaction
dnf history redo N                  # Redo transaction
```

### DNF Groups and Modules

```bash
dnf group list                      # List package groups
dnf group install "Group Name"      # Install group
dnf group remove "Group Name"       # Remove group
dnf module list                     # List modules
dnf module install name:stream/prof # Install module
dnf module enable name:stream       # Enable stream
dnf module reset name               # Reset module
```

### RPM Direct Commands

```bash
rpm -qa                             # List installed packages
rpm -qi package                     # Package info
rpm -ql package                     # List files
rpm -qf /path/to/file              # Find owning package
rpm -V package                      # Verify package
rpm -Va                             # Verify all packages
rpm --import GPGKEY                 # Import GPG key
rpm --rebuilddb                     # Rebuild RPM database
```

### Flatpak Operations

```bash
flatpak remotes                     # List remotes
flatpak remote-add name URL         # Add remote
flatpak remote-delete name          # Remove remote
flatpak search keyword              # Search applications
flatpak install flathub app-id      # Install application
flatpak list                        # List installed apps
flatpak info app-id                 # Application info
flatpak update                      # Update all apps
flatpak uninstall app-id            # Remove application
flatpak uninstall --unused          # Remove unused runtimes
```

### Flatpak Permissions

```bash
flatpak override app-id --network           # Grant network
flatpak override app-id --filesystem=home   # Grant home access
flatpak override app-id --device=all        # Grant device access
flatpak override --reset app-id             # Reset overrides
flatpak override app-id                     # View overrides
```

### Repository Configuration Template

```ini
[repo-id]
name=Repository Name
baseurl=https://mirror.example.com/repo/
enabled=1
gpgcheck=1
gpgkey=https://mirror.example.com/RPM-GPG-KEY
```

---

## Review Questions

1. What is the difference between RPM and DNF, and why is DNF preferred for package management?

2. Where are DNF repository configuration files stored, and what is the minimum required content for a working repository configuration?

3. How do you find which RPM package owns a specific file on the system?

4. What command would you use to undo the last DNF transaction?

5. What is the purpose of the `dnf module` command, and how does it differ from regular package installation?

6. How do you verify the integrity of an installed RPM package?

7. What is the NEVRA format, and what does each component represent?

8. How do you add Flathub as a Flatpak remote and install an application from it?

9. What is the difference between `dnf remove` and `dnf autoremove`?

10. How do you temporarily disable GPG checking for a single DNF command?

11. What command clears all DNF caches and forces a metadata refresh?

12. How do you grant filesystem access to a specific Flatpak application?

13. What is the purpose of `dnf provides`, and when would you use it?

14. How do you check for available package updates without installing them?

15. What happens when you run `flatpak uninstall --unused`?

---

## Answers

1. RPM is the low-level package format and database that tracks installed files, permissions, and metadata. It handles individual `.rpm` files but does not automatically resolve dependencies. DNF is the high-level package manager that sits on top of RPM, providing automatic dependency resolution, repository management, transaction rollback, and a user-friendly interface. DNF is preferred because it automates dependency resolution, manages multiple repositories, and provides transaction safety with undo capability.

2. DNF repository configuration files are stored in `/etc/yum.repos.d/` with a `.repo` extension. The minimum required content includes a repository ID in square brackets (e.g., `[repo-name]`), a `name` field for human-readable identification, a `baseurl` (or `mirrorlist`/`metalink`) specifying the repository location, and `enabled=1` to activate the repository. GPG checking is recommended with `gpgcheck=1` and a `gpgkey` directive.

3. Use `dnf provides /path/to/file` or `rpm -qf /path/to/file`. Both commands query the RPM database to identify which installed package owns the specified file. The `dnf provides` command also searches available (not yet installed) packages in configured repositories.

4. First run `dnf history list` to identify the transaction ID number, then run `dnf history undo TRANSACTION_ID`. For example, `dnf history undo 42` would undo transaction number 42. The undo operation reverses all package installs, updates, and removals performed in that transaction.

5. The `dnf module` command manages software collections that provide multiple versions (streams) of related packages. Unlike regular package installation which installs a single version, modules allow you to choose between different version streams (e.g., `nodejs:18` vs `nodejs:20`) and profiles (subsets of packages like `common`, `minimal`, `development`). This enables running multiple versions of software without conflicts.

6. Use `rpm -V package-name` to verify a specific package. This command compares the installed files against the RPM database records, checking file sizes, permissions, ownership, SELinux contexts, and MD5 checksums. Use `rpm -Va` to verify all installed packages. Any discrepancies are reported with flag characters (S=size, M=mode, 5=MD5, etc.).

7. NEVRA stands for Name-Epoch-Version-Release-Architecture. **Name** is the package name (e.g., `nginx`). **Epoch** is an optional integer for complex versioning schemes (rarely used). **Version** is the software version (e.g., `1.24.0`). **Release** is the package build number including distribution tag (e.g., `1.el10`). **Architecture** specifies the target CPU architecture (e.g., `x86_64`, `noarch`).

8. Run `sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo` to add Flathub. Then install an application with `flatpak install flathub com.application.Name`. The `--if-not-exists` flag prevents errors if Flathub is already configured. Application IDs use reverse domain notation.

9. `dnf remove package` removes only the specified package. `dnf autoremove package` removes the specified package along with any dependencies that were automatically installed for it and are no longer needed by any other installed package. Autoremove helps prevent orphaned packages from accumulating on the system.

10. Use the `--nogpgcheck` flag: `dnf install --nogpgcheck package-name`. This temporarily disables GPG signature verification for that single command only. This should only be used in trusted environments or for troubleshooting, as it bypasses package integrity verification.

11. Run `sudo dnf clean all` to clear all cached packages, metadata, and headers. Then run `sudo dnf makecache` to download fresh metadata from all configured repositories. This two-step process ensures DNF works with current repository information.

12. Use `flatpak override com.application.Name --filesystem=PATH` where PATH is the directory to grant access to. Common values include `home` (user home directory), `host` (entire filesystem), or specific paths like `~/Documents`. For example: `flatpak override org.gimp.GIMP --filesystem=home`.

13. `dnf provides` searches for packages that provide a specific file, capability, or pattern. You would use it when you know a filename or binary name but not the package name. For example, `dnf provides */ssh` finds the package that provides the `ssh` binary, or `dnf provides /etc/ssh/sshd_config` finds the package owning that configuration file.

14. Use `dnf check-update` to check for available updates without installing them. This command contacts configured repositories, compares installed package versions with available versions, and reports any available updates. Alternatively, `dnf list updates` shows a detailed list of available updates with version information.

15. `flatpak uninstall --unused` removes runtime packages and extensions that are no longer required by any installed Flatpak application. When you uninstall an application, its runtime may remain installed if other applications share it. This command cleans up orphaned runtimes, freeing disk space. It does not remove any installed applications, only unused supporting packages.
