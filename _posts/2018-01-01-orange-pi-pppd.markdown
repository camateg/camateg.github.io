---
layout: post
title:  "OrangePi MacIPGW pppd"
date:   2018-01-01 00:00:00 -0500
categories: mac linux
---

 * with the TTL to RS232 converter, use the ground and VCC on the 13 pin header (pin 1 and 2) and only the TX/RX on the 3 pin header.  Otherwise, my RS232 board got hot...
 * set `console=display` in `/boot/armbianEnv.txt` to disable the Armbian serial console...
 * `apt-get install ppp`
 * create `/etc/ppp/options.ttyS0`:

```
  noauth
  asyncmap 0
  nocrtscts
  local
  lock
  netmask 255.255.255.0
  passive
  proxyarp
  lcp-echo-interval 30
  lcp-echo-failure 4
  noipx
  persist
```

 * create `/etc/systemd/system/pppd-ttyS0.service`, possibly making the ExecStop more specific:

```
[Unit]
Description=pppd-ttyS0.service

[Service]
Type=forking
ExecStart=/usr/sbin/pppd 172.16.3.1:172.16.3.2 /dev/ttyS0 57600
ExecStop=/usr/bin/killall pppd

[Install]
WantedBy=multi-user.target
``` 

 * disable any serial-getty services?
 * `systemctl enable pppd-ttyS0.servicce` 
