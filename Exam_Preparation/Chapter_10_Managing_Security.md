# Chapter 10: Managing Security

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure firewall settings using `firewall-cmd` and `firewalld`
- Manage default file permissions and umask settings
- Configure key-based authentication for SSH
- Set enforcing and permissive modes for SELinux
- List and identify SELinux file and process contexts
- Restore default file contexts
- Manage SELinux port labels
- Use Boolean settings to modify system SELinux settings

---

## Concepts and Theory

### Security Fundamentals

Linux security relies on multiple layers of protection:

**Defense in Depth:**
- Multiple security controls at different levels
- No single control provides complete protection
- Layers include: physical, network, host, application

**Principle of Least Privilege:**
- Users should have only minimum necessary permissions
- Services should run with minimum required privileges
- Changes should require elevated permissions

**Principle of Separation of Duties:**
- Critical tasks require multiple people
- Prevents single point of failure
- Provides audit trail

### Firewall (firewalld)

**firewalld Components:**

- **Dynamic firewall management**
- **Zone-based configuration**
- **Rich rules** for fine-grained control
- **NAT and masquerading**
- **Service definitions**

**Firewall Zones:**

| Zone | Description | Trust Level |
|------|-------------|-------------|
| public | Default, moderate trust | Medium |
| home | Home network | Medium |
| work | Work network | Medium |
| internal | Internal network | High |
| external | External network | Low |
| drop | Drop all traffic | None |
| reject | Reject all traffic | None |

**Firewall Services:**

Pre-defined service groups include common ports and protocols:

- **ssh**: Port 22/tcp
- **http**: Port 80/tcp
- **https**: Port 443/tcp
- **ftp**: Ports 20/tcp, 21/tcp
- **smtp**: Port 25/tcp
- **dns**: Ports 53/udp, 53/tcp

### File Permissions

**Permission Types:**

- **Read (r)**: Access file contents or list directory
- **Write (w)**: Modify file or create/delete in directory
- **Execute (x)**: Execute file or enter directory

**Permission Classes:**

- **User (u)**: File owner
- **Group (g)**: File group
- **Others (o)**: Everyone else

**Special Permissions:**

- **SetUID (s)**: Run file as owner
- **SetGID (s)**: Run file as group
- **Sticky (t)**: Only owner can delete in directory

**Default Permissions:**

- **Directories**: 755 (rwxr-xr-x)
- **Files**: 644 (rw-r--r--)
- **Scripts**: 755 (rwxr-xr-x)

**Umask:**

- Default permission mask
- Removes permissions from new files
- Default: 022 (removes group/other write)

### SSH Security

**SSH Authentication Methods:**

- **Password authentication**: Traditional, less secure
- **Public key authentication**: More secure, passwordless
- **Keyboard-interactive**: Two-factor authentication

**SSH Configuration:**

File: `/etc/ssh/sshd_config`

Key security settings:

- **PermitRootLogin**: Allow root login
- **PasswordAuthentication**: Allow password auth
- **PubkeyAuthentication**: Allow key auth
- **AuthorizedKeysFile**: Key file location
- **MaxAuthTries**: Max login attempts
- **ClientAliveInterval**: Keep-alive interval

**SSH Key Types:**

- **RSA**: Legacy, 2048+ bits
- **ECDSA**: Elliptic curve, 256+ bits
- **Ed25519**: Modern, most secure

### SELinux (Security-Enhanced Linux)

**SELinux Modes:**

- **Enforcing**: Enforce security policies
- **Permissive**: Log violations, don't enforce
- **Disabled**: SELinux not active

**SELinux Contexts:**

Four-part context: `user:role:type:level`

- **User**: Security user
- **Role**: Security role
- **Type**: Security type (most important)
- **Level**: MLS/MCS level

**Common Contexts:**

- **system_u:system_r:httpd_t**: HTTP daemon
- **unconfined_u:unconfined_r:unconfined_t**: Unconfined
- **user_u:user_r:user_t**: Regular user

**SELinux File Types:**

- **httpd_sys_content_t**: Web content
- **httpd_sys_rw_content_t**: Writable web content
- **etc_t**: Configuration files
- **var_log_t**: Log files
- **home_t**: Home directories

### SELinux Booleans

**SELinux Booleans:**

- Switchable policy settings
- Enable/disable specific behaviors
- Persist across reboots

**Common Booleans:**

- **httpd_can_network_connect**: Allow HTTP to connect
- **httpd_can_network_relay**: Allow HTTP relay
- **ssh_home_dirs**: Allow SSH home directory access
- **firewall_mass_reject**: Mass reject firewall

