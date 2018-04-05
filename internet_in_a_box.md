# Networking Interfaces

## Bridge Configuration:  /etc/sysconfig/network-scripts/ifcfg-<Bridge_Name_Here>

## Ethernet Configuration (static):  /etc/sysconfig/network-scripts/ifcfg-<Interface_Name_Here>
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

# DHCP

Install DHCPD
``` yum install dhcpd ```
## DHCPD Configuration:  /etc/dhcp/dhpcd.conf

# DNS

Install BIND 9
``` yum install named ```
## NameD Configuration: /etc/named.conf

## Zone Files/var/named

These two files must and the directory must be owned by named

```
$ touch /var/named/dynamic/<Host_Name.Domain_Name>.fw
$ touch /var/named/dynamic/<Host_Name.Domain_Name>.rev
$ chown named:named /var/named/dynamic
```

### Forward Zone: /var/named/dynamic/<Host_Name.Domain_Name>.fw

### Reverse Zone: /var/named/dynamic/<Host_Name.Domain_Name>.rev

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
