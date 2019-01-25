---
layout: post
title:  "Boot vEOS failed"
date:   2019-01-25 17:00 +0800
categories: virtualization
---

I have some virtual machines running the vEOS image from Aristra. Recently the images failed to boot up after reboot. Content of /mnt/flash/boot-config:

```
$ cat /mnt/flash/boot-config
SWI=flash:/vEOS.swi
```

Below error is shown on console:

```
ISOLINUX 4.05 2011-12-09  Copyright (C) 1994-2011 H. Peter Anvin et al
Loading linux.....
Loading initrd.....ready.


Aboot 8.0.0-3255441
running dosfsck
Seek to -261194240:Invalid argument
dosfsck 2.11, 12 Mar 2005, FAT32, LFN


Press Control-C now to enter Aboot shell
Booting flash:/vEOS.swi
 not found or not a file
Welcome to Aboot.
Aboot#
```

If type `boot falsh:/vEOS.swi` on Aboot shell, then the VM can boot up normally.

The fix for this issue is simple. After the VM is up, run below command:
```
$ echo "SWI=flash:vEOS.swi" > /mnt/flash/boot-config
```

The slash before image name is removed. Then it works. The VMs can always boot up corretly after reboot. I don't know why, but it just works.