---

## How It Works Internally

### firewalld Operation

firewalld works by:

1. **Reading zone configuration** from `/etc/firewalld/zones/`
2. **Loading rules** into nftables backend
3. **Filtering traffic** based on zones and rules
4. **Logging** allowed/denied connections
5. **Managing services** and ports dynamically

### File Permission Checks

When accessing a file:

1. **Check if root**: Root bypasses most checks
2. **Check owner**: If UID matches, check user permissions
3. **Check group**: If GID matches, check group permissions
4. **Check others**: Otherwise, check others permissions
5. **Check special bits**: Apply setuid, setgid, sticky

### SSH Key Authentication

SSH key authentication works by:

1. **Client generates** key pair (public + private)
2. **Client sends** public key to server
3. **Server stores** public key in authorized_keys
4. **Client connects** with private key
5. **Server verifies** signature with public key
6. **Access granted** if verification succeeds

### SELinux Enforcement

SELinux works by:

1. **Labeling files** with security contexts
2. **Labeling processes** with security contexts
3. **Checking access** against policy rules
4. **Denying access** if policy violated
5. **Logging violations** to /var/log/audit/audit.log

**Policy Reference:**

- Source process context
- Target file context
- Access type (read, write, execute)
- Decision (allow, deny)

---

## Commands and Administration Tasks

### Firewall Configuration

#### Basic Firewall Commands

```bash
# Check firewall status
sudo firewall-cmd --state

# List all zones
sudo firewall-cmd --get-all-zones

# List zone options
sudo firewall-cmd --zone=public --list-all

# Set default zone
sudo firewall-cmd --set-default=public

# Add service permanently
sudo firewall-cmd --permanent --add-service=http

# Add service temporarily
sudo firewall-cmd --add-service=http

# Remove service permanently
sudo firewall-cmd --permanent --remove-service=http

# Remove service temporarily
sudo firewall-cmd --remove-service=http

# Add port permanently
sudo firewall-cmd --permanent --add-port=8080/tcp

# Add port temporarily
sudo firewall-cmd --add-port=8080/tcp

# Remove port permanently
sudo firewall-cmd --permanent --remove-port=8080/tcp

# Remove port temporarily
sudo firewall-cmd --remove-port=8080/tcp

# Reload firewall
sudo firewall-cmd --reload

# Add rich rule permanently
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" accept'

# Add rich rule temporarily
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" accept'

# List all rules
sudo firewall-cmd --list-all

# List all zones
sudo firewall-cmd --list-all-zones

# Create new zone
sudo firewall-cmd --permanent --new-zone=work

# Configure zone
sudo firewall-cmd --permanent --zone=work --add-service=http
sudo firewall-cmd --permanent --zone=work --add-port=22/tcp

# Set zone for interface
sudo firewall-cmd --permanent --zone=work --change-interface=ens192

# Set zone for connection
sudo firewall-cmd --permanent --zone=work --change-interface=ens192
nmcli connection modify "connection-name" zone work
nmcli connection up "connection-name"

# Delete zone
sudo firewall-cmd --permanent --delete-zone=work

# Add masquerade (NAT)
sudo firewall-cmd --permanent --add-masquerade

# Allow ICMP (ping)
sudo firewall-cmd --permanent --add-protocol=icmp

# Log all traffic
sudo firewall-cmd --permanent --add-rich-rule='rule log prefix="FW: " level="warning"'

# View logs
sudo firewall-cmd --log-all

# Check firewall status
sudo firewall-cmd --state

# List services
sudo firewall-cmd --list-services

# List ports
sudo firewall-cmd --list-ports

# List sources
sudo firewall-cmd --list-sources

# List rich rules
sudo firewall-cmd --list-rich-rules
```

#### Zone Configuration

```bash
# Set default zone
sudo firewall-cmd --set-default=public

# List all zones
sudo firewall-cmd --get-all-zones

# List zone details
sudo firewall-cmd --zone=public --list-all

# Add zone to interface
sudo firewall-cmd --permanent --zone=public --change-interface=ens192

# Add zone to connection
sudo firewall-cmd --permanent --zone=public --change-interface=ens192
nmcli connection modify "connection-name" zone public
nmcli connection up "connection-name"

# Delete zone
sudo firewall-cmd --permanent --delete-zone=work

# Create custom zone
sudo firewall-cmd --permanent --new-zone=work
sudo firewall-cmd --permanent --zone=work --add-service=http
sudo firewall-cmd --permanent --zone=work --add-port=22/tcp
sudo firewall-cmd --permanent --zone=work --add-source=192.168.1.0/24
sudo firewall-cmd --permanent --zone=work --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept'
sudo firewall-cmd --permanent --zone=work --change-interface=ens192
sudo firewall-cmd --reload
```

