# Chapter 8: Managing Basic Networking

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure IPv4 and IPv6 addresses using NetworkManager
- Configure hostname resolution with DNS
- Configure network services to start automatically at boot
- Restrict network access using firewalld and firewall-cmd
- Use `nmcli` and `nmtui` for NetworkManager configuration
- Troubleshoot network connectivity issues
- Configure static and DHCP network connections

---

## Concepts and Theory

### NetworkManager Overview

NetworkManager is the default network management daemon in RHEL 10. It provides:

- Automatic connection management
- Multiple protocol support (Ethernet, Wi-Fi, bonding, teaming)
- IPv4 and IPv6 address management
- DNS and hostname configuration
- Network service integration

**NetworkManager Commands:**

- `nmcli`: Command-line interface
- `nmtui`: Text-based user interface
- `nm-tool`: Quick status tool

### Network Interfaces

**Interface Types:**

- **Ethernet**: Wired network connections
- **Wi-Fi**: Wireless network connections
- **Bonding**: Multiple interfaces combined
- **Teaming**: Multiple interfaces combined
- **Loopback**: Local interface (lo)
- **Virtual**: Bridge, VLAN, etc.

**Interface Naming:**

- Predictable network interface names (PCI addresses)
- Examples: `ens192`, `enp3s0`, `eth0`
- Format: `e` + PCI bus + `s` + slot + number

### IPv4 Addressing

**Address Types:**

- **Static**: Manually configured address
- **DHCP**: Dynamic Host Configuration Protocol
- **DHCP with Static IP**: DHCP with fixed address

**IPv4 Components:**

- **IP Address**: 32-bit identifier (e.g., 192.168.1.10)
- **Subnet Mask**: Network boundary (e.g., 255.255.255.0)
- **Gateway**: Router for external access
- **DNS Servers**: Domain name resolution

**CIDR Notation:**

- `/24` = 255.255.255.0 (256 addresses)
- `/16` = 255.255.0.0 (65,536 addresses)
- `/8` = 255.0.0.0 (16,777,216 addresses)

### IPv6 Addressing

**IPv6 Components:**

- **128-bit address space**
- **Hexadecimal notation** (e.g., 2001:0db8:85a3:0000:0000:8a2e:0370:7334)
- **Address types**: Unicast, Multicast, Anycast
- **SLAAC**: Stateless Address Autoconfiguration
- **DHCPv6**: Dynamic Host Configuration Protocol for IPv6

**IPv6 Abbreviations:**

- Leading zeros can be omitted: `0db8` → `db8`
- Consecutive zeros can be replaced with `::`: `2001:0db8:0000:0000:0000:0000:0000:0001` → `2001:db8::1`

### Hostname Resolution

**Resolution Methods:**

1. **/etc/hosts**: Local hostname file
2. **DNS**: Domain Name System
3. **mDNS**: Multicast DNS (e.g., .local)
4. **NetBIOS**: Windows network names

**/etc/hosts Format:**

```
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
127.0.1.1   hostname.domain.com hostname
```

**DNS Configuration:**

- **Resolv.conf**: DNS resolver configuration
- **Nameservers**: IP addresses of DNS servers
- **Search domains**: Domain search list
- **Options**: Resolver behavior flags

### Network Services

**Network Services:**

- **NetworkManager**: Network management daemon
- **firewalld**: Dynamic firewall daemon
- **chronyd**: Time synchronization
- **sshd**: Secure shell daemon
- **httpd**: Web server

**Service Dependencies:**

- Services should start after network is ready
- `network-online.target`: Network fully online
- `network.target`: Network interfaces configured

### Firewall (firewalld)

**firewalld Components:**

- **Dynamically managed firewall**
- **Zone-based configuration**
- **Rich rules** for fine-grained control
- **NAT (Network Address Translation)**
- **Masquerading** for outbound traffic

**Firewall Zones:**

- **public**: Default zone, moderate trust
- **home**: Home network, moderate trust
- **work**: Work network, moderate trust
- **internal**: Internal network, high trust
- **external**: External network, low trust
- **drop**: Drop all incoming traffic
- **reject**: Reject all incoming traffic

