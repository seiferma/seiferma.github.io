---
id: 79
title: Watching Netflix Even If Using IPv6 Tunnelbroker
last_modified_at: 2017-06-18T17:15:57+01:00
permalink: /2017/06/18/watching-netflix-even-if-using-ipv6-tunnelbroker/
categories:
  - Configuration
tags:
  - dns
  - iptables
  - netflix
  - openwrt
  - tunnelbroker
---
Unfortunately, there are still providers that do not provide IPv6 connectivity. Therefore, I have to use a tunnel broker to gain IPv6 connectivity. Netflix, however, [blocks the address ranges of tunnel brokers](https://torrentfreak.com/netflix-blocks-ipv6-over-geo-unblocking-fears-160608/) because they have been used to circumvent the geographical blocking of content in several countries. Therefore, I configured my OpenWrt router to not use IPv6 for Netflix.  
<!--more-->

The basic approach for solving the problem is not blocking the IPv6 ranges of the Netflix services but adjusting the answer of the DNS server to only reply with IPv4 addresses. I chose this approach because Netflix uses the Amazon Web Services and blocking these address ranges would affect other services as well. Fortunately, there already is an [excellent manual](https://gist.github.com/jamesmacwhite/6a642cb6bad00c5cefa91ec3d742e2a6) for configuring an OpenWRT router like this. The required steps are

  1. Recompiling the bind package for OpenWrt (necessary until including 15.05.1)
  2. Configure bind and dnsmasq to discard AAAA records for Netflix domains
  3. Configure destination nat for Google DNS servers (not covered in the manual completely)

# Recompiling Bind-Server

First of all, the build environment has to be set up. I used the [OpenWrt manual](https://wiki.openwrt.org/doc/howto/buildroot.exigence) to do this. Basically, you have to execute

```bash
apt-get build-essential file gawk gettext git git-core libncurses5-dev libssl-dev mercurial python subversion unzip zlib1g-dev
```
on a Debian system and checkout the OpenWrt code
```bash
git clone --depth 1 --branch v15.05.1 https://github.com/openwrt/openwrt.git
```

The compilation of the bind server is covered in [another manual](https://www.ploek.org/post/netflix_openwrt/). You have to

  1. `./scripts/feeds update -a`
  2. `./scripts/feeds install -a`
  3. `make menuconfig` 
      1. Select correct target platform
      2. Select bind-server (modular) under `Network` → `IP Addresses and Names` → `bind-server`
      3. Save configuration
  4. Edit `feeds/packages/net/bind/Makefile` and add `--enable-filter-aaaa` to configuration args
  5. `make tools/install`
  6. `make toolchain/install` 
      1. If this fails, append `j=1 V=s` to the command and reexecute
      2. If the kernel could not be downloaded, look for the required kernel on [kernel.org](https://cdn.kernel.org/pub/linux/kernel/v3.x/) and copy it into the `dl` folder.
  7. `make package/bind/compile`
  8. `make package/bind/install`
  9. Copy the results to your router via SCP `scp bin/ar71xx/packages/packages/bind-* root@192.168.0.1:/tmp/`

# Configuration of OpenWRT

Install the new bind-server and bind-libs via
```bash
opkg install --force-checksum /tmp/bind-*
```
Afterwards, replace the configuration of bind located in `/etc/bind/named.conf` with

```
options {
    directory "/tmp";

    forwarders {
	8.8.8.8;
	8.8.4.4;
    };
    forward only;
    dnssec-enable no;
    auth-nxdomain no;
    listen-on port 2053 { 127.0.0.1; };
    filter-aaaa-on-v4 yes;
};
```

Restart the bind-server with `/etc/init.d/named restart`. Add the following entries to the DNS forwardings configuration and restart dnsmasq.

```
/netflix.com/127.0.0.1#2053
/netflix.net/127.0.0.1#2053
/nflxext.com/127.0.0.1#2053
/nflximg.com/127.0.0.1#2053
/nflxvideo.net/127.0.0.1#2053
```

Now, all requests for Netflix domains will only return A records as long as the DNS server of your OpenWrt router is used. Unfortunately, Chromecast devices and the Netflix app have hard coded DNS servers (at least on Android). Therefore, we have to force them to use our DNS server as well. We cannot do this by simply blocking the DNS servers because this prohibits the videos from being loaded. Instead, we apply destination NAT provided by iptables to redirect the traffic to the hard coded DNS servers of Google. Simply add the following rules to your custom firewall configuration and apply it. The rules redirect all traffic from your local network to the Google DNS servers towards your local DNS server that will resolve the addresses correctly.

```bash
iptables -t nat -A PREROUTING -s 192.168.0.0/24 -d 8.8.8.8 -p udp --dport 53 -j DNAT --to 192.168.0.1
iptables -t nat -A PREROUTING -s 192.168.0.0/24 -d 8.8.8.8 -p tcp --dport 53 -j DNAT --to 192.168.0.1
iptables -t nat -A PREROUTING -s 192.168.0.0/24 -d 8.8.4.4 -p udp --dport 53 -j DNAT --to 192.168.0.1
iptables -t nat -A PREROUTING -s 192.168.0.0/24 -d 8.8.4.4 -p tcp --dport 53 -j DNAT --to 192.168.0.1
```