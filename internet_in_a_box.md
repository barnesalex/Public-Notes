# Networking Interfaces

## Bridge Configuration:  /etc/sysconfig/network-scripts/ifcfg-<Bridge_Name_Here>
```
DEVICE="<Bridge Name Here>"
TYPE=BRIDGE
ONBOOT=yes
BOOTPROTO=static
IPADDR="<IP you want for the NAT>"
PREFIX=24
ZONE=internal
NM_CONTROLLED=no
NOZEROCONF=yes

```

## Ethernet Configuration (static):  /etc/sysconfig/network-scripts/ifcfg-<Interface_Name_Here>

```
TYPE="Ethernet"
NAME="<Interface_Name_Here>"
DEVICE="<Interface_Name_Here>"
ONBOOT="yes"
UUID=
ZONE=external
DNS1="<>"
DNS2="<>"
GATEWAY=<>
PREFIX=<>
IPADDR=<>
NM_CONTROLLED=no
NOZEROCONF=yes
```
Make sure Network Manager isn't messing up things
```
systemctl disable NetworkManager
systemctl stop NetworkManger
```
And set SELinux to permissive for the configuration unitl I understand the correct configs
```
setenforce 0

```
And for other ports you want controlled by the bridge
## Other Ethernet Configuration /etc/sysconfig/network-scripts/ifcfg-<Interface_Name_Here>
```
DEVICE=<Interface_Name_Here>
TYPE=Ethernet
BRIDGE=<Bridge Name Here>
BOOTPROT=dhcp
ONBOOT=yes
NM_CONTROLLED=no
```

# DHCP

Install DHCPD
``` yum install dhcpd ```
## DHCPD Configuration:  /etc/dhcp/dhpcd.conf

```
allow bootp;
allow booting;
option domain-name-servers 172.18.0.1;

default-lease-time 1800;
max-lease-time 3600;

ddns-update-style interim;
ddns-updates           on;
update-static-leases   on;

authoritative;
log-facility local7;

### HostName.DomainName
subnet 172.18.0.0 netmask 255.255.255.0 {
  range 172.18.0.100 172.18.0.254;      #Range of IP addresses handed out
  option routers 172.18.0.1;            #IP that hosts the DHCP Server
  option domain-name "Host_Name.Domain_Name";
  option ntp-servers 129.6.15.30, 128.206.4.128, 131.151.2.65, 134.124.1.200, 134.193.44.201;
  option time-offset -21600;

  next-server 172.18.0.1;               #Location of tftp scripts
  filename "pxelinux.0";                #This translates for the BIOS, the contents of the TFTP folder and it's directories

  zone Host_Name.Domain_Name. { primary 172.18.0.1; } #Tells the client the location of the server and what zone
  zone 0.18.172.in-addr.arpa. { primary 172.18.0.1; } #Reverse of the above
  }
```



# DNS

Install BIND 9
``` yum install named ```
## NameD Configuration: /etc/named.conf
```
options {
        listen-on port 53 { 127.0.0.1; 172.18.0.1; };
        //listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { localhost; 172.18.0.0/24; };
        recursion yes;  #read later

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "Host_Name.Domain_Name" IN {
    type master;
    file "dynamic/Host_Name.Domain_Name.fw";
    allow-update { 127.0.0.1; 172.18.0.0/24; };
};

zone "0.18.172.in-addr.arpa" IN {
    type master;
    file "dynamic/Host_Name.Domain_Name.rev";
    allow-update { 127.0.0.1; 172.18.0.0/24; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

## Zone Files/var/named

These two files must and the directory must be owned by named

```
$ touch /var/named/dynamic/<Host_Name.Domain_Name>.fw
$ touch /var/named/dynamic/<Host_Name.Domain_Name>.rev
$ chown named:named /var/named/dynamic
```

### Forward Zone: /var/named/dynamic/<Host_Name.Domain_Name>.fw
```
$ORIGIN .
$TTL 3600       ; 1 hour
Host_Name.Domain_Name   IN SOA  ns1.Host_Name.Domain_Name. root.Host_Name.Domain_Name. (
                                2018010211 ; serial
                                3600       ; refresh (1 hour)
                                1800       ; retry (30 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS      Host_Name.Domain_Name.
                        A       172.18.0.1
$ORIGIN Host_Name.Domain_Name.
$TTL 1800       ; 30 minutes
ns1                       A     172.18.0.1

```

### Reverse Zone: /var/named/dynamic/<Host_Name.Domain_Name>.rev
```
$ORIGIN .
$TTL 3600       ; 1 hour
0.18.172.in-addr.arpa   IN SOA  ns1.Host_Name.Domain_Name. root.Host_Name.Domain_Name. (
                                2018010211 ; serial
                                3600       ; refresh (1 hour)
                                1800       ; retry (30 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS      ns1.Host_Name.Domain_Name.
                        A       172.18.0.1
                        PTR     Host_Name.Domain_Name.
$ORIGIN 0.18.172.in-addr.arpa.
1                       PTR     ns1.Host_Name.Domain_Name.
```

# Firewall Configuration (If on Centos)

## Commands

Your interfaces need to be in the correct zones.  Put your WAN port in external and your bridge/other interfaces into internal

``` 
firewall-cmd --permanent --zone=external --change-interface=eth0 
firewall-cmd --permanent --zone=internal --change-interface=br0
firewall-cmd --permanent --zone=internal --change-interface=<other interfaces> 
```
Now you need to make sure you have the proper services enabled (dns, dhcp, and tftp).
``` 
firewall-cmd --permanent --zone=internal --add-service=dns 
firewall-cmd --permanent --zone=internal --add-service=dhcp
firewall-cmd --permanent --zone=internal --add-service=tftp
```

## Internal Zone
```
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces: br1
  sources: 
  services: ssh dns dhcp tftp
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

## External Zone

```
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: em1
  sources: 
  services: ssh
  ports: 
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```
  
# TFTP
Install tftp-server and syslinux (and wget if you do not have it)
``` yum install tftp-server syslinux wget ```

wget the initrd.img and vmlinuz in RHEL based OSes

Make directories to store these files per release and make menu entries for each

## TFTP Configuration: /var/lib/tftpboot/pxelinux.cfg/default
Each entry needs to have a file or directory in /var/lib/tftpboot/ for the config to utilize
```
DEFAULT menu.c32
TIMEOUT 100
PROMPT 1

MENU TITLE <Type whatever you want to be the menu title>
LABEL local
MENU LABEL Local Boot
  localboot 0

LABEL memtest
  MENU LABEL memtest
  KERNEL memtest

LABEL fedora27
  MENU LABEL Fedora_27
  KERNEL fedora27/vmlinuz
  APPEND initrd=fedora27/initrd.img ip=dhcp inst.stage2=http://mirror.rnet.missouri.edu/fedora/linux/releases/27/Server/x86_64/os/

LABEL centos7
  MENU LABEL CentOS_7.4.1708
  KERNEL centos7/vmlinuz
  APPEND initrd=centos7/initrd.img ip=dhcp inst.repo=http://mirror.rnet.missouri.edu/centos/7.4.1708/os/x86_64/
```