---

## How It Works Internally

### NetworkManager Operation

NetworkManager works by:

1. **Scanning** available network interfaces
2. **Reading** connection profiles from `/etc/NetworkManager/system-connections/`
3. **Applying** active connections to interfaces
4. **Managing** DHCP, static IPs, and routing
5. **Handling** connection state changes

### DHCP Process

DHCP works by:

1. **DHCP Discover**: Client broadcasts request
2. **DHCP Offer**: Server responds with offer
3. **DHCP Request**: Client requests the offer
4. **DHCP ACK**: Server confirms and sends lease

### DNS Resolution

DNS resolution works by:

1. **Client queries** local resolver
2. **Resolver checks** /etc/hosts first
3. **Resolver queries** DNS servers
4. **DNS servers** query other servers if needed
5. **Response** returned to client

### firewalld Operation

firewalld works by:

1. **Reading** zone configuration
2. **Loading** rules into nftables backend
3. **Filtering** incoming/outgoing traffic
4. **Logging** allowed/denied connections
5. **Managing** services and ports

---

## Commands and Administration Tasks

### NetworkManager Configuration

#### Using nmcli

```bash
# View NetworkManager status
nmcli status

# View all connections
nmcli connection show

# View active connections
nmcli connection show --active

# View device status
nmcli device status

# View all devices
nmcli device show

# View IP addresses
nmcli device show | grep IP4
nmcli device show | grep IP6

# View connection details
nmcli connection show "connection-name"

# View device details
nmcli device show eth0
```

#### Creating Connections

```bash
# Create Ethernet connection
nmcli connection add type ethernet ifname ens192 con-name MyEthernet autoconnect yes

# Create Wi-Fi connection
nmcli connection add type wifi ifname wlp2s0 con-name MyWiFi autoconnect yes
nmcli connection modify MyWiFi wifi.secret "password"

# Create bond connection
nmcli connection add type bond ifname bond0 con-name MyBond autoconnect yes
nmcli connection modify MyBond bond.options "miimon=100 mode=802.3ad"
nmcli connection add "slave1" type bond-slave ifname ens192 master bond0
nmcli connection add "slave2" type bond-slave ifname ens200 master bond0

# Create VLAN connection
nmcli connection add type vlan ifname ens192 con-name MyVLAN id 100 autoconnect yes

# Create team connection
nmcli connection add type team ifname team0 con-name MyTeam autoconnect yes
nmcli connection add "slave1" type team-slave ifname ens192 master team0
```

#### Modifying Connections

```bash
# Modify connection name
nmcli connection modify "connection-name" connection.id "new-name"

# Modify IP address (static)
nmcli connection modify "connection-name" ipv4.addresses 192.168.1.10/24
nmcli connection modify "connection-name" ipv4.gateway 192.168.1.1
nmcli connection modify "connection-name" ipv4.method manual
nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify "connection-name" ipv4.dns-search "example.com"

# Modify IP address (DHCP)
nmcli connection modify "connection-name" ipv4.method auto
nmcli connection modify "connection-name" ipv6.method auto

# Modify Wi-Fi settings
nmcli connection modify "connection-name" wifi.security WPA2
nmcli connection modify "connection-name" wifi.key-mgmt wpa-psk
nmcli connection modify "connection-name" wifi-psk "password"

# Modify DNS
nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify "connection-name" ipv4.dns-search "example.com"

# Modify MTU
nmcli connection modify "connection-name" connection.mtu 9000

# Modify zone
nmcli connection modify "connection-name" zone internal
```

#### Activating Connections

```bash
# Activate connection
nmcli connection up "connection-name"

# Deactivate connection
nmcli connection down "connection-name"

# Delete connection
nmcli connection delete "connection-name"

# Rename connection
nmcli connection rename "old-name" "new-name"

# Clone connection
nmcli connection add type ethernet con-name "new-connection" clone "existing-connection"
```

#### Using nmtui

```bash
# Start nmtui
nmtui

# Menu options:
# 1. Show all connections
# 2. Add a connection
# 3. Edit a connection
# 4. Delete a connection
# 5. Select a connection to activate
# 6. Select a connection to deactivate
# 7. Quit
```

