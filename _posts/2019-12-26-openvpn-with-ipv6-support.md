---
id: 206
title: OpenVPN with IPv6 Support
last_modified_at: 2019-12-26T18:02:09+01:00
permalink: /2019/12/26/openvpn-with-ipv6-support/
categories:
  - Configuration
---
OpenVPN is a commonly used, easy to set up VPN solution. A huge benefit is that it uses common SSL/TLS encryption and common transport protocols. Therefore, blocking it is pretty hard without affecting usual surfing. Getting it to work with IPv6 support is, however, not trivial. This article explains how to set up a dual-stack supporting OpenVPN gateway VPN.

<!--more-->

This manual is based on another [blog post](https://www.bjoerns-techblog.de/2017/07/openvpn-und-ipv6/) that covers IPv6 specific configuration options. We will not cover the setup of a certificate authority but refer to [another tutorial](https://www.howtoforge.com/tutorial/how-to-install-openvpn-server-and-client-with-easy-rsa-3-on-centos-8/) about that. Instead, we assume that certificates are already available.

## Server Configuration

First, we start with the OpenVPN configuration:

```
dev tun
proto tcp
proto tcp6
port 443
ca /etc/openvpn/server/gateway/ca.pem
cert /etc/openvpn/server/gateway/server.pem
key /etc/openvpn/server/gateway/server.key
dh /etc/openvpn/server/gateway/dh.pem
user openvpn
group openvpn
server 10.254.255.0 255.255.255.0
server-ipv6 XXXX:XXXX:XXXX:XXXX:ffff:ffff:ffff:0/112
persist-key
persist-tun
verb 0
client-to-client
push "redirect-gateway def1 bypass-dhcp"
push "redirect-gateway ipv6"
push "route-ipv6 XXXX:XXXX:XXXX:XXXX::/64"
push "dhcp-option DNS 194.150.168.168"
push "dhcp-option DNS6 2001:1608:10:25::1c04:b12f"
log-append /var/log/openvpn
comp-lzo

script-security 2
learn-address "/usr/bin/sudo -u root /etc/openvpn/ndp-proxy-update.sh"

keepalive 10 120

cipher AES-256-CBC
auth SHA256
tls-auth /etc/openvpn/server/gateway/ta.key 0
```

Most of the configuration is pretty straight forward. The only uncommon part is the `learn-address` configuration. It is necessary bridge the VPN subnet (/112) and the assigned IPv6 subnet (/64). It has to be called via sudo because the contained commands require rot permissions. The script looks as follows (copied from [Björns Techblog](https://www.bjoerns-techblog.de/2017/07/openvpn-und-ipv6/)):

```bash
#!/bin/bash

action="$1"
addr="$2"
pubif="eth0"

if [[ "${addr//:/}" == "$addr" ]]
then
    echo "Ignoring learning request because of IPv4 address $addr."
    # not an ipv6 address
    exit
fi

echo "Learning $action $addr -&gt; $pubif"

case "$action" in
    add)
        ip -6 neigh add proxy ${addr} dev ${pubif}
        ;;
    update)
        ip -6 neigh replace proxy ${addr} dev ${pubif}
        ;;
    delete)
        ip -6 neigh del proxy ${addr} dev ${pubif}
        ;;
esac
```

The script has to be executable. In addition, you have to configure sudo to allow executing it without a password prompt for the openvpn user:

```
openvpn ALL=(ALL:ALL) NOPASSWD: /etc/openvpn/ndp-proxy-update.sh
```

In order to get the networking running, you first have to enable IP forwarding. It is important to note that this configuration disables IPv6 auto configuration. Therefore, you have to make sure that you have defined a gateway for your IPv6 connection or add a default route manually. The following configuration can be placed in the file `/etc/sysctl.d/50-openvpn.conf`:

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.all.proxy_ndp=1
```

The iptables firewall should be configured in the following way (first IPv4, IPv6 afterwards):

```
iptables -t nat -A POSTROUTING -s 10.254.255.0/24 -o eth0 -j SNAT --to-source XXX.XXX.XXX.XXX
```

```
iptables -t filter -A FORWARD -s XXXX:XXXX:XXXX:XXXX:ffff:ffff:ffff:0/112 -j ACCEPT
iptables -t filter -A FORWARD -d XXXX:XXXX:XXXX:XXXX:ffff:ffff:ffff:0/112 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -t filter -A FORWARD -d XXXX:XXXX:XXXX:XXXX:ffff:ffff:ffff:0/112 -p ipv6-icmp -j ACCEPT
```

## Client Configuration

The client configuration is much simpler and can be used on desktop and mobile installations. `XXX` should be replaced with the hostname of the VPN server.

```
client
dev tun

proto tcp6
remote XXX 443

resolv-retry infinite
nobind
persist-key
persist-tun

<ca>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
....
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
....
-----END PRIVATE KEY-----
</key>

key-direction 1
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
....
-----END OpenVPN Static key V1-----
</tls-auth>

verify-x509-name XXX name

verb 3
comp-lzo

keepalive 10 120

cipher AES-256-CBC
auth SHA256
```

## Troubleshooting

Despite of the correct configuration mentioned above, I could not manage to get IPv6 connectivity working. After trying a few things, I recognized that my hoster (Ionos) does not allow to assign IPv6 subnets but only individual IP addresses to hosts. After assigning a few IP addresses of the VPN clients to the server, the connectivity worked as expected.