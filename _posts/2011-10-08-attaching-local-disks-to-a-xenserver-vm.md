---
layout: post
title: Attaching local disks to a Citrix XenServer VM
excerpt: "Attach your local disks to a VM running on Citrix XenServer"
tags: [harddisk, linux, udev, citrix, xenserver, virtualization]
modified: 2011-10-18
date: 2011-10-18
comments: true
---

So this story goes like this: All my precious media files are kept inside two enterprise level hard disks (think of WD RE4 and such) buried inside a few external USB cases. However, after setting up an Ubuntu VM in PV mode in my Citrix XenServer testing system, I got the urge to attach the aforementioned hard disk to the VM I had just installed and share it over my local network through Samba (and later, iSCSI).

**Disclaimer: As this story involves procedures related to tampering with hard disk drives, always keep an up-to-date backup of your data before trying anything! I will not be responsible for any data loss resulting from poor safety measures!**

This would be an extremely easy thing to do if I wanted USB speeds, as XenServer supports attaching external USB storage to VMs out-of-the-box (for backup purposes). But USB external storage maxes out at ~ 30 MB/s and my hard disk has a read/write throughput of ~80 MB/s, so I popped the server case open, connected it through the actual SATA interface and started trying to figure a way to make XenServer believe that this was actually a USB disk and not a locally attached drive.

What we already know is that XenServer uses the well-praised Linux 2.6 kernel right under the Xen hypervisor which abstracts USB as well as SATA (and newer libata-based PATA) connected hard disks with the a SCSI disk device naming scheme of sdxy, where x is a letter assigned to the device in boot time (depending on the device-scanning order) and y stands for the legacy (not EFI) partition number(s) the drive contains. XenServer also hosts a full-blown Linux distribution (think Redhat), so this naming scheme (any much, much more) is read by the udev daemon at boot time and the daemon creates the appropriate device nodes in the /dev directory.

Now, after some digging, I found out that the Xen hypervisor (the XAPI backend to be precise) actually utilized the udev-created device nodes in the /dev tree to attach physically connected devices to the VMs.

**This is done by simply creating a symlink of the device in question (i.e. sda, sdb, sdc etc.) inside the /dev/xapi/block directory!
So what I did was to modify the server’s /etc/udev/rules.d/50-udev.rules and add these lines (sdb was the detected device name of my drive):**

{% highlight cfg %}
ACTION=="add", KERNEL=="sdb", SYMLINK+="xapi/block/%k", RUN+="/bin/sh -c '/opt/xensource/libexec/local-device-change %k 2>&1 >/dev/null&'"
ACTION=="remove", KERNEL=="sdb", RUN+="/bin/sh -c '/opt/xensource/libexec/local-device-change %k 2>&1 >/dev/null&'"
{% endhighlight cfg %}

**Hey!** You can repeat the above rules for more than one hard drive, replacing the relevant kernel device names.
{: .notice}

The code above adds 2 actions to udev:

* When the device is detected, add a symlink to /dev/xapi/block/(kernel name given to the device) and run a script (this was copied from the USB mass storage device detection rules)
* When the device is removed (that is, if the disk is detached by hot-unplugging it-this is a whole procedure, don’t try this at home if you don’t do your research first), run the same script as in "add"

After rebooting the server, the hard disk was immediately recognized, symlinked to the /dev/xapi/block directory and was available through the xe command or XenCenter, waiting to be attached to a VM! As a nice extra, in Citrix XenServer, the USB-attached mass storage device is not emulated as a drive attached to an emulated USB controller. Rather, it is connected through an emulated SCSI controller and led by a near-native speed PVOPS driver on Linux guests, effectively removing any bandwidth limit posed by emulating USB without loosing the hot-plugging ability!

In the end, attaching the disk was a breeze and the transfer rate of the drive prove the effort’s worth as I got more than 50 MB/s throughput, doing network file copy over CIFS and GbE.

That’s it! Feedback is always welcome :)