### IPv4 Configuration

#### Static IPv4 Configuration

```bash
# Using nmcli
nmcli connection add type ethernet ifname ens192 con-name StaticIPv4 autoconnect yes
nmcli connection modify StaticIPv4 ipv4.addresses 192.168.1.10/24
nmcli connection modify StaticIPv4 ipv4.gateway 192.168.1.1
nmcli connection modify StaticIPv4 ipv4.method manual
nmcli connection modify StaticIPv4 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify StaticIPv4 ipv4.dns-search "example.com"
nmcli connection up StaticIPv4

# Using ip command
ip addr add 192.168.1.10/24 dev ens192
ip route add default via 192.168.1.1
ip route add 10.0.0.0/8 via 192.168.1.1
ip link set ens192 up

# Using NetworkManager config file
# /etc/NetworkManager/system-connections/StaticIPv4.nmconnection
[connection]
id=StaticIPv4
type=ethernet
interface-name=ens192

[ipv4]
address1=192.168.1.10/24
method=manual
gateway=192.168.1.1
dns=8.8.8.8;8.8.4.4
dns-search=example.com

[ipv6]
method=auto
```

#### DHCP IPv4 Configuration

```bash
# Using nmcli (default)
nmcli connection add type ethernet ifname ens192 con-name DHCPIPv4 autoconnect yes
nmcli connection modify DHCPIPv4 ipv4.method auto
nmcli connection modify DHCPIPv4 ipv6.method auto
nmcli connection up DHCPIPv4

# Using NetworkManager config file
# /etc/NetworkManager/system-connections/DHCPIPv4.nmconnection
[connection]
id=DHCPIPv4
type=ethernet
interface-name=ens192

[ipv4]
method=auto

[ipv6]
method=auto
```

#### IPv4 Configuration Examples

```bash
# Multiple IP addresses
nmcli connection modify "connection-name" ipv4.addresses "192.168.1.10/24 192.168.2.10/24"

# No gateway
nmcli connection modify "connection-name" ipv4.gateway ""

# Multiple DNS servers
nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 1.1.1.1 208.67.222.222"

# No DNS search
nmcli connection modify "connection-name" ipv4.dns-search ""

# Disable IPv4
nmcli connection modify "connection-name" ipv4.method disabled
```

### IPv6 Configuration

#### Static IPv6 Configuration

```bash
# Using nmcli
nmcli connection add type ethernet ifname ens192 con-name StaticIPv6 autoconnect yes
nmcli connection modify StaticIPv6 ipv6.addresses 2001:db8::10/64
nmcli connection modify StaticIPv6 ipv6.gateway 2001:db8::1
nmcli connection modify StaticIPv6 ipv6.method manual
nmcli connection modify StaticIPv6 ipv6.dns "2001:4860:4860::8888 2001:4860:4860::8844"
nmcli connection up StaticIPv6

# Using NetworkManager config file
# /etc/NetworkManager/system-connections/StaticIPv6.nmconnection
[connection]
id=StaticIPv6
type=ethernet
interface-name=ens192

[ipv6]
address1=2001:db8::10/64
method=manual
gateway=2001:db8::1
dns=2001:4860:4860::8888;2001:4860:4860::8844
```

#### DHCPv6 Configuration

```bash
# Using nmcli
nmcli connection add type ethernet ifname ens192 con-name DHCPv6 autoconnect yes
nmcli connection modify DHCPv6 ipv6.method auto
nmcli connection modify DHCPv6 ipv6.dad-disabled true
nmcli connection up DHCPv6

# Using NetworkManager config file
# /etc/NetworkManager/system-connections/DHCPv6.nmconnection
[connection]
id=DHCPv6
type=ethernet
interface-name=ens192

[ipv6]
method=auto
dad-disabled=true
```

#### IPv6 Configuration Examples

```bash
# SLAAC (Stateless Address Autoconfiguration)
nmcli connection modify "connection-name" ipv6.method slaa

# Disable IPv6
nmcli connection modify "connection-name" ipv6.method disabled

# Enable IPv6 privacy extensions
nmcli connection modify "connection-name" ipv6.addr-gen-mode eui64

# View IPv6 addresses
ip -6 addr show
nmcli device show | grep IP6
```