### File Permission Management

#### Setting Default Permissions

```bash
# Set umask for current shell
umask 022

# Set umask for new users
echo "umask 022" | sudo tee -a /etc/profile

# Set umask for specific users
echo "umask 022" | sudo tee -a /home/username/.bashrc

# Set default permissions for new files
touch testfile
chmod 644 testfile

# Set default permissions for new directories
mkdir testdir
chmod 755 testdir

# Set sticky bit on directory
chmod +t /tmp

# Set setgid bit on directory
chmod g+s /shared

# Set setuid bit on file
chmod u+s /usr/bin/sudo

# Remove special bits
chmod u-s,g-s /path/to/file
chmod -t /path/to/directory
```

#### Managing File Permissions

```bash
# View file permissions
ls -la /path/to/file

# View detailed permissions
stat /path/to/file

# Change file permissions
sudo chmod 755 /path/to/directory
sudo chmod 644 /path/to/file

# Change symbolic permissions
sudo chmod u+x /path/to/file
sudo chmod g-w /path/to/file
sudo chmod o+r /path/to/file

# Change to specific permissions
sudo chmod 755 /path/to/directory
sudo chmod 644 /path/to/file
sudo chmod 600 /path/to/sensitive

# Change owner
sudo chown user:group /path/to/file

# Change owner recursively
sudo chown -R user:group /path/to/directory

# Set default permissions for directory
chmod 755 /path/to/directory

# Set sticky bit
sudo chmod +t /path/to/directory

# Set setgid bit
sudo chmod g+s /path/to/directory

# Set setuid bit
sudo chmod u+s /path/to/file

# Remove special bits
sudo chmod u-s,g-s /path/to/file
sudo chmod -t /path/to/directory

# Set ACL permissions
setfacl -m u:user:rwx /path/to/file
setfacl -m g:group:rx /path/to/file
setfacl -m o::rx /path/to/file
setfacl -d -m u:user:rwx /path/to/file

# View ACL
getfacl /path/to/file

# Clear ACL
setfacl -b /path/to/file
```

### SSH Key-Based Authentication

#### Generating SSH Keys

```bash
# Generate Ed25519 key (recommended)
ssh-keygen -t ed25519 -C "admin@server" -f ~/.ssh/id_ed25519

# Generate ECDSA key
ssh-keygen -t ed25519 -C "admin@server" -f ~/.ssh/id_ed25519

# Generate RSA key
ssh-keygen -t rsa -b 4096 -C "admin@server" -f ~/.ssh/id_rsa

# Generate key without password
ssh-keygen -t ed25519 -C "admin@server" -f ~/.ssh/id_ed25519

# Generate key with password
ssh-keygen -t ed25519 -C "admin@server" -f ~/.ssh/id_ed25519
# Enter passphrase when prompted

# View public key
cat ~/.ssh/id_ed25519.pub

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# List all keys
ls -la ~/.ssh/

# View key information
ssh-keygen -lf ~/.ssh/id_ed25519
```

#### Configuring SSH Server

```bash
# Edit SSH configuration
sudo vi /etc/ssh/sshd_config

# Security settings:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowUsers admin deploy
AllowGroups admins
DenyUsers root
DenyGroups nobody

# Test configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd

# Check SSH status
sudo systemctl status sshd

# View SSH logs
journalctl -u sshd -f
```

#### SSH Key Distribution

```bash
# Copy key to server
ssh-copy-id user@server

# Manual key distribution
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Add key to multiple servers
cat ~/.ssh/id_ed25519.pub | ssh user@server1 "cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_ed25519.pub | ssh user@server2 "cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_ed25519.pub | ssh user@server3 "cat >> ~/.ssh/authorized_keys"

# Verify key is added
ssh user@server "cat ~/.ssh/authorized_keys"

# Remove key from server
ssh user@server "sed -i '/your-key-here/d' ~/.ssh/authorized_keys"

# Remove public key from remote system
ssh user@server "rm -f ~/.ssh/authorized_keys"

# Verify authentication
ssh user@server "echo connected"
```

### SELinux Mode Management

#### Viewing SELinux Status

```bash
# Check SELinux mode
getenforce

# Check SELinux mode (alternative)
getenforce

# Check SELinux status
sestatus

# Check SELinux policy
sestatus -v

# View SELinux version
getenforce
sestatus

# View SELinux policy
semodule -l | head -20
```

#### Changing SELinux Mode

