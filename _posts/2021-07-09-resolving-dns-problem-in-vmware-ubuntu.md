---
layout: post
title:  "Resolving DNS problem in Ubuntu running in a vm"
---

i have an Ubuntu 18.04.5 LTS running inside a VMWare Workstation 15 player in bridged mode. i could ping with an ip address (ipv4/ipv6) but not hostname.
`cat`ing `/etc/resolv.conf` showed this:
&nbsp;
```
nameserver 127.0.0.53
options edns0
```
&nbsp;
after changing the ip to my router's ip - `192.168.1.254` - and restarting network-manager using `sudo service network-manager restart`, i could finally ping the hostnames too. 