### Hostname Resolution

#### Configuring Hostname

```bash
# View current hostname
hostname
hostnamectl

# Set static hostname
sudo hostnamectl set-hostname myserver.example.com

# Set temporary hostname
hostname myserver

# Edit /etc/hosts
sudo vi /etc/hosts

# Example /etc/hosts:
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
127.0.1.1   myserver.example.com myserver
192.168.1.10 myserver.example.com myserver
```

#### Configuring DNS

```bash
# View DNS configuration
cat /etc/resolv.conf

# Configure DNS using NetworkManager
nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 8.8.4.4 1.1.1.1"
nmcli connection modify "connection-name" ipv4.dns-search "example.com"
nmcli connection up "connection-name"

# Configure DNS using NetworkManager config file
# /etc/NetworkManager/system-connections/Connection.nmconnection
[connection]
id=Connection
type=ethernet
interface-name=ens192

[ipv4]
method=auto
dns=8.8.8.8;8.8.4.4;1.1.1.1
dns-search=example.com

[ipv6]
method=auto

# View DNS servers
nmcli dev show | grep DNS
cat /etc/resolv.conf

# Test DNS resolution
ping -c 4 example.com
nslookup example.com
dig example.com
```

### Network Services Boot Configuration

#### Enabling Network Services

```bash
# Enable NetworkManager
sudo systemctl enable NetworkManager

# Start NetworkManager
sudo systemctl start NetworkManager

# Check NetworkManager status
systemctl status NetworkManager

# Enable network services
sudo systemctl enable network-online.target

# Check network status
systemctl is-active network-online.target

# View network services
systemctl list-units --type=service | grep network
```

#### Service Dependencies

```bash
# Create service that waits for network
sudo vi /etc/systemd/system/myapp.service

[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp

[Install]
WantedBy=multi-user.target

# Enable service
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
```

### Firewall Configuration

#### Basic Firewall Commands

```bash
# Check firewall status
sudo firewall-cmd --state

# List firewall zones
sudo firewall-cmd --get-zones

# List zone options
sudo firewall-cmd --zone=public --list-all

# List all zones
sudo firewall-cmd --get-all-zones

# Set default zone
sudo firewall-cmd --set-default=public

# Add service
sudo firewall-cmd --permanent --add-service=http

# Add port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Remove service
sudo firewall-cmd --permanent --remove-service=http

# Remove port
sudo firewall-cmd --permanent --remove-port=8080/tcp

# Reload firewall
sudo firewall-cmd --reload

# Add temporary rule
sudo firewall-cmd --add-service=http

# Remove temporary rule
sudo firewall-cmd --remove-service=http

# Add rich rule
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" accept'

# List all rules
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all-zones
```

#### Zone Configuration

```bash
# Create new zone
sudo firewall-cmd --permanent --new-zone=work

# Configure zone
sudo firewall-cmd --permanent --zone=work --add-service=http
sudo firewall-cmd --permanent --zone=work --add-port=22/tcp
sudo firewall-cmd --permanent --zone=work --add-source=192.168.1.0/24

# Set zone for interface
sudo firewall-cmd --permanent --zone=work --change-interface=ens192

# Set zone for connection
sudo firewall-cmd --permanent --zone=work --change-interface=ens192
nmcli connection modify "connection-name" zone work
nmcli connection up "connection-name"

# List zone members
sudo firewall-cmd --get-zone-maps=work

# Delete zone
sudo firewall-cmd --permanent --delete-zone=work
```

#### Common Firewall Rules

```bash
# Allow SSH
sudo firewall-cmd --permanent --add-service=ssh

# Allow HTTP/HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Allow custom port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8443/tcp

# Allow from specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept'

# Allow from subnet
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" accept'

# Block from specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.200" reject'

# Log all traffic
sudo firewall-cmd --permanent --add-rich-rule='rule log prefix="FW: " level="warning"'

# Masquerade (NAT)
sudo firewall-cmd --permanent --add-masquerade

# Allow ICMP (ping)
sudo firewall-cmd --permanent --add-protocol=icmp

# View logs
sudo firewall-cmd --log-all
journalctl -f | grep FW:
```

