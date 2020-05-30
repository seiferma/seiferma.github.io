---
id: 113
title: Installing LineageOS on Samsung Galaxy S2
last_modified_at: 2018-04-07T22:45:26+01:00
permalink: /2018/04/07/installing-lineageos-on-samsung-galaxy-s2/
categories:
  - Configuration
tags:
  - android
  - i9100
  - lineageos
---
If you run a Samsung Galaxy S2 with stock ROM, you cannot install LineageOS in the usual, straight forward way: Partitions have to be resized and flashing custom recovery is not as simple as for other phones. This summarizes the necessary steps for setting up LineageOS on the device running a stock ROM.

<!--more-->

# Preparation

Download the following tools/files

  * [Heimdall flashing tool](https://glassechidna.com.au/heimdall/)
  * [LineageOS](https://download.lineageos.org/i9100)
  * Partition table layout (see attachments)
  * [TWRP recovery](https://dl.twrp.me/i9100/)

# Flashing

  1. Boot in download mode (Volume Down + Home + Power)
  2. Execute the driver setup for Heimdall (the relevant device is Gadget Serial)
  3. Execute the following command as priviledged user(boot.img is part of LineageOS)  
    `heimdall flash -repartition -pit I91001GB_4GB.pit -RECOVERY twrp-3.1.0-0-i9100.img -KERNEL boot.img -no-reboot`
  4. Power off phone, remove battery, insert battery, and boot into recovery (Volume Up + Home + Power)
  5. Resize Partitions (do this for system, data, internal storage and ignore shown errors) 
      1. Advanced wipe - select partition - repair or change file system - resize
      2. Back - change file system - ext4
  6. Power off phone, remove battery, insert battery, and boot into recovery
  7. Wipe dalvik cache, cache, system, data, internal storage
  8. Push zip files of LineageOS and OpenGapps to sdcard0
  9. Install zip files

# Sources

I used the [manual of Lukas Knauer](https://www.planetknauer.net/blog/archives/2017-05-LineageOS-auf-Galaxy-S2-i9100-installieren.html) that has been written in German.