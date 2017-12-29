---
layout: post
title:  "PCE/macplus with socat"
date:   2017-11-01 00:00:00 -0500
categories: emu mac
---

I was able to compile tun support for [pce-macplus](http://www.hampa.ch/pce/pce-macplus.html) Mac OS X 10.11, at least on the surface, after a couple of hours of research. Granted, I haven't made an Internet connection yet, but I'm hoping that this could be because of my inexperience with tuntap on OS X. I wrote some quick notes in hopes that someone could make use of them and get some actual results.

The [original guide](http://www.toughdev.com/content/2016/11/pcemacplus-the-ultimate-68k-classic-macintosh-emulator/) gave me another idea after I re-read it and I actually had a little bit more success with that. Although [tty0tty](https://github.com/freemed/tty0tty) relies on a Linux kernel module, another utility called "[socat](http://www.dest-unreach.org/socat/doc/socat.html)" runs on a wider range of platforms and is [available through homebrew](http://brewformulas.org/Socat) for OS X.

```
iMac:~ matt$ socat -d -d pty,raw,ignoreeof,echo=0,link=/tmp/modem1 pty,raw,ignoreeof,echo=0,link=/tmp/modem2  
2017/11/22 18:56:55 socat[1708] N PTY is /dev/ttys007  
2017/11/22 18:56:55 socat[1708] N PTY is /dev/ttys008  
```

The above basically creates a fake null modem cable as originally suggested. The "link" parameter allowed me to maintain some degree of consistency between calling socat; the ttys are allocated numbers at random, but you can link a filename to put in your .cfg.

I chose `/tmp` instead of `/dev` because even after disabling SIP on the Mac, I couldn't easily write to `/dev`. On a side note, you'll have to disable SIP eventually to write the pppd options in `/etc` because apparently `/etc/ppp/options` is hardcoded into pppd.

Here's the snippet of my config when I configure the serial port:

```
serial {  
        port = 0

        multichar = 1

        driver = "stdio:file=/tmp/modem1"
        #driver = "ppp:if=tap0:host-ip=10.10.10.2:guest-ip=10.10.10.3"
}
```

In true form, I got ahead of myself and immediately tried to fire up FreePPP on Mac TCP. I ran pppd as described on the guide with the exception of the device name which is the second generated device from above, `/dev/ttys008`. pppd gets angry if you specify a "device" file outside of `/dev`, so that's necessary.

You can use the following commands in separate terminals to monitor the two sides of the "cable:"

```
cat < /tmp/modem1  
cat < /tmp/modem2  
```

After I had introduced way too many variable into my "equation," I stepped back and tried [ZTerm](http://www.dalverson.com/zterm/) on System 7 and [minicom on Mac OS X](http://brewformulas.org/Minicom) through the "cable".

I chose minicom because it's much more agnostic about allowing non-standard device names compared to something more GUI friendly and it's also available on homebrew. I chose the modem port on System 7, which the .cfg pointed to `/tmp/modem1` and `/tmp/modem2` in minicom.

When I typed on the emulator I could see appropriate characters on minicom, but the reverse wasn't true (I saw nothing on the Mac Plus). I had the cat commands running in two other terminals and they produced output as expected for the corresponding cable ends.

Hopefully, someone can take this and run with it to make something usable. I hope it helps.

Update
===

I guess some time away from this little exercise was helpful. When I came back, I rethought my config file and wondered why I was using stdio. I guess it was just the first thing I tried and I stuck with it for no apparent reason.

I ended up substituting this config line for the serial section:

```
driver = "posix:file=/tmp/modem1"
```

This allowed me to invoke a comm terminal on the "Plus" and communicate as expected. I haven't gotten pppd quite working yet. They begin to communicate and the Plus / FreePPP gives me a success message, but the connection times out before it's fully established.

I'm thinking / hoping that there are some nuances between the original guide's Linux pppd and OS X as well as the fact that I'm using a different approach. The following is some console output from pppd:

```
Thu Nov 23 23:45:05 2017 : ipcp: returning Configure-ACK
Thu Nov 23 23:45:05 2017 : sent [IPCP ConfAck id=0xc <compress VJ 0f 01> <addr 192.168.2.245>]
Thu Nov 23 23:45:08 2017 : sent [IPCP ConfReq id=0x1 <compress VJ 0f 01> <addr 192.168.2.244>]
Thu Nov 23 23:45:08 2017 : rcvd [IPCP ConfReq id=0xd <compress VJ 0f 01> <addr 192.168.2.245>]
Thu Nov 23 23:45:08 2017 : ipcp: returning Configure-ACK
Thu Nov 23 23:45:08 2017 : sent [IPCP ConfAck id=0xd <compress VJ 0f 01> <addr 192.168.2.245>]
Thu Nov 23 23:45:11 2017 : IPCP: timeout sending Config-Requests
Thu Nov 23 23:45:11 2017 : sent [LCP TermReq id=0x2 "No network protocols running"]
Thu Nov 23 23:45:11 2017 : Connection terminated.
```

If anyone would care to try, here are the steps, in order so far:

 - call socat as described above
 - note the second ttys device's number
 - start pce
 - initiate the ppp connection on the emulated Mac
 - initiate a yet to be determined pppd command on the OS X host that includes the tty generated from the socat output

The last two items could be interchangeable or I might have them backwards, but the "fake success" message was kind of neat to see, even it was a lie.