---

## Configuration Examples

### NetworkManager Configuration

File: `/etc/NetworkManager/system-connections/StaticIPv4.nmconnection`

```ini
[connection]
id=StaticIPv4
type=ethernet
interface-name=ens192
autoconnect=true
uuid=a1b2c3d4-e5f6-7890-abcd-ef1234567890

[ethernet]
mac-address=00:11:22:33:44:55

[ipv4]
address1=192.168.1.10/24
method=manual
gateway=192.168.1.1
dns=8.8.8.8;8.8.4.4
dns-search=example.com

[ipv6]
method=auto
addr-gen-mode=eui64
```

File: `/etc/NetworkManager/system-connections/DHCP.nmconnection`

```ini
[connection]
id=DHCP
type=ethernet
interface-name=ens192
autoconnect=true

[ipv4]
method=auto

[ipv6]
method=auto
```

File: `/etc/NetworkManager/system-connections/Bond.nmconnection`

```ini
[connection]
id=Bond
type=bond
interface-name=bond0
autoconnect=true

[bond]
miimon=100
mode=802.3ad
lacp_rate=fast

[ipv4]
address1=192.168.1.10/24
method=manual
gateway=192.168.1.1
dns=8.8.8.8;8.8.4.4

[ipv6]
method=auto
```

File: `/etc/NetworkManager/system-connections/VLAN.nmconnection`

```ini
[connection]
id=VLAN
type=vlan
interface-name=ens192
autoconnect=true

[vlan]
id=100
mac-address=00:11:22:33:44:55

[ipv4]
address1=192.168.100.10/24
method=manual
gateway=192.168.100.1
dns=8.8.8.8;8.8.4.4
```

### /etc/hosts Configuration

```bash
# /etc/hosts
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
::ip6-localhost ip6-prefer-local

# Localhost aliases
127.0.1.1   myserver.example.com myserver

# Network hosts
192.168.1.10  web01.example.com web01
192.168.1.20  db01.example.com db01
192.168.1.30  mail01.example.com mail01

# External hosts
8.8.8.8       dns1.google.com
1.1.1.1       dns1.cloudflare.com
```

### firewalld Configuration

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

---

## Real-World Examples

### Scenario 1: Setting Up a Web Server

A web server needs static IP configuration and firewall rules.

```bash
# Configure static IP
nmcli connection add type ethernet ifname ens192 con-name WebServer autoconnect yes
nmcli connection modify WebServer ipv4.addresses 192.168.1.10/24
nmcli connection modify WebServer ipv4.gateway 192.168.1.1
nmcli connection modify WebServer ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify WebServer ipv4.dns-search "example.com"
nmcli connection modify WebServer zone public
nmcli connection up WebServer

# Configure hostname
hostnamectl set-hostname web01.example.com

# Configure /etc/hosts
echo "192.168.1.10 web01.example.com web01" | sudo tee -a /etc/hosts

# Configure firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-source=192.168.1.0/24
sudo firewall-cmd --reload

# Enable network service
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager

# Enable web server at boot
sudo systemctl enable httpd
sudo systemctl start httpd
```

### Scenario 2: High Availability with Bonding

A server needs redundant network connections for high availability.

```bash
# Create bond connection
nmcli connection add type bond ifname bond0 con-name HA_Bond autoconnect yes
nmcli connection modify HA_Bond bond.options "miimon=100 mode=802.3ad"
nmcli connection modify HA_Bond ipv4.addresses 192.168.1.10/24
nmcli connection modify HA_Bond ipv4.gateway 192.168.1.1
nmcli connection modify HA_Bond ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify HA_Bond zone internal

# Create slave connections
nmcli connection add type bond-slave ifname ens192 con-name ens192.autoconnect yes master bond0
nmcli connection add type bond-slave ifname ens200 con-name ens200.autoconnect yes master bond0

# Activate connections
nmcli connection up HA_Bond

# Configure firewall
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --permanent --zone=internal --add-service=https
sudo firewall-cmd --permanent --zone=internal --add-service=ssh
sudo firewall-cmd --reload
```