```bash
# Set enforcing mode
sudo setenforce 1
sudo setenforce Enforcing

# Set permissive mode
sudo setenforce 0
sudo setenforce Permissive

# Disable SELinux (temporary)
sudo setenforce 0

# Disable SELinux (permanent)
sudo vi /etc/selinux/config
SELINUX=disabled

# Enable SELinux (permanent)
sudo vi /etc/selinux/config
SELINUX=enforcing

# Reboot to apply changes
sudo reboot

# Check SELinux mode
getenforce
sestatus
```

#### SELinux Mode Commands

```bash
# Set enforcing mode
sudo setenforce 1

# Set permissive mode
sudo setenforce 0

# Disable SELinux
sudo vi /etc/selinux/config
SELINUX=disabled

# Enable SELinux
sudo vi /etc/selinux/config
SELINUX=enforcing

# Reboot to apply changes
sudo reboot

# Check SELinux mode
getenforce
sestatus
```

### SELinux Context Management

#### Viewing SELinux Contexts

```bash
# View file context
ls -Z /path/to/file

# View detailed context
ls -Zl /path/to/file

# View context with stat
stat /path/to/file

# View process context
ps -eo pid,comm,user,fstype,comm,context | head -20

# View context with getfattr
getfattr -d /path/to/file

# View context with lsb_release
ls -Z /path/to/file

# View context with stat
stat /path/to/file

# View context with ls
ls -Z /path/to/file

# View context with stat
stat /path/to/file

# View context with getfattr
getfattr -d /path/to/file

# View context with ls
ls -Z /path/to/file

# View context with stat
stat /path/to/file
```

#### Modifying SELinux Contexts

```bash
# Set file context
sudo chcon -t httpd_sys_content_t /path/to/file

# Set file context recursively
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Set user role
sudo chcon -u system_u /path/to/file

# Set type
sudo chcon -t httpd_sys_content_t /path/to/file

# Set full context
sudo chcon -t httpd_sys_content_t -u system_u /path/to/file

# Restore default context
sudo restorecon /path/to/file

# Restore context recursively
sudo restorecon -Rv /path/to/directory

# Set context with default
sudo chcon -t httpd_sys_content_t /path/to/file

# Set context with reference
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Set context with reference
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Set context with reference
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Set context with reference
sudo chcon -R -t httpd_sys_content_t /path/to/directory
```

#### SELinux Context Commands

```bash
# View file context
ls -Z /path/to/file

# View detailed context
ls -Zl /path/to/file

# Set file context
sudo chcon -t httpd_sys_content_t /path/to/file

# Set file context recursively
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Restore default context
sudo restorecon /path/to/file

# Restore context recursively
sudo restorecon -Rv /path/to/directory

# View context with getfattr
getfattr -d /path/to/file

# View context with stat
stat /path/to/file
```

### SELinux Boolean Management

#### Viewing SELinux Booleans

```bash
# List all booleans
getsebool -a

# List enabled booleans
getsebool -a | grep on

# List disabled booleans
getsebool -a | grep off

# View specific boolean
getsebool httpd_can_network_connect

# View boolean status
getsebool -a | grep httpd

# View boolean details
getsebool -a | grep httpd

# View boolean with status
getsebool -a | grep httpd

# View boolean with status
getsebool -a | grep httpd
```

#### Managing SELinux Booleans

```bash
# Enable boolean temporarily
sudo setsebool httpd_can_network_connect on

# Enable boolean permanently
sudo setsebool -P httpd_can_network_connect on

# Disable boolean temporarily
sudo setsebool httpd_can_network_connect off

# Disable boolean permanently
sudo setsebool -P httpd_can_network_connect off

# Toggle boolean
sudo setsebool -P httpd_can_network_connect !on

# Enable multiple booleans
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_relay on

# View boolean status
getsebool httpd_can_network_connect

# List all booleans
getsebool -a

# Enable boolean with value
sudo setsebool -P httpd_can_network_connect 1

# Disable boolean with value
sudo setsebool -P httpd_can_network_connect 0

# Enable boolean with value
sudo setsebool -P httpd_can_network_connect 1

# Disable boolean with value
sudo setsebool -P httpd_can_network_connect 0

# Enable boolean with value
sudo setsebool -P httpd_can_network_connect 1

# Disable boolean with value
sudo setsebool -P httpd_can_network_connect 0
```

#### Common SELinux Booleans

