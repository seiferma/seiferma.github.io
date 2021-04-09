---
id: 207
title: IPv6 Prefix Delegation via Wireguard
last_modified_at: 2021-04-09T16:20:00+02:00
permalink: /2021/04/09/ipv6-prefix-delegation-via-wireguard
categories:
  - Configuration
---
Hosting applications on virtual machines at commercial hosters is simple but can be costly if many resources are required. In contrast, resources might be freely available at home by reusing existing hardware. The downside of this is that dynamically assigned IPs impede connectivity. In addition, simply forwarding ports in a local router is not the best way to isolate publicly available resources. Therefore, I decided to delegate a static IPv6 prefix to a dedicated network to route the traffic via VPN to a small VPS.

<!--more-->

## Goals
The goals I had in mind are
* Create an isolated DMZ network that contains all publicly available hosts
* Assign every machine in my DMZ network a publicly reachable IPv6 address
* Still support IPv4 connections

## Solution Overview
My solution is shown in the network diagram below. To summarize, I decided to build the DMZ by adding a switch that connects all DMZ hosts to the DMZ network. A DMZ router, which essentially is a linux host, connects the DMZ to the outer world and also isolates it from my home network. All connections from and to the DMZ are made through a small VPS connected via a VPN. The VPS is the destination for a set of IPv6 subnets and delegates these subnets through the VPN tunnel into the DMZ. The subnets originate from a 6in4 tunnel, which complicates the solution a bit as we will see later.

![network diagram of solution]({{"/assets/images/ipv6-prefix-delegation.svg"| relative_url}} "Network Diagram of Solution")

## Scope of Article
There are many topics involved in realizing the proposed solution. I will not cover all foundations but focus on the following aspects:
* VPN setup using Wireguard
* Routing configuration for 6in4 tunnel
* Routing in DMZ router
* Prefix delegation in DMZ router

I assume that the VPS and the DMZ router are linux boxes that already have IPv4 and IPv6 connectivity.

## Routing for 6in4 Tunnel
Using a 6in4 tunnel even if there is already native IPv6 connectivity means that there are two possible routes for IPv6 traffic. It is crucial to choose the correct route because each route will only accept traffic from expected networks. The router of the VPS provider does not accept traffic having a source address out of the 2001:db8:0:98::/64 subnet. The 6in4 router does only accept traffic from the 2001:db8:0:2::/63 subnet.

First, we extend the interface definition of our native network interface `ens3` by some blackhole rules for the 6in4 subnets. The purpose of these rules is to provide fallback rules that discard traffic to these subnets if the 6in4 tunnel or the VPN tunnel breaks down. This is reasonable because as soon as there is no more direct route via a tunnel, these subnets can never be reached and the traffic should not be routed through the public internet where it will be discarded anyway. The high metric ensures that these fallback routes will only be used if there is no better route available.