### Scenario 3: VLAN Configuration

A server needs to be on multiple VLANs.

```bash
# Create VLAN 10 connection
nmcli connection add type vlan ifname ens192 con-name VLAN10 id 100 autoconnect yes
nmcli connection modify VLAN10 ipv4.addresses 192.168.10.10/24
nmcli connection modify VLAN10 ipv4.gateway 192.168.10.1
nmcli connection modify VLAN10 ipv4.dns "8.8.8.8"

# Create VLAN 20 connection
nmcli connection add type vlan ifname ens192 con-name VLAN20 id 200 autoconnect yes
nmcli connection modify VLAN20 ipv4.addresses 192.168.20.10/24
nmcli connection modify VLAN20 ipv4.gateway 192.168.20.1
nmcli connection modify VLAN20 ipv4.dns "8.8.8.8"

# Activate connections
nmcli connection up VLAN10
nmcli connection up VLAN20

# Configure firewall
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --reload
```

### Scenario 4: Network Troubleshooting

A server has network connectivity issues.

```bash
# Check network status
nmcli status
nmcli device status
nmcli connection show --active

# Check IP configuration
ip addr show
ip route show

# Check DNS resolution
cat /etc/resolv.conf
nslookup example.com
ping -c 4 example.com

# Check firewall status
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# Check network logs
journalctl -u NetworkManager -n 50
journalctl -u firewalld -n 50

# Restart NetworkManager
sudo systemctl restart NetworkManager

# Bring interface down and up
sudo ip link set ens192 down
sudo ip link set ens192 up

# Renew DHCP lease
nmcli connection down ens192
nmcli connection up ens192

# Test connectivity
ping -c 4 192.168.1.1
ping -c 4 8.8.8.8
traceroute 8.8.8.8
```

---

## Troubleshooting

### Network Connectivity Issues

**Problem**: Cannot obtain IP address via DHCP

```bash
# Check NetworkManager status
systemctl status NetworkManager

# Check device status
nmcli device status

# Check connection
nmcli connection show --active

# Bring interface down and up
sudo nmcli connection down ens192
sudo nmcli connection up ens192

# Check for errors
journalctl -u NetworkManager -f

# Renew DHCP lease
sudo dhclient -r ens192
sudo dhclient ens192

# Check DHCP server
nmcli dev show | grep DHCP
```

**Problem**: Static IP not working

```bash
# Check IP configuration
ip addr show

# Check connection settings
nmcli connection show "connection-name"

# Verify /etc/NetworkManager/system-connections/
cat /etc/NetworkManager/system-connections/Static.nmconnection

# Check for typos
grep address /etc/NetworkManager/system-connections/Static.nmconnection

# Restart NetworkManager
sudo systemctl restart NetworkManager

# Check interface is up
ip link show ens192
```

**Problem**: DNS resolution failing

```bash
# Check DNS configuration
cat /etc/resolv.conf

# Check DNS servers
nmcli dev show | grep DNS

# Test DNS
nslookup example.com
dig example.com

# Check DNS service
systemctl status systemd-resolved

# Configure DNS
nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection up "connection-name"

# Check /etc/resolv.conf
cat /etc/resolv.conf
```

**Problem**: Cannot connect to gateway

```bash
# Check routing table
ip route show

# Check gateway
ip route | grep default

# Test gateway
ping -c 4 192.168.1.1

# Check interface
ip addr show ens192

# Check firewall
sudo firewall-cmd --list-all

# Check for routing issues
traceroute 8.8.8.8

# Check ARP
ip neigh show
```

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
```

### NetworkManager Issues

**Problem**: NetworkManager not starting

```bash
# Check status
systemctl status NetworkManager

# Check logs
journalctl -u NetworkManager -f

# Enable and start
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager

# Check configuration
cat /etc/NetworkManager/NetworkManager.conf

# Check interfaces
ls /sys/class/net/

# Check for errors
dmesg | grep -i network
```

**Problem**: Connection not activating

```bash
# Check connection
nmcli connection show

