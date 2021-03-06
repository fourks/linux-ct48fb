ct48fb - Linux kernel 2.4.x driver for C&T 65548/65545/65540 video chips
========================================================================


Maciej Witkowiak <ytm@elysium.pl>
19.01.2003-14.08.2004

Here is everything you need to have accelerated framebuffer support on
a Chips&Technologies 65548/65545/65540 video chips.

65548 video chip is installed in my Toshiba Satellite Pro 410CS
and it was the only testing platform when writing this code.
Supported video modes: 640x480x8, 640x480x16, 800x600x8 and 800x592x16.
For previous versions of this driver I had success reports about 65545 so
only 65540 support remains unconfirmed.

PCI support has been written by Martin Krohn <martin.krohn@web.de>.

NOTE: This driver have been developed only using LCD panel. If it works with
CRT display - fine, but I don't give any guarantee that it does.



#Bugs and limitations

Help me with these if you can:
- Hardware accelerated putc (put a character on screen) is disabled by default
  because it will hang your machine if you run gpm. I don't know the reason,
  it probably has something with gpm cursor. Scrolling up through shell history
  may cause problems too
- when using native chips driver in X11 the screen may be garbled after
  exiting X, run 'fbset -pixclocks 20001' then (or sth similar - just to change
  video clock setting)


#Patching the kernel

Unpack kernel sources to e.g. /usr/src/linux-2.4.23/, copy ct48fb.patch there,
there and do:

```
    cd /usr/src/linux-2.4.23/
    patch -p1 <ct48fb.patch
```
	
Copy ct48fb/ct48fb.c to

```
    /usr/src/linux-2.4.23/drivers/video/
```

Then do make menuconfig:
- Code maturity level options
- enable  'Prompt for development and/or incomplete code/drivers'
- Console Drivers
- disable 'VGA text console' (important!)
- enable  'Video mode selection support'
- Frame-buffer support
- enable  'Chips 65548 display support'
- enable  'Advanced low-level driver options'
- enable  '8bpp packed pixel support'
- enable  '16bpp packed pixel support'

(Note that it is not necessary to enable PCI bus support unless your computer
 has it.)

Then make bzImage, make modules, etc. the usual stuff
Install the new kernel, setup LILO, read and apply notes below about using
ct48fb compiled in kernel.



#Compiling only as a module

Go to module/ directory, do 'make'. If it doesn't work edit Makefile and set
correct path to kernel headers (usually /usr/include/linux or /usr/src/linux/include/)
Do 'make install' or manually copy resulting ct48fb.o file to
/lib/<kernel-version>/kernel/drivers/video/, then do 'depmod -a'.
Read and apply notes about using a module below.

Make sure that you have modules fbcon-cfb8 and fbcon-cfb16 (they are selected
by '8 (16) bpp packed pixel support' as explained above. To force their build
as modules, select e.g. VGA16 framebuffer support as a module and select them.



#Using ct48fb compiled in kernel

This framebuffer driver is unable to set initial video mode on its own.
You have to do it. When using ct48fb as compiled in (DO NOT enable VGA text
console then!) all you have to do is to add:

```
  vga=377
 ```

to kernel boot parameters. This is my setup from /etc/lilo.conf. As you see
there is a global vga parameter as well as a local one to fb kernel.


```
    vga=4
    image=/boot/vmlinuz-2.4.23
	label=linux
	root=/dev/hda2
	read-only
    image=/boot/vmlinuz-2.4.23-fb
	label=fblinux
	root=/dev/hda2
	vga=377
	read-only
```


#Using ct48fb as a module

Again, you have to setup video mode to be able to see anything.
Included program: ct48mode will do it for you.
Copy it somewhere, e.g. to /usr/sbin/ and add following line to your
/etc/modules.conf file:

```
    pre-install ct48fb /usr/sbin/ct48mode
```
	
Do depmod -a
Now it is safe to load the module by issuing:

```
    modprobe ct48fb
```

Note that ct48mode must be run by root or be set suid root.
There is also similar ct48text program that will eventually help you to
regain control over text console without reboot.



#Troubleshooting

There are two kind of troubles that both will result with blank screen:
- ct48fb module was loaded but you forgot to update modules.conf as explained
  above - check paths etc.
- ct48mode was executed but module was not loaded. This can be caused by either
  actual kernel version and used kernel headers version mismatch or you don't
  have modules that ct48fb depends on: fbcon-cfb8 and fbcon-cfb16
  Blindly try to execute ct48text tool and once you resolve the problem try again.

In any case - it's just the screen that went blank. The rest of your system still
works so you can still safely reboot. If you can't use telnet or ssh to check for
problems try redirecting modprobe output to a file:

```
  modprobe ct48fb >err1 &>err2
```
  
and then check what error messages are there. Checking system logs for possible
error messages will be a good idea too.

DO NOT use ct48mode when driver is loaded and running - you will get blank
screen and you will have to reboot.



#Additional options

The driver supports following options when compiled in kernel:

    accel/noaccel	- enable/disable hardware accelerating engine (default=enable)
    accputc/noaccputc	- enable/disable accelerated putc (default=disable)
    hwcursor/nohwcursor	- enable/disable hardware cursor (default=enable)
    blink/noblink	- enable/disable blinking of hardware cursor (default=disable, because it looks ugly)
    inverse/noinverse	- enable/disable screen inverse (default=disable)
    mode:<xxx>x<yyy>x<bpp> - select one of predefined modes (default=640x480x8)

You can pass it in a similar way like other fb drivers by appending e.g.

```
    video=ct48fb:accel,noblink,noaccputc,hwcursor,noinverse,mode:640x480x8 \
       vga=377
```
	   
to kernel boot parameters. Note that this line will not change anything because
it has only default values.

For kernel module there are following options:

```
    noaccel, noaccputc, nohwcursor, noblink, noinverse, mode
```
	
Each option can be disabled (0) or enabled (1). Default would be:

```
    modprobe ct48fb noaccel=0 noaccputc=1 nohwcursor=0 noblink=1 \
	 noinverse=1 mode=640x480x8
```

There are 4 supported modes:

```
  640x480x8, 640x480x16, 800x600x8, 800x592x16
```

It is possible to select one of them upon boot/loading module or switch the
video mode in runtime by using fbset utility - e.g.:

```
  fbset -xres 640 -depth 8
  fbset -xres 640 -depth 16
  fbset -xres 800 -depth 8
  fbset -xres 800 -depth 16
```

It is sufficient to provide only xres and depth, the driver will select proper
mode automatically. You can also provide only xres or only depth. Of course
things like

```
  fbset 640x480-60
```

will work too.

If you don't like default refresh setting (e.g. 16bpp mode requires higher
clock rate) you can change it by using -pixclock option from fbset. Note that
the value provided for -pixclock is time in picoseconds, not frequency!
Therefore, you have to provide a smaller number to have higher clock rate.



Have fun!

ytm
