
What to do with an old Time Capsule, as it drifts toward
incompatibility with future MacOS?

Specifically, the problem is:

* Time machine exports via Apple Filing Protocol (AFP), and
  this won't be supported as a target for Time Machine in
  future MacOS.
* It also provides a Samba version 1 share, but Samba version
  1 is buggy, appears unusable for Time Machine backups, and
  won't be supported in future MacOS.

## (Rejected) Make bridge to Time Capsule via another server

For example, the server might be a Raspberry Pi.

The idea is that the Pi mounts the Time Capsule data via SMB 1 or AFS.

I spend some considerable time on this, and ended up concluding it could work reliably.

Also see [this discussion](https://github.com/maxx27/afpfs-ng-deb) for working notes and background.

### Mount time capsule via Samba 1

This proved difficult to configure.  If you want to try it,
read further.  Summary — as [James Chang
found](https://github.com/jamesyc/TimeCapsuleSMB/blob/main/building/build.sh#L7)
— one can mount the Time Capsule, and export that mount with
a more recent version of Samba, but Time Machine backups fail
after about 10 minutes, without explanation.  I suspect that
the Capsule Samba 1 implementation is buggy.

With that background, read on if you're interested.

Note: *CIFS* ([Common Internet Filing
System](https://www.techtarget.com/searchstorage/definition/Common-Internet-File-System-CIFS))
is an alternative name for the Samba 1 protocol.

Say the Time Capsule is at address `AncientTime.local`, and
Time Capsule share name is `Data` (the default). You've
already make a mount point on the Pi of `/mnt/ancient`.  In
`/etc/fstab`. Note `cifs` as mount type (Samba 1).

```bash
//AncientTime.local/Data /mnt/ancient cifs nofail,rw,workgroup=WORKGROUP,iocharset=utf8,credentials=/home/pi/.ancient.conf,sec=ntlm,rsize=130049,wsize=81920,cache=loose,vers=1.0,uid=pi,gid=pi 0 0
```

Where `~pi/.ancient.conf` is:

```bash
username=pi
password=<password to Time Capsule>
```

Notice I'm forcing the user to `pi`, and I'll have to later
attach to the modern Samba service as this user to be able to
read and write to the share.  I had to do this to overcome
permission errors.

You can do a similar mount from the command line:

```bash
sudo mount.cifs //AncientTime.local /mnt/ancient -o username=pi,password=<smbpass>,vers=1.0,uid=pi,gid=pi,sec=ntlm
```

For these to work, you need `ntlm` authentication in the Linux kernel.  This was removed in kernel version 5.15.  I used an old distribution of Raspberry Pi OS; you could also try recompiling the kernel with `ntlm` restored.  See below.

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

[ancient-share]
Comment = Ancient Time Capsule share
Path = /mnt/ancient
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

### NTLM authentication and the Linux kernel

The kernel on your Linux (maybe Raspberry Pi) server must be
< 5.15 to allow `sec=ntlm` above - see [ntlm 'bad
option'](https://bbs.archlinux.org/viewtopic.php?id=271264).

Accordingly the mount command mount command works on my
`5.10.103-v7+` Raspberry Pi, but not on my
`6.12.34+rpt-rpi-2712` Raspberry Pi.

It's difficult to downgrade the kernel to
5.10 on Raspberry Pi OS, and it is probably not possible to do
this on the Pi5 (see below).

If you want to start with a newer kernel, and downgrade, you
can try these instructions, that failed for me. For a Pi4,
I've tried suitable modifications to these [kernel downgrade
instructions](https://github.com/HinTak/RaspberryPi-Dev/blob/master/Downgrading-Pi-Kernel.md),
and
[rpi-update](https://www.reddit.com/r/raspberry_pi/comments/5lo5do/how_do_i_downgradeupgrade_to_a_specific_kernel),
without success.  I succeeded in making the Pi unbootable with
the first set of instructions.  I believe it is not possible
to downgrade the kernel below [6.1 on the
Pi5](https://www.raspberrypi.com/software/operating-systems)
— because 6.1 was the first kernel to support the Pi5's
`bcm2712` chip.

I solved these problems on a Pi5, by starting with an image
with a 5.10 kernel.  Specifically, I downloaded
`2022-09-22-raspios-buster-armhf.img.xz` from
<https://archive.raspberrypi.org/debian/pool/main/r/raspberrypi-firmware/>,
and burnt it to an SD card with the [Raspberry Pi
imager](https://www.raspberrypi.com/software/) (selecting
Other, image file). I believe this is the last image with
a kernel < 5.15 (it has `5.10.103-v7+`). I made sure I did no
updates that might upgrade the kernel after installing, and
ran `sudo apt-mark hold raspberrypi-kernel`, as in the kernel
downgrade instructions above.

Anyway, as I noted above, this mount and represent method gave me failed backups without any error — I assume this is because the CIFS / Samba 1 protocol support is buggy on the Time Capsule, and I was reluctant to explore further.

### Mounting via AFS

I also tried mounting the Time Capsule via AFP (Apple Filing
Protocol).

You can do this with the
[afpfs-ng](https://github.com/Netatalk/afpfs-ng) package.
This isn't available with recent versions of Debian, including
on the Pi. If you followed along above, you'll see I had an
old version of Raspbian, and could do `sudo apt install
afpfs-ng`, giving me version `0.8.1-5+rpi1`.

See [this discussion of AFPFS on
Debian](https://github.com/maxx27/afpfs-ng-deb) for more on
compiling yourself for more recent distributions.

AFPFS uses the FUSE (Filesystem in USErspace) interface to
provide a filesystem from an AFP server.  However it has seen very little development since 2009 — which is why the package has been dropped from recent distributions.  I did manage to mount and read the disk for the default `pi` user, with:

```
sudo mount_afp "afp://myuser:my-tc-password@AncientTime/Data" /mnt/ancient
```

(where `my-tc-password` is obviously the password to the Time
Capsule).  I also tried mounting as `pi` with `sudo --user=pi
mount_afp ... `. But in neither case could I get the Pi's
Samba server to read this mount.  Perhaps I needed some way of
starting the `afpfs` daemon for the relevant users, but
reading around, it seemed to me that `afpfs-ng` was
sufficiently poorly supported that it was unlikely to give me
reliable backups, and I baulked at spending more time
debugging.


## Insert newer Samba server into Time Capsule

See [overview of process](https://github.com/jamesyc/TimeCapsuleSMB).

This looks extremely fiddly, and in particular, the process of
cross-compiling a Samba server for `evbarm` architecture, and
configuring it for the Time Capsule, which has a very sparse
operating system.

My Time Capsule is:

```
$ uname -a
NetBSD flashyphase 4.0_STABLE NetBSD 4.0_STABLE #0: Fri May 24 19:48:23 PDT 2019  root@xapp190.apple.com:/BuildRoot/Library/Caches/com.apple.xbs/Sources/M52/AirPortFW-78100.3/Embedded/Firmware/NetBSD/Targets/M52/release/obj/build.kernel-target.conf evbarm
```

I plan to keep an eye out for someone who has done the
cross-compilation, and `rc.local` etc setup.

## My plan

I'm leaning towards taking the disk out of the Time Capsule,
putting it into a USB 3.5 inch SATA enclosure, and running
a Time Machine Samba interface to it via the Raspberry Pi.
The `smb.conf` sections above worked correctly for this on
testing.

Then again, I have a spare Raspberry Pi and a spare USB
enclosure.  If I didn't, I guess I'd buy a cheap NAS to do the
same thing.