# Check if connection exists
nmcli connection show "connection-name"

# Activate connection
nmcli connection up "connection-name"

# Check for errors
journalctl -u NetworkManager -n 50

# Delete and recreate connection
nmcli connection delete "connection-name"
nmcli connection add type ethernet ifname ens192 con-name "connection-name" autoconnect yes
nmcli connection up "connection-name"
```

---

## Verification Procedures

### Network Configuration Verification

```bash
# Verify network status
nmcli status
nmcli device status

# Verify IP configuration
ip addr show
nmcli device show | grep IP4

# Verify routing
ip route show

# Verify DNS
cat /etc/resolv.conf
nmcli dev show | grep DNS

# Verify hostname
hostname
hostnamectl
```

### Firewall Verification

```bash
# Verify firewall status
sudo firewall-cmd --state
sudo firewall-cmd --list-all

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

### Service Verification

```bash
# Verify NetworkManager
systemctl status NetworkManager

# Verify network-online.target
systemctl is-active network-online.target

# Verify firewall
systemctl status firewalld

# Verify services
systemctl list-units --type=service | grep network
```

---

## RHCSA Exam Notes

### Exam Relevance

Managing basic networking is heavily tested on the RHCSA. Expect 2-3 questions covering NetworkManager configuration, firewall rules, hostname/DNS configuration, and network troubleshooting.

### Key Exam Points

1. **NetworkManager**: Know `nmcli` commands for creating, modifying, and activating connections.

2. **IPv4/IPv6**: Know how to configure static and DHCP addresses using NetworkManager.

3. **Hostname/DNS**: Know how to configure hostname and DNS using NetworkManager and /etc/hosts.

4. **Firewall**: Know `firewall-cmd` commands for adding services, ports, and zones.

5. **Troubleshooting**: Know how to diagnose network connectivity issues.

6. **Service Management**: Know how to enable network services at boot.

### Common Exam Traps

- Forgetting to reload firewall after changes
- Not enabling NetworkManager service
- Using wrong nmcli syntax (missing quotes)
- Forgetting to activate connection after creating
- Not specifying interface name for connections
- Confusing `--permanent` with `--reload` for firewall

### Time Management Tips

- Use `nmcli connection show --active` to quickly check active connections
- Use `firewall-cmd --list-all` to quickly check firewall rules
- Use `ip addr show` to quickly check IP configuration
- Use `systemctl status service` to quickly check service status
- Use `nmcli device status` to quickly check device status

### Exam Environment Notes

- NetworkManager is pre-installed
- Firewall is enabled by default
- You may need to configure connections from scratch
- Test connectivity with ping and nslookup
- Use `nmcli` for most network tasks

---

## Chapter Summary

This chapter covered basic network management including NetworkManager configuration, IPv4 and IPv6 addressing, hostname and DNS configuration, firewall management with firewalld, and network troubleshooting. You learned how to use nmcli and nmtui for NetworkManager, configure static and DHCP networks, set up firewall rules, and diagnose common network issues.

These skills are essential for managing network connectivity on RHEL systems. Understanding NetworkManager, firewalld, and troubleshooting techniques helps administrators configure and maintain reliable network connections.

---

## Quick Reference

### NetworkManager Commands

```bash
nmcli status                   # View status
nmcli connection show          # List connections
nmcli device status            # List devices
nmcli connection add type ethernet ifname ens192 con-name MyConn autoconnect yes
nmcli connection modify MyConn ipv4.addresses 192.168.1.10/24
nmcli connection modify MyConn ipv4.method manual
nmcli connection up MyConn
nmcli connection down MyConn
nmcli connection delete MyConn
```

### IPv4 Configuration

```bash
# Static
nmcli connection modify "conn" ipv4.addresses 192.168.1.10/24
nmcli connection modify "conn" ipv4.gateway 192.168.1.1
nmcli connection modify "conn" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify "conn" ipv4.method manual

# DHCP
nmcli connection modify "conn" ipv4.method auto
```

### IPv6 Configuration

