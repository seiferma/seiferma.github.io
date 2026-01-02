---
id: 213
title: Install debian without default user via cloudinit
last_modified_at: 2026-01-02T21:00:00+01:00
permalink: /2026/01/02/debian-cloudinit-root
categories:
  - Configuration
---
Cloudinit provides means to initialize a VM instance after it has been provisioned by an image.
Many images create a default user but you can skip this step.

<!--more-->

The snippet allows you to only create a root user with a predefined ssh key.
It has only been tested with Debian but should work as well for other distributions.

```
#cloud-config
timezone: Europe/Berlin
disable_root: false
users: []
autoinstall:
  ssh:
    allow-pw: false
    authorized-keys:
      - "ssh-ed25519 YOURKEY"
    install-server: true
```
