---
id: 195
title: Collection of Fedora Tweaks
last_modified_at: 2019-09-21T18:56:15+01:00
permalink: /2019/09/21/collection-of-fedora-tweaks/
categories:
  - Uncategorized
---
## Window Switcher

By default, windows in the window switcher (ALT + Tab) are grouped by applications. The grouping can be disabled by changing the shortcut from triggering the application switcher to the window switcher. In order to do this, the `dconf-editor`_` can be used to switch the keybinding. The keybinding is in the folder `org/gnome/desktop/wm/keybindings`.

First, the value of keybinding `switch-applications` has to be set to `[]`. After that, the keybinding of _switch-windows_ can be set to `['<Super>Tab', '<Alt>Tab']`.

## Nextcloud Client

The nextcloud client is available from the repository and can be installed via `nextcloud-client`. Unfortunately, the login credentials are not stored after installing the client. The reason is that the package `libgnome-keyring` is missing. After installing it, credentials are stored properly.

## Enable Netflix and Spotify in Firefox

Obviously, DRM has to be enabled in the Firefox preferences. However, this is not sufficient. What is still missing, are the required codecs. First, the required repositories have to be enabled:

```bash
dnf -y install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
dnf -y install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

After that, `ffmpeg` can be installed. After that, the plugins `OpenH264` and `Widevin` have to be enabled.