```bash
# Static
nmcli connection modify "conn" ipv6.addresses 2001:db8::10/64
nmcli connection modify "conn" ipv6.method manual

# SLAAC
nmcli connection modify "conn" ipv6.method slaa

# Disabled
nmcli connection modify "conn" ipv6.method disabled
```

### Hostname and DNS

```bash
hostnamectl set-hostname myserver.example.com
nmcli connection modify "conn" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify "conn" ipv4.dns-search "example.com"
```

### Firewall Commands

```bash
firewall-cmd --state                  # Check status
firewall-cmd --get-zones              # List zones
firewall-cmd --zone=public --list-all # List zone
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --remove-service=http
firewall-cmd --reload
```

### Troubleshooting Commands

```bash
ip addr show                          # Check IP
ip route show                         # Check routing
ping -c 4 example.com                 # Test connectivity
nslookup example.com                  # Test DNS
nmcli device status                   # Check devices
systemctl status NetworkManager       # Check NM
journalctl -u NetworkManager -f       # Check logs
```

---

## Review Questions

1. What is the difference between NetworkManager and traditional network configuration?

2. How do you create a new Ethernet connection using nmcli?

3. What command would you use to view the status of all network devices?

4. How do you configure a static IPv4 address using NetworkManager?

5. What is the purpose of the `autoconnect` option in NetworkManager?

6. How do you configure DNS servers using NetworkManager?

7. What command would you use to check the current firewall status?

8. How do you add a service to the firewall permanently?

9. What is the difference between `--permanent` and `--reload` in firewall-cmd?

10. How do you configure a bond interface with NetworkManager?

11. What command would you use to set the system hostname?

12. How do you verify that DNS resolution is working correctly?

13. What is the purpose of the network-online.target?

14. How do you restart NetworkManager?

15. What command would you use to list all available firewall zones?

---

## Answers

1. NetworkManager is a modern network management daemon that automates network configuration, supports multiple connection types, and integrates with systemd. Traditional network configuration uses ifcfg files in /etc/sysconfig/network-scripts/ which is deprecated in RHEL 8+.

2. Use `nmcli connection add type ethernet ifname ens192 con-name MyEthernet autoconnect yes` to create a new Ethernet connection. Replace `ens192` with the actual interface name.

3. Use `nmcli device status` to view the status of all network devices. This shows each interface, its state (connected, unmanaged, disconnected), and the connection profile.

4. Use `nmcli connection modify "connection-name" ipv4.addresses 192.168.1.10/24` followed by `nmcli connection modify "connection-name" ipv4.method manual` to set a static IPv4 address.

5. The `autoconnect` option determines whether the connection should be automatically activated when the system boots or when the interface becomes available. Setting it to `yes` enables autoconnect.

6. Use `nmcli connection modify "connection-name" ipv4.dns "8.8.8.8 8.8.4.4"` to configure DNS servers. Multiple DNS servers can be specified separated by semicolons.

7. Use `firewall-cmd --state` to check the current firewall status. It returns "running" if the firewall is active, or "not running" if inactive.

8. Use `firewall-cmd --permanent --add-service=http` to add a service to the firewall permanently. The `--permanent` flag ensures the rule persists across reboots.

9. `--permanent` adds the rule to the configuration files so it persists across reboots. `--reload` applies the changes from the configuration files without rebooting. You typically use both: `--permanent` to add and `--reload` to apply.

10. Use `nmcli connection add type bond ifname bond0 con-name MyBond` to create a bond, then add slave connections with `type bond-slave ifname ens192 master bond0`. Configure bond options with `bond.options`.

11. Use `hostnamectl set-hostname myserver.example.com` to set the system hostname. This updates /etc/hostname and the systemd hostname.

12. Use `nslookup example.com` or `dig example.com` to test DNS resolution. You can also use `ping -c 4 example.com` to test both DNS and network connectivity.

13. The network-online.target is a systemd target that indicates the network is fully online and ready. Services can depend on this target to ensure they start only after the network is ready.

14. Use `systemctl restart NetworkManager` to restart the NetworkManager service. This will re-scan interfaces and apply active connections.

15. Use `firewall-cmd --get-all-zones` to list all available firewall zones. This shows all defined zones including public, home, work, internal, external, drop, and reject.
