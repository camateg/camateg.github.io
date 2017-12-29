---
layout: post
title:  "PCE/macplus and tuntap"
date:   2017-10-31 00:00:00 -0500
categories: emu mac
---

I just found a 68K Mac emulator that I hadn't heard of before called [pce-macplus](http://www.hampa.ch/pce/pce-macplus.html). It appealed to me because vMac doesn't support serial interfaces or networking and also because Basilisk II doesn't seem to be as actively maintained as it once was and I have some stability issues on OS X with it.

The source code is very Linux-centric and not especially well documented, but I found [an excellent blog post](http://www.toughdev.com/content/2016/11/pcemacplus-the-ultimate-68k-classic-macintosh-emulator/) which helped immensely with initial compilation and initial configuration, so thanks for that.

As I said, I was particularly interested in network support, which doesn't seem to have been successfully accomplished on Mac OS X yet for pce, so naturally I had to try. Admittedly, I don't have much porting or compiled language development since the early 2000s, but hopefully some of it will come back.

I've been able to compile tun support (on the surface) into pce-macplus on OS X, but haven't successfully gotten the ppp driver to activate or made a connection. I hope that posting this will help to help propel someone more capable in that direction.

Networking in pce-macplus is accomplished through a tun/tap interface. Mac OS X supports a sort of tun (but no tap), but is likely very different programatically. The source code for pce is clearly written with Linux in mind (i.e. direct references to the Linux kernel source with no error checking), so I figured I'd try to find something more universal in terms of handling tun:

`src/lib/tun.c:`

```
    ...
    #include <linux/if.h>
    #include <linux/if_tun.h>
    ...
```

There is / was a tuntap package for OS X, tuntaposx, but I wasn't sure if it was still actively maintained or how dependencies would be handled, so I ended up using a Homebrew repackaging of tuntap.

The tuntap package in particular had be relegated to Homebrew-cask so I used the following commands related to homebrew:
Install homebrew (you probably already have):

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Install homebrew-cask:

```
brew tap caskroom/cask  
```

Install dependencies:

```
brew install sdl libpcap  
brew cask install tuntap  
```

We need SDL for the UI because XQuartz is slow and clunky, libpcap (I think; I have to verify that this is necessary) and tuntap.

The next issue that most people have encountered is hardcoded reference to the Linux kernel sources, such as `src/lib/tun.c`:

```
    ...
    #include <linux/if.h>
    #include <linux/if_tun.h>
    ...
```

This obviously doesn't apply to everyone, particularly to me on a Mac. I ended up putting the following early in `tun.c`, more details to come on that later:

```
#ifdef __APPLE__
    #ifdef TARGET_OS_MAC
        #include <net/if.h>
        #include <net/if_types.h>
        #include <net/bpf.h>

        #define TUNSETIFF BIOCSETIF
        #define IFF_TAP IFT_PKTAP
        #define IFF_TUN APPLE_IF_FAM_TUN
        #define IFF_NO_PI 0x1000
    #endif
#else
    #include <linux/if.h>
    #include <linux/if_tun.h>
#endif
```

Basically, this tries to account for the possibility of being on a Mac / Darwin system without breaking the initial intent of including `linux/if.h` and `linux/if_tun.h`.

My reference to `__APPLE__` appears to be archaic, but it gets the job done temporarily. If you'd like to improve this, [this might be a good reference](https://developer.apple.com/library/content/documentation/Porting/Conceptual/PortingUnix/compiling/compiling.html).

The `#define` statements are a part that I'm not necessarily sure about. I did my best to determine a one-to-one relationship between Linux and *BSD macros related to tunneling and packet filtering. Take them with a grain of salt. There may also be stray references to hardcoded Linux device names in the pce source code, but I haven't gotten that far yet.

My configure command ended up being pretty simple, which was a pleasant surprise. Initially, I specified some additional CFLAGS and LDFLAGS because of the third-party libpcap, but I left them out by accident and things worked anyways. This is what led me to question the need for installing libpcap, but it's late and I'll confirm later. Clearly something made me want to install libpcap?

These are the compiler flags I had added that didn't seem to matter. They will be empty by default. I actually noticed that additional carriage returns might have been introduced into generated Makefile.inc when I specified these in the configure command, so these are just left as a reference:

```
export CFLAGS= -I/usr/local/opt/libpcap/include  
export LDFLAGS= -L/usr/local/opt/libpcap/lib  
```

Configure your build parameters: 

```
./configure --with-sdl -enable-char-ppp --enable-tun -enable-char-tcp --enable-char-slip --enable-char-pty --enable-char-posix --enable-char-termios
```

This results in the following configuration: 

```
pce 20171112-4c256b0-mod is now configured:  
  CC:                      gcc -g -O2
  LD:                      gcc 

  prefix:                  /usr/local

  Emulators built:         atarist cpm80 ibmpc macplus rc759 sim405 sim6502 simarm sims32
  Emulators not built:    
  ROMs built:             
  ROMs not built:          ibmpc macplus
  Terminals built:         null x11 sdl
  Terminals not built:    
  Char drivers built:      null stdio posix ppp pty slip tcp termios
  Char drivers not built:  wincom
  Sound drivers built:     null wav sdl
  Sound drivers not built: oss
  Enabled options:         tun
  Disabled options:        readline
```

I did a make && make install and fired up a machine. Here is the config file I used, many thanks to the author of the post above:

```
path = "-."


memtest = 0


system {  
    model="mac-plus"
}

cpu {  
    model = "68000"
}


ram {  
    address = 0
    size = 4096K
    default = 0x00
}

rom {  
    file = "plus.rom"
    address = 0x400000

    size = (system.model == "mac-classic") ? 512K : 256K

    default = 0xff
}


terminal {  
    driver = "sdl"

    scale = 1

    aspect_x = 3
    aspect_y = 2

    border = 0

    fullscreen = 0

    mouse_mul_x = 1
    mouse_div_x = 1
    mouse_mul_y = 1
    mouse_div_y = 1
}

sound {  
    lowpass = 8000

    driver = "sdl:wav=speaker.wav:lowpass=0:wavfilter=0"
}


keyboard {  
    model = 0
    intl  = 0

    keypad_motion = 0
}


adb {  
    mouse = true
    keyboard = true
    keypad_motion = false
}


rtc {  
    file = "pram-mac-classic.dat"
    realtime = 1
}


cfg.sony = 1  
sony {  
    insert_delay = 1
}

if (cfg.sony) {  
    rom {
        file = "macplus-pcex.rom"
        address = 0xf80000
        sizes = 256K
    }
}

scsi {  
    device {
        id = 0
        drive = 2
        vendor = " SEAGATE"
        product = "          ST225N"
    }
}


serial {  
    port = 0

    multichar = 1

    driver = "ppp:if=tun0:host-ip=10.10.10.2:guest-ip=10.10.10.3"
}

serial {  
    port = 1
    driver = "stdio:file=ser_b.out"
}


video {  
    color0 = 0x000000
    color1 = 0xffffff
    brightness = 1000
}


disk {  
    drive   = 2
    type    = "auto"
    file    = "hd1.img"
    optional = 0
}
```

I start the emulator with the command:

```
pce-macplus -v -c plus.cfg -r.
```

Here are the initial console messages. Note `*** can't open driver (ppp:if=tun0:host-ip=10.10.10.2:guest-ip=10.10.10.3)`

```
Copyright (C) 2007-2012 Hampa Hug <hampa@hampa.ch>  
CONFIG:   file="plus.cfg"  
SYSTEM:   model=mac-plus  
RAM:      addr=0x00000000 size=4194304 file=<none>  
ROM:      addr=0x00400000 size=262144 file=plus.rom  
ROM:      addr=0x00f80000 size=65536 file=macplus-pcex.rom  
RAM:      disabling memory test  
CPU:      model=68000 speed=0  
VIA:      addr=0xefe000 size=0x2000  
SCC:      addr=0x800000 size=0x400000  
SERIAL:   port=0 multichar=1 driver=ppp:if=tun0:host-ip=10.10.10.2:guest-ip=10.10.10.3  
*** can't open driver (ppp:if=tun0:host-ip=10.10.10.2:guest-ip=10.10.10.3)
SERIAL:   port=1 multichar=1 driver=stdio:file=ser_b.out  
RTC:      file=pram-mac-classic.dat realtime=1 start=<now> romdisk=0  
KEYBOARD: model=0 international=0 keypad=keypad  
DISK:     drive=2 type=auto blocks=409600 chs=406/16/63 rw file=hd1.img  
IWM:      addr=0xd00000  
SCSI:     addr=0x580000 size=0x80000  
SCSI:     id=0 drive=2 vendor=" SEAGATE" product="          ST225N"  
SONY:     drive=1 delay=1  
SONY:     drive=2 delay=1  
SONY:     drive=3 delay=1  
SONY:     drive=4 delay=1  
SOUND:    addr=0x3FFD00 lowpass=8000 driver=sdl:wav=speaker.wav:lowpass=0:wavfilter=0  
2017-11-22 01:41:54.029 pce-macplus[12614:133236] 01:41:54.028 WARNING:  140: This application, or a library it uses, is using the deprecated Carbon Component Manager for hosting Audio Units. Support for this will be removed in a future release. Also, this makes the host incompatible with version 3 audio units. Please transition to the API's in AudioComponent.h.  
TERM:     driver=sdl ESC=ESC aspect=3/2 min_size=512*384 scale=1 mouse=[1/1 1/1]  
VIDEO:    addr=0x3FA700 w=512 h=342 bright=100%  
[000000] mac: reset
[4000D2] main sound buffer
00400594: undefined operation: 4E7B [4E7B 0002 600C 7201 0C6F]  
00400594: exception 04 (ILLG) IW=4E7B  
SONY:     PCE ROM extension at 0xf8001c  
SONY:     sony driver at 0x417d30  
SONY:     insert drive 2  
```

On my code that I cloned from git yesterday (see version in console messages), this appears to have been invoked from `src/arch/macplus/macplus.c` on line 826.
For reference, here is `ifconfig tun0`. I just noticed the 8-bit subnet thing, which seems kind of odd:

```
tun0: flags=8851<UP,POINTOPOINT,RUNNING,SIMPLEX,MULTICAST> mtu 1500  
    inet 10.10.10.2 --> 10.10.10.255 netmask 0xff000000 
    open (pid 7286)
```