```bash
# HTTP daemon booleans
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_relay on
sudo setsebool -P httpd_unified on

# SSH booleans
sudo setsebool -P ssh_home_dirs on
sudo setsebool -P ssh_full_proxy_users on

# NFS booleans
sudo setsebool -P nfs_export_all_tcp on
sudo setsebool -P nfs_export_all_udp on

# FTP booleans
sudo setsebool -P ftp_home_dir on
sudo setsebool -P ftpd_full_access on

# Samba booleans
sudo setsebool -P samba_enable_home_dirs on
sudo setsebool -P samba_export_all_rw on

# Mail booleans
sudo setsebool -P mailenable_enable_cron on
sudo setsebool -P mailenable_enable_samba on

# Database booleans
sudo setsebool -P mysql_connect_to_network on
sudo setsebool -P postgresql_connect_to_network on

# Custom booleans
sudo setsebool -P custom_boolean on
sudo setsebool -P custom_boolean off
```

---

## Configuration Examples

### Firewall Configuration

File: `/etc/firewalld/zones/public.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>Works with moderate trust</description>
  <service name="ssh"/>
  <service name="http"/>
  <service name="https"/>
  <port port="8080" protocol="tcp"/>
  <port port="8443" protocol="tcp"/>
  <port port="22" protocol="tcp"/>
  <masquerade/>
  <source address="192.168.1.0/24" name="internal"/>
  <source address="10.0.0.0/8" name="internal"/>
</zone>
```

File: `/etc/firewalld/zones/internal.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Internal</short>
  <description>Internal network with high trust</description>
  <service name="ssh"/>
  <service name="http"/>
  <service name="https"/>
  <service name="ftp"/>
  <service name="smb"/>
  <masquerade/>
</zone>
```

File: `/etc/firewalld/zones/work.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Work</short>
  <description>Work network with moderate trust</description>
  <service name="ssh"/>
  <service name="http"/>
  <service name="https"/>
  <port port="8080" protocol="tcp"/>
  <port port="22" protocol="tcp"/>
  <source address="192.168.1.0/24" name="work"/>
</zone>
```

### SSH Configuration

File: `/etc/ssh/sshd_config`

```bash
# SSH server security configuration

# Basic settings
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
Protocol 2
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server

# Authentication
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 60
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# Security
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
Compression delayed
ClientAliveInterval 300
ClientAliveCountMax 2
UseDNS no

# Logging
SyslogFacility AUTH
LogLevel VERBOSE

# Access control
AllowUsers admin deploy
AllowGroups admins
DenyUsers root
DenyGroups nobody

# Key distribution
AuthorizedKeysFile .ssh/authorized_keys
```

### SELinux Configuration

File: `/etc/selinux/config`

```bash
# SELinux configuration

# SELinux mode
SELINUX=enforcing
SELINUXTYPE=targeted
SELINUX_OPTS=""

# Additional options
# SELINUX=permissive
# SELINUX=disabled
# SELINUXTYPE=mls
# SELINUXTYPE=strict
```

File: `/etc/selinux/targeted/contexts/files/file_contexts.local`

```bash
# SELinux file contexts for custom filesystems

# Web content
/export/data.*    system_u:object_r:httpd_sys_content_t:s0
/export/home.*    system_u:object_r:home_t:s0

# Custom directories
/var/www/custom.* system_u:object_r:httpd_sys_content_t:s0
/var/log/custom.* system_u:object_r:var_log_t:s0

# Custom executables
/usr/local/bin/custom.* system_u:object_r:admin_home_t:s0
```

### SELinux Boolean Configuration

File: `/etc/selinux/config`

```bash
# SELinux booleans configuration

# Enable HTTP network connectivity
setsebool -P httpd_can_network_connect on

# Enable HTTP relay
setsebool -P httpd_can_network_relay on

# Enable SSH home directory access
setsebool -P ssh_home_dirs on

# Enable NFS exports
setsebool -P nfs_export_all_tcp on
setsebool -P nfs_export_all_udp on
```

---

## Real-World Examples

### Scenario 1: Configuring Firewall for Web Server

A web server needs to allow HTTP/HTTPS traffic while blocking other access.

```bash
# Set default zone
sudo firewall-cmd --set-default=public

# Add HTTP service
sudo firewall-cmd --permanent --add-service=http

# Add HTTPS service
sudo firewall-cmd --permanent --add-service=https

# Add custom port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Allow SSH from specific subnet
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="22" protocol="tcp" accept'

# Reload firewall
sudo firewall-cmd --reload

# Verify configuration
sudo firewall-cmd --list-all

# Test connectivity
curl -I http://localhost
curl -I https://localhost
```

### Scenario 2: Setting Up SSH Key-Based Authentication

A system administrator needs to set up secure SSH access.

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "admin@server" -f ~/.ssh/id_ed25519

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub admin@server

