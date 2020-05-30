---
id: 77
title: IPv6 Tunnelbroker in OpenVZ Environments
last_modified_at: 2017-06-18T16:21:52+01:00
permalink: /2017/06/18/ipv6-tunnelbroker-in-openvz-environments/
categories:
  - Configuration
tags:
  - ipv6
  - openvz
  - systemd
  - tunnelbroker
---
In times of diminishing IPv4 resources, dual stack should be available everywhere. Unfortunately, there are still providers that do not provide acceptable dual stack solutions. The provider of my virtual server does not assign a continuous IPv6 subnet to servers but single addresses with a /128 subnet. Use cases like providing a OpenVPN gateway for IPv4 and IPv6 become impossible. In this post, I will explain how to get a /64 IPv6 subnet and use it in a restricted OpenVZ environment.  
<!--more-->

# Gathering an IPv6 Subnet

If your provider does not assign an proper IPv6 subnet, you can use an IPv6 tunnel. Such tunneling techniques were meant to provide IPv6 subnets to those who could not get a subnet from their provider and those who wanted to make experiments with IPv6. Many free tunnel brokers closed their services because of the increasing coverage of IPv6 or because [they were claimed for freeing the providers from investing in IPv6 infrastructure](https://www.sixxs.net/sunset/). Nevertheless, there are still [some free tunnelbrokers](https://en.wikipedia.org/wiki/List_of_IPv6_tunnel_brokers).

I chose the service of Hurricane Electric because of its points of presence (PoPs) in multiple countries. Additionally, it is said to be reliable. First, you have to register a free account on the [tunnel broker website](https://tunnelbroker.net). Apply for a regular tunnel and provide the IPv4 address of your server. Select a PoP in your geographical region. The nearer the POP, the lower the increase of the latency by using the tunnel will be. In my case, I could select a PoP in Frankfurt, Germany. Almost all traffic of my server was routed via Frankfurt anyway, so there should be basically no latency penalty. After completing the application, you receive a IPv6 subnet.

# Configuration of the Server

Setting up the tunnel is pretty straight forward by using the configuration snippets from Hurricane Electric. If you are using a virtual server that is as restricted as mine running on OpenVZ, you have to use a workaround to get the tunnel up and running. I was not able to modify my interface configuration or enable the commonly used tunnel interface types. While trying this, I lost IPv4 connectivity of my server and had a pretty hard time to get my SSH connection back (my ISP does not provide IPv6 at all).

Fortunately, [other people](https://www.cybermilitia.net/2013/07/22/ipv6-tunnel-on-openvz/) already had the same problem and found a solution. You can use the small application [tb-tun](https://code.google.com/archive/p/tb-tun) to establish the necessary IO based on sockets and create a interface using this afterwards. First, you have to download the source code and compile it using `gcc tb_userspace.c -l pthread -o tb_userspace`. I could not use the binary included in the archive.

The next step is the configuration of the network interface. I used two shell scripts that are started via systemd. The first script opens the tunnel socket.

```bash
#!/bin/bash
POP=216.66.80.30
IP4=203.0.113.1
setsid /opt/6in4tunnel/tb_userspace tb $POP $IP4 sit > /dev/null 2>&1
```

The according service unit looks like following:

```
[Unit]
Description=Tunnelbroker Userspace
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/opt/6in4tunnel/tb_userspace.sh

[Install]
WantedBy=multi-user.target
```

The second script opens the tunnel interface and configures the routing. `IP6` is the ip address that shall be assigned to the network interface. This is an IP from your assigned /64 subnet. `OVPN` is the subnet of OpenVPN clients that shall be tunneled through the server. `IP6GW` is the IPv6 server endpoint that can be found on in the information about your tunnel on the Hurricane Electric website. The special routing for the OpenVPN subnet was necessary for me because I wanted to use the natively assigned IPv6 address for all communication from and to my server. The tunnel should only be used for the OpenVPN clients. This requires [source routing](http://www.tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.rpdb.simple.html) provided by the `iproute` package. You have to add the entry `20 openvpn` to `/etc/iproute2/rt_tables`.

```bash
#!/bin/bash
IP6=2001:db8::1/128
OVPN=2001:0db8:0000:0000:8000::/112
IP6GW=2001:470:1d0b:4cf::1

sleep 3s
ifconfig tb up
ifconfig tb inet6 add $IP6
ifconfig tb mtu 1480
route -A inet6 add ::/0 dev tb
ip -6 rule add from $OVPN table openvpn
ip -6 route add default via IP6GW dev tb table openvpn
```

The configuration script is run via the following systemd service unit:

```
[Unit]
Description=Tunnelbroker Config
Wants=tunnelbroker-userspace
After=tunnelbroker-userspace

[Service]
Type=simple
ExecStart=/opt/6in4tunnel/tunnelconfig.sh

[Install]
WantedBy=multi-user.target
```