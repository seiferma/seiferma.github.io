---
id: 208
title: Adding Additional IPv6 Address to Netcup VPS
last_modified_at: 2021-04-09T16:30:00+02:00
permalink: /2021/04/09/adding-additional-ipv6-address-to-netcup-vps
categories:
  - Configuration
---
Most VPS providers provide a full /64 IPv6 subnet to a VPS. However, some do not route this subnet but switch it. Therefore, some additional work is required to get the VPS to listen to an additional IPv6 address.

<!--more-->

It is important to limit the time the additional IP address shall be used as source address (`preferred_lft`) to zero to ensure that still the primary address is used for outgoing connections.

{% highlight conf %}
iface ens3 inet6 static
  address 2001:db8:0:98::1
  netmask 64
  gateway fe80::1

  # additional IPv6 address
  post-up ip -6 addr add 2001:db8:0:98::2/128 dev ens3 preferred_lft 0
  post-up ip -6 neigh add proxy 2001:db8:0:98::2 dev ens3
  pre-down ip -6 addr del 2001:db8:0:98::2/128 dev ens3
  pre-down ip -6 neigh del proxy 2001:db8:0:98::2 dev ens3
{% endhighlight %}