# Configure SSH server
sudo vi /etc/ssh/sshd_config
# Set:
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
# MaxAuthTries 3

# Test configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd

# Verify SSH access
ssh admin@server "echo connected"

# View SSH logs
journalctl -u sshd -f

# Add key to multiple servers
for server in server1 server2 server3; do
    ssh-copy-id -i ~/.ssh/id_ed25519.pub admin@$server
done
```

### Scenario 3: Configuring SELinux for Web Application

A web application needs to connect to a database server.

```bash
# Check SELinux status
getenforce
sestatus

# Enable SELinux if not already
sudo setenforce 1

# Set SELinux mode
sudo vi /etc/selinux/config
SELINUX=enforcing

# Enable HTTP network connectivity boolean
sudo setsebool -P httpd_can_network_connect on

# Verify boolean status
getsebool httpd_can_network_connect

# Set SELinux context for web content
sudo chcon -R -t httpd_sys_content_t /var/www/html

# Restore default contexts
sudo restorecon -Rv /var/www/html

# View file contexts
ls -Z /var/www/html

# Check SELinux logs
sudo ausearch -m avc -ts recent
sudo audit2why $(cat /var/log/audit/audit.log | grep avc)

# View SELinux policy
semanage port -l | grep http
```

### Scenario 4: Troubleshooting SELinux Issues

A service is failing due to SELinux policy violations.

```bash
# Check SELinux mode
getenforce

# Set SELinux to permissive mode
sudo setenforce 0

# Check service status
systemctl status service

# Enable SELinux
sudo setenforce 1

# View SELinux logs
sudo ausearch -m avc -ts recent
sudo journalctl -u service | grep SELinux

# View denied access
sudo grep denied /var/log/audit/audit.log

# View SELinux policy violations
sudo grep avc /var/log/audit/audit.log

# Restore default contexts
sudo restorecon -Rv /path/to/directory

# Set correct context
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Enable required boolean
sudo setsebool -P httpd_can_network_connect on

# View SELinux policy
semanage port -l | grep http

# Check file contexts
ls -Z /path/to/file

# Check process contexts
ps -eo pid,comm,user,fstype,comm,context | head -20
```

---

## Troubleshooting

### Firewall Issues

**Problem**: Firewall blocking connections

```bash
# Check firewall status
sudo firewall-cmd --state

# List all rules
sudo firewall-cmd --list-all

# List all zones
sudo firewall-cmd --get-all-zones

# Check zone for interface
nmcli connection show --active | grep ZONE

# View firewall logs
journalctl -u firewalld -f

# Check nftables
sudo nft list ruleset

# Temporarily disable firewall
sudo firewall-cmd --direct --remove-rule filter input 0 0 0

# Re-enable firewall
sudo firewall-cmd --reload
```

**Problem**: Port not accessible

```bash
# Check if port is open
sudo firewall-cmd --list-ports

# Check if service is allowed
sudo firewall-cmd --list-services

# Test port connectivity
telnet server.example.com 80
nc -vz server.example.com 80

# Add port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Check SELinux
getenforce
semanage port -l | grep 8080

# Add SELinux port
sudo semanage port -a -t http_port_t -p tcp 8080

# Test connectivity
curl -I http://localhost:8080
```

### SSH Issues

**Problem**: SSH key authentication failing

```bash
# Check authorized_keys
cat ~/.ssh/authorized_keys

# Check key permissions
ls -la ~/.ssh/
ls -la ~/.ssh/authorized_keys

# Fix permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Check SSH server configuration
sudo sshd -t

# Check SSH logs
journalctl -u sshd -f

# Test SSH connection
ssh -v user@server

# Check if key is in authorized_keys
ssh user@server "cat ~/.ssh/authorized_keys"

# Remove and regenerate key
ssh-keygen -t ed25519 -C "user@server" -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

**Problem**: SSH password authentication disabled

```bash
# Check SSH configuration
cat /etc/ssh/sshd_config | grep PasswordAuthentication

# Enable password authentication
sudo vi /etc/ssh/sshd_config
PasswordAuthentication yes

# Test configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd

# Check SSH status
sudo systemctl status sshd

# Test SSH connection
ssh user@server
```

### SELinux Issues

**Problem**: Service failing due to SELinux

