# Chapter 8: Managing Basic Networking

## Objectives

After this chapter, you should be able to:

- distinguish a network device from a NetworkManager connection profile;
- configure persistent IPv4 and IPv6 addressing;
- configure gateways, DNS servers, and search domains;
- configure hostnames and local name resolution;
- ensure networking and network services start at boot;
- assign a connection to a firewalld zone;
- allow or restrict access with firewalld; and
- troubleshoot from the local link outward.

## 1. The NetworkManager model

A **device** is an interface such as `enp1s0`. A **connection profile** is saved configuration that can be activated on a compatible device. A device can have several profiles, but normally only one is active at a time.

Inspect:

~~~bash
$ nmcli general status
$ nmcli device status
$ nmcli connection show
$ nmcli connection show --active
$ nmcli device show enp1s0
$ ip -brief address
$ ip route
$ ip -6 route
~~~

Do not assume the profile name equals the interface name:

~~~bash
$ nmcli -f NAME,UUID,TYPE,DEVICE,AUTOCONNECT connection show
~~~

NetworkManager stores persistent profiles as keyfiles below `/etc/NetworkManager/system-connections/`. Use `nmcli` or `nmtui` rather than hand-editing unless a task specifically requires file work.

## 2. Gather the required values first

For a static profile, write down:

- interface name;
- profile name;
- IPv4 address and prefix;
- IPv4 gateway, if any;
- IPv6 address and prefix;
- IPv6 gateway, if any;
- DNS server addresses;
- DNS search domain;
- whether this connection should supply a default route;
- firewalld zone.

An address without its prefix is incomplete. `192.0.2.10/24` and `192.0.2.10/27` belong to different networks.

## 3. Create a static dual-stack profile

~~~bash
# nmcli connection add \
    type ethernet \
    ifname enp1s0 \
    con-name server-lan \
    ipv4.method manual \
    ipv4.addresses 192.0.2.10/24 \
    ipv4.gateway 192.0.2.1 \
    ipv4.dns "192.0.2.53 192.0.2.54" \
    ipv4.dns-search example.com \
    ipv6.method manual \
    ipv6.addresses 2001:db8:1::10/64 \
    ipv6.gateway 2001:db8:1::1 \
    connection.autoconnect yes
~~~

Activate and verify:

~~~bash
# nmcli connection up server-lan
$ nmcli connection show server-lan
$ ip -brief address show enp1s0
$ ip route
$ ip -6 route
$ cat /etc/resolv.conf
~~~

If working over SSH, a wrong activation can disconnect you. Keep console access or a second verified management path.

## 4. Modify an existing profile

~~~bash
# nmcli connection modify server-lan \
    ipv4.method manual \
    ipv4.addresses 192.0.2.20/24 \
    ipv4.gateway 192.0.2.1 \
    ipv4.dns "192.0.2.53 192.0.2.54" \
    ipv4.ignore-auto-dns yes \
    connection.autoconnect yes
# nmcli connection up server-lan
~~~

Replace an address list:

~~~bash
# nmcli connection modify server-lan ipv4.addresses 192.0.2.20/24
~~~

Add one without replacing:

~~~bash
# nmcli connection modify server-lan +ipv4.addresses 192.0.2.21/24
~~~

Remove a particular value:

~~~bash
# nmcli connection modify server-lan -ipv4.addresses 192.0.2.21/24
~~~

The `+` and `-` prefixes matter for multivalue properties.

## 5. Dynamic addressing

IPv4 DHCP:

~~~bash
# nmcli connection modify server-lan \
    ipv4.method auto \
    ipv4.addresses "" \
    ipv4.gateway "" \
    ipv4.ignore-auto-dns no
# nmcli connection up server-lan
~~~

IPv6 automatic configuration:

~~~bash
# nmcli connection modify server-lan ipv6.method auto
# nmcli connection up server-lan
~~~

Disable an IP family only when required:

~~~bash
# nmcli connection modify server-lan ipv6.method disabled
~~~

Do not disable IPv6 merely because an IPv4-only task was assigned. The objective requires the ability to configure both families, and other services can depend on IPv6.

## 6. Connections without a default route

A storage-only or backup network should not replace the main default route:

~~~bash
# nmcli connection modify storage-net \
    ipv4.method manual \
    ipv4.addresses 198.51.100.10/24 \
    ipv4.never-default yes \
    ipv6.never-default yes
~~~

Verify:

~~~bash
$ ip route
$ ip -6 route
~~~

For an explicit static route:

~~~bash
# nmcli connection modify server-lan \
    +ipv4.routes "198.51.100.0/24 192.0.2.254"
# nmcli connection up server-lan
$ ip route
~~~