A [wiki entry](https://www.thomas-krenn.com/en/wiki/Two_Default_Gateways_on_One_System) about so-called source routing describes how to handle routing on multi-homed hosts. In short, there are two routing tables. One table contains the routes for the native connection and one table contains the routes for the 6in4 connection. There are rules on when to use which table. This type of configuration is only necessary on the VPS. We will see the reason for this when configuring the VPN later.

We add an additional routing table `he_ipv6` by adding the line `1 he_ipv6` to the routing table file `/etc/iproute2/rt_tables`. We populate the routing table with a default route and also specify the source IP to be used when communicating with the 6in4 gateway. The remaining statements introduce rules for selecting the new routing table. Basically, the table shall be used if a packet from or to a subnet of the 6in4 tunnel shall be routed.

{% highlight conf %}
auto he-ipv6
iface he-ipv6 inet6 v4tunnel
  address 2001:db8:0:123::2
  netmask 64
  endpoint 203.0.113.1
  local 198.18.98.6
  ttl 255

  # Default gateway
  post-up ip route add 2001:db8:0:123::/64 dev he-ipv6 src 2001:db8:0:123::2 table he_ipv6
  post-up ip route add default via 2001:db8:0:123::1 dev he-ipv6 table he_ipv6
  pre-down ip route del 2001:db8:0:123::/64 dev he-ipv6 src 2001:db8:0:123::2 table he_ipv6
  pre-down ip route del default via 2001:db8:0:123::1 dev he-ipv6 table he_ipv6

  # Rules for selecting special routing table
  post-up ip -6 rule add from 2001:db8:0:123::2/128 table he_ipv6
  post-up ip -6 rule add from 2001:db8:0:2::/64 table he_ipv6
  post-up ip -6 rule add from 2001:db8:0:3::/64 table he_ipv6
  post-up ip -6 rule add to 2001:db8:0:123::2/128 table he_ipv6
  post-up ip -6 rule add to 2001:db8:0:2::/64 table he_ipv6
  post-up ip -6 rule add to 2001:db8:0:3::/64 table he_ipv6
  pre-down ip -6 rule del from 2001:db8:0:123::2/128 table he_ipv6
  pre-down ip -6 rule del from 2001:db8:0:2::/64 table he_ipv6
  pre-down ip -6 rule del from 2001:db8:0:3::/64 table he_ipv6
  pre-down ip -6 rule del to 2001:db8:0:123::2/128 table he_ipv6
  pre-down ip -6 rule del to 2001:db8:0:2::/64 table he_ipv6
  pre-down ip -6 rule del to 2001:db8:0:3::/64 table he_ipv6
{% endhighlight %}

You can check the contents of the routing table with `ip -6 route show table he_ipv6` and the available rules with `ip -6 rule show`.

Last, we extend the interface definition of our native network interface `ens3` by some blackhole rules for the 6in4 subnets. The purpose of these rules is to provide fallback rules that discard traffic to these subnets if the 6in4 tunnel or the VPN tunnel breaks down. This is reasonable because as soon as there is no more direct route via a tunnel, these subnets can never be reached and the traffic should not be routed through the public internet where it will be discarded anyway. The high metric ensures that these fallback routes will only be used if there is no better route available.

{% highlight conf %}
iface ens3 inet6 static
  address 2001:db8:0:98::1
  netmask 64
  gateway fe80::1

  # sink routes for 6in4 subnets (just in case the tunnel breaks down)
  post-up ip -6 route add unreachable 2001:db8:0:123::/64 metric 2048
  post-up ip -6 route add unreachable 2001:db8:0:2::/64 metric 2048
  post-up ip -6 route add unreachable 2001:db8:0:3::/64 metric 2048
  pre-down ip -6 route del unreachable 2001:db8:0:123::/64 metric 2048
  pre-down ip -6 route del unreachable 2001:db8:0:2::/64 metric 2048
  pre-down ip -6 route del unreachable 2001:db8:0:2::/64 metric 2048
{% endhighlight %}

To make the routing work, the usual configuration flags have to be set. Please note that this disables automatic IPv6 configuration on the whole system. If you still need this (e.g. because you do not want to statically configure the IPv6 connection in your home network), you have to re-enable this by `echo 2 > /proc/sys/net/ipv6/conf/eth0/accept_ra` e.g. in a pre-up command for the network interface.
{% highlight conf %}
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
{% endhighlight %}

## VPN Setup using Wireguard
First of all, we have to generate the cryptographic keys to be used in the VPN configuration
* Generate a preshared key via `wg genpsk > vps_dmz.psk`
* Generate keys for VPS and DMZ router via `wg genkey | tee privatekey | wg pubkey > publickey`

Next, we create the VPS configuration `/etc/wireguard/wg-dmz.conf`. There are a few statements in the configuration shown below that are beyond what popular tutorials show. The `Table` configuration specifies that routes shall be added to the routing table `he_ipv6`. This is crucial because the routing of the tunnel subnets must not interfere with the routing of the VPS subnet as motivated before. The `PostUp` statements add rules that tell the routing engine to look into the `he_ipv6` table if one of the VPN or DMZ subnets is the source or destination of a connection. The `PreDown` statements remove these rules when the VPN connection is closed. The `AllowedIPs` have to contain all subnets to be routed through VPN because a route will be created for every of these subnets.

{% highlight conf %}
[Interface]
Address = 192.168.3.1/30
Address = 2001:db8:0:3::1/64
Table = he_ipv6
SaveConfig = false
ListenPort = 51820
PrivateKey = PRIVATE_KEY_VPS

PostUp = ip -4 rule add from 192.168.2.0/24 table he_ipv6
PostUp = ip -4 rule add from 192.168.3.0/30 table he_ipv6
PostUp = ip -4 rule add to 192.168.2.0/24 table he_ipv6
PostUp = ip -4 rule add to 192.168.3.0/30 table he_ipv6
PostUp = ip -6 rule add from 2001:db8:0:2::/64 table he_ipv6
PostUp = ip -6 rule add from 2001:db8:0:3::/64 table he_ipv6
PostUp = ip -6 rule add to 2001:db8:0:2::/64 table he_ipv6
PostUp = ip -6 rule add to 2001:db8:0:3::/64 table he_ipv6

PreDown = ip -4 rule del from 192.168.2.0/24 table he_ipv6
PreDown = ip -4 rule del from 192.168.3.0/30 table he_ipv6
PreDown = ip -4 rule del to 192.168.2.0/24 table he_ipv6
PreDown = ip -4 rule del to 192.168.3.0/30 table he_ipv6
PreDown = ip -6 rule del from 2001:db8:0:2::/64 table he_ipv6
PreDown = ip -6 rule del from 2001:db8:0:3::/64 table he_ipv6
PreDown = ip -6 rule del to 2001:db8:0:2::/64 table he_ipv6
PreDown = ip -6 rule del to 2001:db8:0:3::/64 table he_ipv6

[Peer]
PublicKey = PUBLIC_KEY_DMZ
PresharedKey = PSK
AllowedIPs = 192.168.2.0/24, 192.168.3.0/30, 2001:db8:0:2::/64, 2001:db8:0:3::/64
PersistentKeepalive = 25
{% endhighlight %}

The client side VPN configuration `/etc/wireguard/wg-vps.conf` is a bit simpler. This time, we do not want Wireguard to create any routes, so we disable this via the `Table` statement. Our goal now is to route all traffic of the DMZ router and all hosts through the VPN. To achieve this, we add default routes for IPv4 and IPv6 with a sufficiently low metric to take precendence of the default route given by the connection to the home router. To avoid disconnecting the VPN connection itself (obviously it cannot be routed through itself), we have to add an additional route that forces the VPN connection through the home router. It is beneficial to use a dedicated IPv6 address for the connection to the VPS to simplify firewall rules in the case that the DMZ would like to access the VPS for other purposes than VPN.

{% highlight conf %}
[Interface]
Address = 192.168.3.2/30, 2001:db8:0:3::2/64
PrivateKey = PRIVATE_KEY_DMZ
SaveConfig = false
Table = off

PostUp = ip -4 route add default via 192.168.3.1 dev wg-vps metric 128
PostUp = ip -6 route add 2001:db8:0:98::2/128 via fe80::1 dev eth0
PostUp = ip -6 route add default via 2001:db8:0:3::1 dev wg-vps metric 512

PreDown = ip -4 route del default via 192.168.3.1 dev wg-vps metric 128
PreDown = ip -6 route del 2001:db8:0:98::2/128 via fe80::1 dev eth0
PreDown = ip -6 route del default via 2001:db8:0:3::1 dev wg-vps metric 512

[Peer]
PublicKey = PUBLIC_KEY_VPS
PresharedKey = PSK
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [2001:db8:0:98::2]:51820
PersistentKeepalive = 25
{% endhighlight %}

In distributions using systemd, the VPN can be started with `systemctl start wg-quick@wg-dmz` and started automatically with `systemctl enable wg-quick@wg-dmz`. For Alpine Linux, there does not seem to be such a nice solution. Instead, I copied the [init script](https://gitweb.gentoo.org/repo/gentoo.git/tree/net-vpn/wireguard-tools/files/wg-quick.init?id=ca76f01b51e1fe77cbd85e0ff15892209b6624bc&h=master) of Gentoo, which works basically in the same way.

To make the routing of IPv4 traffic to the internet work, we have to provide NAT to the VPN clients. A nftables rule set can provide this:
{% highlight conf %}
table ip nat {
	chain postrouting {
		type nat hook postrouting priority 100; policy accept;
		ip saddr 192.168.2.0/24 oif ens3 snat 198.18.98.6
		ip saddr 192.168.3.0/24 oif ens3 snat 198.18.98.6
	}
}
{% endhighlight %}

After these steps, you should be able to ping the VPS from the DMZ router and vice versa. Additionally, you should be able to ping hosts in the internet.

## DMZ Isolation
Up to now, the DMZ is no DMZ yet. The major characteristic of a DMZ is that it is accessible from the outside but cannot access internal networks outside of the DMZ. In our case, this is the home network. Therefore, we add filter rules for the network interface `eth0` that connects the DMZ router to the home network. Because all connections shall be made through the VPN, the only really necessary connection is the connection to the VPS in order to establish the VPN.

However, there is still some other necessary traffic. One common rule is that the DMZ is allowed to answer to traffic from other networks. This covers answers to TCP, ICMP and ICMPv6. For ICMPv6, some further message types are required to make autoconfiguration and proper status reports work. All remaining traffic leaving the `eth0` interface shall be filtered.

In addition, [people](https://keremerkan.net/posts/wireguard-mtu-fixes/) are reporting problems when routing traffic from a subnet through a wireguard tunnel. Somehow, the [path MTU discovery](https://en.wikipedia.org/wiki/Path_MTU_Discovery) does not work, which potentially breaks TCP connections. Quickfixes are explicitly setting the MTU of an host in the DMZ to 1412 (which is the MTU of the wireguard tunnel) or [let a firewall rule adjust the maximum segment size of TCP packages](https://wiki.nftables.org/wiki-nftables/index.php/Mangle_TCP_options), which effectively leads to smaller packages.

{% highlight conf %}
table inet filter {

  chain forward {
    
    type filter hook forward priority 0; policy accept;

    oifname wg-vps tcp flags syn tcp option maxseg size set rt mtu \
    comment "Fix TCP MTU over wireguard interface."

  }

  chain output {

    type filter hook output priority 0; policy accept;

    oifname eth0 ip6 daddr 2001:db8:0:98::2 udp dport 51820 accept \
    comment "Accept outgoing wireguard traffic to eth0"

    oifname eth0 ct state { established, related } accept \
    comment "Accept established traffic to eth0"

    oifname eth0 icmp type {echo-reply, destination-unreachable, source-quench, time-exceeded, parameter-problem, timestamp-reply, info-reply} accept \
    comment "Accept outgoing echo-reply to eth0"

    oifname eth0 icmpv6 type {destination-unreachable, packet-too-big, time-exceeded, parameter-problem, echo-reply, nd-router-advert, nd-router-solicit, nd-neighbor-advert, nd-neighbor-solicit} accept \
    comment "Accept outgoing echo-reply, error reporting and NDP to eth0"

    oifname eth0 drop \
    comment "Drop all remaining traffic to eth0"

  }

}
{% endhighlight %}

As can be seen from the rules, DNS traffic is also not allowed. This is no issue because no DNS is necessary to establish the VPN connection. If you would like to use a domain name instead of an ip address as endpoint, you have to allow DNS traffic as well.

## Prefix Delegation on DMZ Router
In order to only have one service for assigning IPv4 and IPv6 configurations, I used dnsmasq. The configuration is done stateless, which means that clients use SLAAC for IPv6 assignment and use DHCPv6 for getting other configurations such as the DNS server.

{% highlight conf %}
domain-needed
bogus-priv
interface=eth1
expand-hosts
domain=some.domain.com
dhcp-range=192.168.2.50,192.168.2.150,24h
dhcp-range=2001:db8:0:2::, ra-stateless
dhcp-option=option6:dns-server,[::]
conf-dir=/etc/dnsmasq.d/,*.conf
{% endhighlight %}

## Final Remarks
Actually, the configuration is quite simple and works surprisingly well. The only real source of complexity is the 6in4 tunnel setup that requires fiddling around with two routing tables and multiple rules for handling them. If your provider assigns you a properly routed subnet with a prefix size of at most 63, configuration becomes much simpler.

Please also consider some further firewall rules to protect your DMZ hosts and limit access to necessary services. Using IPv6, these hosts are publicly accessible.