---
id: 75
title: Jailed Teamspeak 3 Server Using Systemd
last_modified_at: 2017-06-18T15:25:24+01:00
permalink: /2017/06/18/jailed-teamspeak-3-server-using-systemd/
categories:
  - Configuration
tags:
  - chroot
  - jail
  - systemd
  - teamspeak
---
Teamspeak is the de facto standard for voice chats during gaming. Unfortunately, Teamspeak does not offer a package usable with commonly available packet managers. Therefore, it is likely to be outdated. Therefore, I decided to execute the server jailed to limit the consequences of a possible attack. I will explain how to jail a Teamspeak 3 server and automatically start it using systemd.

<!--more-->

# Building Jail for Teamspeak 3 Server

First of all, we should get a copy of the latest Teamspeak server available in the [Teamspeak download section](https://www.teamspeak.com/downloads.html#server). After extracting the compressed archive, we can find several files including the server binary `ts3server`. This file will be executed later. Therefore, we have to find its dependencies in order to create the jail. I used [existing instructions (German)](https://www.feldstudie.net/2009/12/22/teamspeak-3-how-to-chroot/) but did not follow all of it completely. I will just give a short recap on what I did. The goal is to create a striped-down copy of your system that contains only the necessary parts to run the server.

You have to create a new folder for the jail that contains the following folders:

  * dev 
      * dev/null
      * dev/random
      * dev/urandom
      * dev/zero
  * etc
  * home/ts3
  * lib
  * lib64 (if you have a amd64 system)
  * tmp

The command `ldd ts3server` will give us all used shared libraries. For my system, the output is as follows

```
linux-vdso.so.1 (0x00007fff27fbf000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f5828543000)
librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f582833b000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f5828039000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f5827e1c000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f5827a71000)
/lib64/ld-linux-x86-64.so.2 (0x00007f582874e000)
```

You can find the mentioned libraries in the shown locations. You do not have to find `linux-vdso.so.1`. copy all libraries to their respective folders in your jail directory.

Afterwards, copy the files `group`, `passwd`, `hostname`, `host`, `resolv.conf` from your `/etc` folder into the jailed etc folder. Strip down the groups and passwords files, so they only contain the user ts3 that will run the server and will be jailed. If you have not created the user yet, create a user ts3 as a system user with a home directory at `/home/ts3`.

Copy the extracted server into the home directory inside the jail. The remaining task for setting up the jail is mounting the devices as defined in the script below.

```bash
#!/bin/bash

mount -o bind /dev/urandom /opt/ts3/chroot/dev/urandom
mount -o bind /dev/random /opt/ts3/chroot/dev/random
mkdir /opt/ts3/chroot/dev/shm
mount -t tmpfs tmpfs /opt/ts3/chroot/dev/shm/
```

The script will be executed by systemd.

# Configurations for Systemd

Systemd is the new default init system in Debian and Ubuntu. We will use it to automatically start and stop the teamspeak server. [Service units](https://www.freedesktop.org/software/systemd/man/systemd.unit.html) replace the former init scripts. The service units are rather configuration files than scripts. There are several build-in features that have not to be scripted anymore. One of this features is the execution inside a chroot environment.

The script consists of three sections: The unit, service, and install section. The unit section defines the name and dependencies. Because the server requires a network connection, we define dependencies to the online status of the network. The service section basically consists of the service to execute and preparation of the execution. As preparation step, be jail preparation script above has to be executed. It is important to set `PermissionsStartOnly` and `RootDirectoryStartOnly` to `yes` in order to execute the preparation script outside of the jail and as root user. The `RootDirectory` has to be set to the jail folder. the `ExecStart` command has to be given relative to the jail path. For the teamspeak server, it is necessary to adjust the `LD_LIBRARY_PATH`. In order to make this command work as expected, the working directory has to be set to the folder, in which the server binary is located. The install section defines the default execution.

```
[Unit]
Description=Teamspeak 3 Server
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
RootDirectory=/opt/ts3/chroot
ExecStartPre=/opt/ts3/prepare.sh
Environment="LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH"
ExecStart=/home/ts3/teamspeak3-server_linux_amd64/ts3server
RootDirectoryStartOnly=yes
PermissionsStartOnly=yes
WorkingDirectory=/home/ts3/teamspeak3-server_linux_amd64
User=ts3
Group=ts3

[Install]
WantedBy=multi-user.target
```

The service unit has to be stored in the `/etc/systemd/system` directory. Afterwards, it has to be enabled via `systemctl enable ts3`. The server will now be started and stopped automatically.