```bash
# Check SELinux mode
getenforce

# Set SELinux to permissive mode
sudo setenforce 0

# Check service status
systemctl status service

# Enable SELinux
sudo setenforce 1

# View SELinux logs
sudo ausearch -m avc -ts recent
sudo journalctl -u service | grep SELinux

# View denied access
sudo grep denied /var/log/audit/audit.log

# View SELinux policy violations
sudo grep avc /var/log/audit/audit.log

# Restore default contexts
sudo restorecon -Rv /path/to/directory

# Set correct context
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Enable required boolean
sudo setsebool -P httpd_can_network_connect on

# View SELinux policy
semanage port -l | grep http

# Check file contexts
ls -Z /path/to/file

# Check process contexts
ps -eo pid,comm,user,fstype,comm,context | head -20
```

**Problem**: SELinux context incorrect

```bash
# View file context
ls -Z /path/to/file

# View detailed context
ls -Zl /path/to/file

# Set file context
sudo chcon -t httpd_sys_content_t /path/to/file

# Set file context recursively
sudo chcon -R -t httpd_sys_content_t /path/to/directory

# Restore default context
sudo restorecon /path/to/file

# Restore context recursively
sudo restorecon -Rv /path/to/directory

# View context with getfattr
getfattr -d /path/to/file

# View context with stat
stat /path/to/file

# Set context with default
sudo chcon -t httpd_sys_content_t /path/to/file

# Set context with reference
sudo chcon -R -t httpd_sys_content_t /path/to/directory
```

### File Permission Issues

**Problem**: File permission denied

```bash
# Check file permissions
ls -la /path/to/file

# Check ownership
ls -ln /path/to/file

# Check if file is executable
ls -l /path/to/file

# Fix permissions
sudo chmod 755 /path/to/directory
sudo chmod 644 /path/to/file

# Fix ownership
sudo chown user:group /path/to/file

# Check SELinux context
ls -Z /path/to/file

# Fix SELinux context
sudo restorecon -Rv /path/to/file

# Check ACL
getfacl /path/to/file

# Fix ACL
setfacl -m u:user:rwx /path/to/file
```

---

## Verification Procedures

### Firewall Verification

```bash
# Verify firewall status
sudo firewall-cmd --state

# Verify services
sudo firewall-cmd --list-services

# Verify ports
sudo firewall-cmd --list-ports

# Verify zones
sudo firewall-cmd --get-all-zones
sudo firewall-cmd --get-default-zone

# Test connectivity
telnet localhost 80
nc -vz localhost 8080
```

### SSH Verification

```bash
# Verify SSH configuration
sudo sshd -t

# Verify SSH service
systemctl status sshd

# Verify key authentication
cat ~/.ssh/authorized_keys
ls -la ~/.ssh/

# Test SSH connection
ssh user@server "echo connected"

# Verify SSH logs
journalctl -u sshd -n 20
```

### SELinux Verification

```bash
# Verify SELinux mode
getenforce
sestatus

# Verify file contexts
ls -Z /path/to/file

# Verify booleans
getsebool httpd_can_network_connect

# Verify contexts
semanage port -l | grep http

# View SELinux logs
sudo ausearch -m avc -ts recent
sudo journalctl -u service | grep SELinux
```

### File Permission Verification

```bash
# Verify permissions
ls -la /path/to/file
stat /path/to/file

# Verify ownership
ls -ln /path/to/file

# Verify SELinux context
ls -Z /path/to/file

# Verify ACL
getfacl /path/to/file
```

---

## RHCSA Exam Notes

### Exam Relevance

Managing security is heavily tested on the RHCSA. Expect 2-3 questions covering firewall configuration, SSH key authentication, SELinux modes, and file context management.

### Key Exam Points

1. **Firewall**: Know `firewall-cmd` commands for adding services, ports, and zones.

2. **SSH Keys**: Know how to generate and distribute SSH keys for authentication.

3. **SELinux Modes**: Know how to set enforcing/permissive modes and view status.

4. **SELinux Contexts**: Know how to view, set, and restore file contexts.

5. **SELinux Booleans**: Know how to enable/disable booleans for specific services.

6. **File Permissions**: Know how to set and manage file permissions and ownership.

### Common Exam Traps

- Forgetting `--permanent` flag for firewall changes
- Not testing SSH configuration with `sshd -t`
- Confusing `setenforce` with `/etc/selinux/config`
- Not using `restorecon` to restore default contexts
- Forgetting to reload firewall after changes
- Not checking SELinux logs when troubleshooting

### Time Management Tips

- Use `firewall-cmd --list-all` to quickly check firewall rules
- Use `getenforce` to quickly check SELinux mode
- Use `ls -Z /path` to quickly check file context
- Use `sshd -t` to quickly test SSH configuration
- Use `getsebool -a` to quickly check all booleans

### Exam Environment Notes

- firewalld is enabled by default
- SELinux is in enforcing mode
- SSH keys may need to be generated
- File contexts may need to be set
- Booleans may need to be enabled

