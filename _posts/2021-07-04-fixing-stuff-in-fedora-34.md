---
id: 207
title: Fixing Stuff After Upgrading to Fedora 34
last_modified_at: 2021-07-04T19:45:00+02:00
permalink: /2021/07/04/fixing-stuff-in-fedora-34
categories:
  - Configuration
---
I upgraded to Fedora 34 and experienced some difficulties. This is mostly a dump of things I changed, so I can remember.

<!--more-->

# USB Sound Device not Detected Properly
I use a USB audio device but I could not get it to run properly. The solution was to remove UDEV rules as suggested as a workaround in a [pipewire ticket](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/435). The rules are located in the file `/lib/udev/rules.d/90-pipewire-alsa.rules`, so I removed it.

# Monitor Looses Signal Frequently
I use an integrated Intel GPU and a monitor connected via HDMI. Errors like `CPU pipe B FIFO underrun` were logged. The solution was to add the kernel parameter `i915.enable_dp_mst=0`. In Fedora 34, `grubby` shall be used to add kernel parameters, so I executed `grubby --update-kernel=ALL --args=i915.enable_dp_mst=0`.
