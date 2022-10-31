---
id: 209
title: Hints for Strato Services
last_modified_at: 2022-10-31T20:00:00+01:00
permalink: /2022/10/31/hints-for-strato-services
categories:
  - Configuration
---
I had a look at quite cheap Strato VPSs and wanted to collect a few tips.

<!--more-->

# Too low task limit

I received strange errors, which seemed random to me. They were all related with thread resources, which are temporarily not available. Luckily, a guy on [stackoverflow](https://stackoverflow.com/questions/57942371/docker-runtime-cgo-pthread-create-failed-resource-temporarily-unavailable) pointed out the issue, which is a too low limit for threads.

To increase the limit, you have to edit the file `/etc/systemd/system.conf` and increase the value of `DefaultTasksMax` (e.g. to 1000).