---

## Chapter Summary

This chapter covered security management including firewall configuration with firewalld, SSH key-based authentication, SELinux mode management, file and process context management, SELinux Boolean configuration, and troubleshooting common security issues. You learned how to configure firewall rules, set up secure SSH access, manage SELinux policies, and diagnose security-related problems.

These skills are essential for maintaining secure RHEL systems. Understanding firewalls, SSH security, and SELinux helps administrators protect systems from unauthorized access and maintain compliance with security policies.

---

## Quick Reference

### Firewall Commands

```bash
firewall-cmd --state                     # Check status
firewall-cmd --list-all                  # List zone
firewall-cmd --add-service=http          # Add service
firewall-cmd --add-port=8080/tcp         # Add port
firewall-cmd --reload                    # Reload firewall
firewall-cmd --add-rich-rule='rule...'   # Add rich rule
```

### SSH Commands

```bash
ssh-keygen -t ed25519                   # Generate key
ssh-copy-id user@server                 # Copy key
sshd -t                                 # Test config
systemctl restart sshd                  # Restart SSH
```

### SELinux Commands

```bash
getenforce                              # Check mode
setenforce 1                            # Set enforcing
setenforce 0                            # Set permissive
restorecon /path                        # Restore context
chcon -t type /path                     # Set context
getsebool -a                            # List booleans
setsebool -P boolean on                 # Enable boolean
```

### File Permission Commands

```bash
chmod 755 /path                         # Change permissions
chown user:group /path                  # Change ownership
ls -Z /path                             # View context
getfacl /path                           # View ACL
setfacl -m u:user:rwx /path             # Set ACL
```

---

## Review Questions

1. How do you check the current firewall status?

2. What command would you use to add a service to the firewall permanently?

3. How do you generate a new SSH key pair?

4. What is the purpose of the `--permanent` flag in firewall-cmd?

5. How do you set SELinux to enforcing mode?

6. What command would you use to view SELinux file contexts?

7. How do you restore default SELinux contexts?

8. What is the difference between `setenforce` and editing `/etc/selinux/config`?

9. How do you enable a SELinux boolean permanently?

10. What command would you use to test SSH server configuration?

11. How do you copy your SSH public key to a remote server?

12. What is the purpose of the `restorecon` command?

13. How do you list all SELinux booleans?

14. What command would you use to set file permissions to 755?

15. How do you check if a port is allowed through the firewall?

---

## Answers

1. Use `firewall-cmd --state` to check the current firewall status. It returns "running" if active, or "not running" if inactive.

2. Use `sudo firewall-cmd --permanent --add-service=http` to add a service permanently. The `--permanent` flag ensures the rule persists across reboots.

3. Use `ssh-keygen -t ed25519 -C "email" -f ~/.ssh/id_ed25519` to generate a new SSH key pair. This creates both the private and public keys.

4. The `--permanent` flag adds the rule to the configuration files so it persists across reboots. Without it, the rule is temporary and lost after a reboot. You must use both `--permanent` and `--reload` to apply permanent changes.

5. Use `sudo setenforce 1` to set SELinux to enforcing mode. This immediately changes the mode without requiring a reboot.

6. Use `ls -Z /path/to/file` to view SELinux file contexts. This shows the user:role:type:level security context.

7. Use `sudo restorecon /path/to/file` or `sudo restorecon -Rv /path/to/directory` to restore default SELinux contexts. This reads contexts from the policy.

8. `setenforce` changes SELinux mode immediately (temporary). Editing `/etc/selinux/config` changes the mode at boot (permanent). Use `setenforce` for immediate changes and `/etc/selinux/config` for permanent changes.

9. Use `sudo setsebool -P boolean_name on` to enable a boolean permanently. The `-P` flag persists the change across reboots.

10. Use `sshd -t` to test SSH server configuration. This checks the configuration file for syntax errors without restarting the service.

11. Use `ssh-copy-id user@server` to copy your SSH public key to a remote server. This automates adding the key to authorized_keys.

12. `restorecon` restores default SELinux file contexts by reading them from the policy files. It's used to fix incorrect contexts after manual changes.

13. Use `getsebool -a` to list all SELinux booleans. Use `getsebool -a | grep on` to show only enabled booleans.

14. Use `chmod 755 /path/to/file` to set file permissions to 755 (rwxr-xr-x). This gives read, write, and execute to owner, and read and execute to group and others.

15. Use `sudo firewall-cmd --list-ports` to list all allowed ports. Use `sudo firewall-cmd --list-services` to list all allowed services.