## 7. Hostname and name resolution

Set a persistent hostname:

~~~bash
# hostnamectl set-hostname servera.example.com
$ hostnamectl
$ hostname
~~~

Local static entries in `/etc/hosts` use:

~~~text
192.0.2.10 servera.example.com servera
2001:db8:1::10 servera.example.com servera
~~~

Put the canonical FQDN before aliases.

Test the same resolver path applications use:

~~~bash
$ getent hosts servera.example.com
$ getent ahosts servera.example.com
~~~

Use `dig` when the `bind-utils` package is available to query DNS specifically:

~~~bash
$ dig servera.example.com
$ dig -x 192.0.2.10
~~~

Do not make persistent DNS changes by editing `/etc/resolv.conf`; NetworkManager normally manages it. Configure DNS in the active profile:

~~~bash
# nmcli connection modify server-lan \
    ipv4.dns "192.0.2.53 192.0.2.54" \
    ipv4.dns-search example.com \
    ipv4.ignore-auto-dns yes
# nmcli connection up server-lan
~~~

## 8. Network services at boot

Ensure NetworkManager itself:

~~~bash
# systemctl enable --now NetworkManager
$ systemctl is-enabled NetworkManager
$ systemctl is-active NetworkManager
~~~

Ensure a profile reconnects:

~~~bash
# nmcli connection modify server-lan connection.autoconnect yes
$ nmcli -f NAME,AUTOCONNECT,DEVICE connection show
~~~

Ensure a network-facing service starts:

~~~bash
# systemctl enable --now sshd
$ systemctl is-enabled sshd
$ systemctl is-active sshd
$ ss -lntp
~~~

An enabled daemon is not necessarily reachable. It must also be active, listening on the expected address and port, allowed by firewalld, and allowed by SELinux.

## 9. firewalld concepts

firewalld assigns interfaces or source networks to zones. A zone defines the allowed inbound services and ports for that trust level.

Two configuration layers:

- **runtime** is active now and is lost on reload or reboot;
- **permanent** survives reload and reboot but must be loaded before it affects runtime.

Inspect before changing:

~~~bash
$ firewall-cmd --state
$ firewall-cmd --get-default-zone
$ firewall-cmd --get-active-zones
$ firewall-cmd --zone=public --list-all
$ firewall-cmd --get-services
$ firewall-cmd --info-service=http
~~~

## 10. Allow services and ports

Allow a predefined service persistently:

~~~bash
# firewall-cmd --permanent --zone=public --add-service=http
# firewall-cmd --reload
$ firewall-cmd --zone=public --query-service=http
~~~

Allow a custom port:

~~~bash
# firewall-cmd --permanent --zone=public --add-port=8080/tcp
# firewall-cmd --reload
$ firewall-cmd --zone=public --query-port=8080/tcp
~~~

Remove:

~~~bash
# firewall-cmd --permanent --zone=public --remove-service=http
# firewall-cmd --permanent --zone=public --remove-port=8080/tcp
# firewall-cmd --reload
~~~

Prefer a service definition when one exists; it documents intent and can include multiple required ports.

### Runtime testing before permanent commit

~~~bash
# firewall-cmd --zone=public --add-service=http
$ firewall-cmd --zone=public --query-service=http
~~~

If the test is correct, make it permanent separately:

~~~bash
# firewall-cmd --permanent --zone=public --add-service=http
~~~

Do not use `--runtime-to-permanent` casually; it copies every current runtime change, including unrelated temporary testing.

## 11. Assign a connection to a zone

Persist the zone in the NetworkManager profile:

~~~bash
# nmcli connection modify server-lan connection.zone public
# nmcli connection up server-lan
$ firewall-cmd --get-active-zones
$ nmcli -f connection.id,connection.zone connection show server-lan
~~~

This is more stable than assigning only a transient interface after activation.

## 12. Restrict access by source

Example: permit SSH only from `192.0.2.0/24` in the public zone.

First ensure console or another management route, then remove the general allowance and add a specific rich rule:

~~~bash
# firewall-cmd --permanent --zone=public --remove-service=ssh
# firewall-cmd --permanent --zone=public \
    --add-rich-rule='rule family="ipv4" source address="192.0.2.0/24" service name="ssh" accept'
# firewall-cmd --reload
$ firewall-cmd --zone=public --list-all
~~~

Do not perform this remotely without a recovery path. Test from an allowed source before ending the current session.

To remove that exact rule, repeat the exact text with `--remove-rich-rule`.

An alternative design is a separate zone bound to a source network:

~~~bash
# firewall-cmd --permanent --zone=internal --add-source=192.0.2.0/24
# firewall-cmd --permanent --zone=internal --add-service=ssh
# firewall-cmd --reload
~~~

