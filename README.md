# Time capsule plans

What to do with an old Time Capsule, as it drifts towards incompatibility with
future MacOS.

Specifically, the problem is:

* Time machine exports via Apple File System, and this won't be supported in
  the future.
* It also provides a Samba version 1 share, but Samba version 1 is buggy, and
  not usable for Time Machine backups.

## Option 1 - represent Time Capsule via another server

For example, the server might be a Raspberry Pi.

On the server, with Time Capsule at IP 192.168.1.50, Time Capsule share name of "Greenish", internal server mount point is `/mnt/greenish`

```bash
sudo mount.cifs //192.168.1.50/Greenish /mnt/greenish -o username=<smbuser>,password=<smbpass>,vers=1.0,uid=1000,gid=1000,sec=ntlm
```

See below for problems using needed `sec=ntlm` on Linux kernels more recent than 5.15.

In due course, put this mount command in `/etc/fstab`.

Edit the `smb.conf` from example below, and restart service:

```
sudo vim /etc/samba/smb.conf
sudo service smbd restart
```

`smb.conf`:

```
[global]
...
security = user
wide links = yes
unix extensions = no
vfs object = acl_xattr catia fruit streams_xattr
fruit:nfc_aces = no
fruit:aapl = yes
fruit:model = MacSamba
fruit:posix_rename = yes
fruit:metadata = stream
fruit:delete_empty_adfiles = yes
fruit:veto_appledouble = no
spotlight = yes
...

[Greenish]
Comment = Time Capsule
Path = /mnt/greenish
Browseable = yes
Writeable = yes
only guest = no
create mask = 0777
directory mask = 0777
Public = yes
Guest ok = no
available = yes
# valid users = (your-username)
vfs objects = catia fruit streams_xattr
fruit:time machine = yes
# fruit:time machine max size = 250G
```

Kernel on server must be < 5.15 to allow `sec=ntlm` above - see [ntlm 'bad
option'](https://bbs.archlinux.org/viewtopic.php?id=271264).

Accordingly mount command at top works on my `5.10.103-v7+` Raspberry Pi, but not on my `6.12.34+rpt-rpi-2712` Raspberry Pi.

It's very difficult to downgrade the kernel to
5.1 on Raspberry Pi OS.  It is probably not possible to do this on the Pi5
(see below).  For a Pi4, I've tried suitable modifications to these [kernel
downgrade
instructions](https://github.com/HinTak/RaspberryPi-Dev/blob/master/Downgrading-Pi-Kernel.md),
and
[rpi-update](https://www.reddit.com/r/raspberry_pi/comments/5lo5do/how_do_i_downgradeupgrade_to_a_specific_kernel),
without success.  I succeeded in making the Pi unbootable with the first set
of instructions.  I ended up downloading
`2022-09-22-raspios-buster-armhf.img.xz` from
<https://archive.raspberrypi.org/debian/pool/main/r/raspberrypi-firmware/>,
and burning to an SD card with the [Raspberry Pi
imager](https://www.raspberrypi.com/software/) (selecting Other, image file).
I believe this is the last image with a kernel < 5.15 (it has `5.10.103-v7+`).
I made sure I did no updates that might upgrade the kernel after installing,
and ran `sudo apt-mark hold raspberrypi-kernel`, as in the kernel downgrade
instructions above.

Google Gemini claims that the earliest kernel that will boot on the Pi 5 is
6.1 — see <https://www.raspberrypi.com/software/operating-systems> as indirect
evidence.  Specifically, it claims the kernel before 6.1 did not support the
Pi 5 `bcm2712` chip.

[Some more discussion of mounts, Time
Capsule](https://github.com/maxx27/afpfs-ng-deb).

## Insert newer Samba server into Time Capsule

See [overview of process](https://github.com/jamesyc/TimeCapsuleSMB).

This looks extremely fiddly, and in particular, the process of cross-compiling
a Samba server for `evbarm` architecture, and configuring it for the Time
Capsule, which has a very sparse operating system.

My Time Capsule is:

```
$ uname -a
NetBSD flashyphase 4.0_STABLE NetBSD 4.0_STABLE #0: Fri May 24 19:48:23 PDT 2019  root@xapp190.apple.com:/BuildRoot/Library/Caches/com.apple.xbs/Sources/M52/AirPortFW-78100.3/Embedded/Firmware/NetBSD/Targets/M52/release/obj/build.kernel-target.conf evbarm
```

Maybe keep an eye for someone who has done the cross-compilation, and
`rc.local` etc setup.
