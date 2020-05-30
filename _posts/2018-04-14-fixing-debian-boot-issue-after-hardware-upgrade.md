---
id: 119
title: Fixing Debian Boot Issue After Hardware Upgrade
last_modified_at: 2018-04-14T10:14:05+01:00
permalink: /2018/04/14/fixing-debian-boot-issue-after-hardware-upgrade/
categories:
  - Configuration
tags:
  - linux grub
---
I decided to upgrade a Debian based system with a new motherboard. I did not expect any issues because Linux is quite flexible regarding hardware changes. Unfortunately, the detection order of my hard drives changed (which is most probably my fault). This lead to an issue during the boot process saying that the root file system cannot be mounted.  
<!--more-->

The specific error shown was  
```
mount: can't read '/etc/fstab': No such file or directory
mount: mounting /dev on /root/dev failed: No such file or directory
mount: mounting /sys on /root/sys failed: No such file or directory
mount: mounting /proc on /root/proc failed: No such file or directory
No init found. Try passing init= bootarg.
```

The cause of the issue was a fixed parameter `root` in the Grub configuration that did not use a UUID but the name of the device. This name, obviously, did not match my new names. The boot failed.

The solution is to press the `e` key in the grub boot menu and change this parameter to the correct name. After booting, you just have to update grub by issuing the commands

  * `update-grub`
  * `grub-install /dev/sdX`

Afterwards, the system boots as before.

Sources: [Stackexchange](https://unix.stackexchange.com/questions/120198/how-to-fix-boot-into-initramfs-prompt-and-mount-cant-read-etc-fstab-no-su) and [HowToUbuntu](http://howtoubuntu.org/how-to-repair-restore-reinstall-grub-2-with-a-ubuntu-live-cd)