Use the design specified by the task and verify which zone actually handles the traffic.

## 13. Troubleshoot from local to remote

Follow the packet path:

### 1. Device and profile

~~~bash
$ nmcli device status
$ nmcli connection show --active
$ ip -brief link
~~~

### 2. Addresses

~~~bash
$ ip -brief address
$ nmcli device show enp1s0
~~~

### 3. Routes

~~~bash
$ ip route get 192.0.2.53
$ ip -6 route get 2001:db8:1::53
~~~

### 4. Local gateway

~~~bash
$ ping -c 3 192.0.2.1
$ ping -6 -c 3 2001:db8:1::1
~~~

### 5. Name resolution

~~~bash
$ getent hosts target.example.com
$ resolvectl status
~~~

### 6. Service listener

~~~bash
$ ss -lntup
$ systemctl status SERVICE
~~~

### 7. Firewall and SELinux

~~~bash
$ firewall-cmd --get-active-zones
$ firewall-cmd --zone=public --list-all
$ getenforce
$ ausearch -m AVC -ts recent
~~~

### 8. Logs

~~~bash
$ journalctl -b -u NetworkManager
$ journalctl -b -u firewalld
$ journalctl -xeu SERVICE
~~~

Do not start with “restart everything.” Inspection tells you which layer is wrong.

## 14. Real-life example: configure a web server interface

Requirement: static dual-stack addressing, DNS, automatic connection, public zone, and HTTP/HTTPS access.

~~~bash
# nmcli connection add type ethernet ifname enp1s0 con-name web-lan \
    ipv4.method manual ipv4.addresses 192.0.2.20/24 \
    ipv4.gateway 192.0.2.1 ipv4.dns "192.0.2.53" \
    ipv6.method manual ipv6.addresses 2001:db8:1::20/64 \
    ipv6.gateway 2001:db8:1::1 connection.autoconnect yes
# nmcli connection modify web-lan connection.zone public
# nmcli connection up web-lan
# firewall-cmd --permanent --zone=public --add-service=http
# firewall-cmd --permanent --zone=public --add-service=https
# firewall-cmd --reload
$ ip -brief address show enp1s0
$ ip route
$ ip -6 route
$ firewall-cmd --zone=public --list-all
~~~

## 15. Exam lab

Using values supplied by your lab:

1. Configure a persistent static IPv4 and IPv6 profile.
2. Configure the supplied gateways, DNS, and search domain.
3. Set the requested FQDN and local host mapping.
4. Ensure the profile and `sshd` start at boot.
5. Bind the profile to the requested zone.
6. Permit SSH and one requested custom TCP port.

Solution pattern:

~~~bash
# nmcli connection add type ethernet ifname INTERFACE con-name exam-net \
    ipv4.method manual ipv4.addresses IPV4/PREFIX ipv4.gateway IPV4_GATEWAY \
    ipv4.dns "DNS1 DNS2" ipv4.dns-search DOMAIN \
    ipv6.method manual ipv6.addresses IPV6/PREFIX ipv6.gateway IPV6_GATEWAY \
    connection.autoconnect yes
# nmcli connection modify exam-net connection.zone ZONE
# nmcli connection up exam-net
# hostnamectl set-hostname FQDN
# vim /etc/hosts
# systemctl enable --now NetworkManager sshd firewalld
# firewall-cmd --permanent --zone=ZONE --add-service=ssh
# firewall-cmd --permanent --zone=ZONE --add-port=PORT/tcp
# firewall-cmd --reload
~~~

Verify:

~~~bash
$ nmcli -f NAME,DEVICE,AUTOCONNECT connection show
$ ip -brief address
$ ip route
$ ip -6 route
$ getent hosts FQDN
$ systemctl is-enabled sshd
$ ss -lntp
$ firewall-cmd --get-active-zones
$ firewall-cmd --zone=ZONE --list-all
~~~

## Quick check

- Can you distinguish a device from a connection profile?
- Can you replace, add, and remove an address without confusing nmcli list properties?
- Can you configure a connection that must not supply a default route?
- Can you configure DNS persistently without editing `/etc/resolv.conf`?
- Can you make a profile autoconnect?
- Can you explain runtime versus permanent firewalld state?
- Can you prove which zone handles an interface?
- Can you troubleshoot address, route, DNS, listener, firewall, and SELinux separately?

## Official references

- [RHEL 10 networking guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_networking/index)
- [Configuring an Ethernet connection](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_networking/configuring-an-ethernet-connection)
- Local pages: `man nmcli`, `man nm-settings`, `man ip`, `man firewalld`, and `man firewall-cmd`

