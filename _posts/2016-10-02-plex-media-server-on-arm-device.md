---
id: 68
title: Plex Media Server on ARM Device
last_modified_at: 2016-10-02T20:04:58+01:00
permalink: /2016/10/02/plex-media-server-on-arm-device/
categories:
  - Configuration
---
There is a wide range of ARM devices that are capable of running a Plex media server. Unfortunately, Plex does not provide a generic package for ARM platforms but only installers for specific devices. Luckily, [Jan Friedrich](https://www.dev2day.de/typo3/) provides such a package.<!--more-->

You can install the server in four easy steps

```bash
wget -O - https://dev2day.de/pms/dev2day-pms.gpg.key | apt-key add -
echo "deb https://dev2day.de/pms/ jessie main" >> /etc/apt/sources.list.d/plex.list
apt-get update
apt-get install plexmediaserver-installer
```

The remaining configuration works as usual. If you are missing the images (covers, fanart, and so on), you have to adjust the [local authentication settings](https://support.plex.tv/hc/en-us/articles/200890058-Require-authentication-for-local-network-access) to allow access from your local network.