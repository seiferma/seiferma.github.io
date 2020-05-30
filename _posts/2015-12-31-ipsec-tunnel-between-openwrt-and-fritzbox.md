---
id: 6
title: IPSec Tunnel between OpenWrt and FritzBox
last_modified_at: 2015-12-31T02:27:14+01:00
permalink: /2015/12/31/ipsec-tunnel-between-openwrt-and-fritzbox/
categories:
  - Configuration
tags:
  - dynamic dns
  - dyndns
  - fritzbox
  - fritzos
  - ipsec
  - openwrt
  - strongswan
---
In this article, I will show you how to connect an OpenWrt and a FritzBox via an IPSec VPN connection. As a result, hosts on the attached networks can reach each other via a secured connection.  
<!--more-->

# Scenario

I want to create a site-to-site VPN connection from my OpenWrt based router to a FritzBox. To be more specific: I want to be able to ping hosts in the FritzBox network from hosts of my OpenWrt network and vice versa. I have to install and configure IPSec on my OpenWrt router because this is the only tunnel protocol supported by the FritzBox. The setting is as follows:

  * Remote (FritzBox) 
      * FritzOS 6.30 (should work with >= 6.00 as well)
      * Remote Network: 192.168.0.0/24
      * Remote Hostname: remotesite.dyndns.org
  * Local (OpenWrt) 
      * OpenWrt Chaos Calmer 15.05
      * Local Network: 192.168.1.0/24
      * Local Hostname: localsite.dyndns.org

# FritzBox Setup

The VPN configuration of a FritzBox usually takes place in [an external desktop application](http://avm.de/service/vpn/uebersicht/) that produces a config file. This configuration can be imported via the web interface later. The configuration is easy and covered in [manuals provided by AVM](http://avm.de/service/fritzbox/fritzbox-3270/wissensdatenbank/publication/show/5_VPN-Verbindung-zwischen-zwei-FRITZ-Box-Netzwerken-einrichten/). Anyway, it is crucial to adjust the configuration file in order to be able to connect from the OpenWrt router later as pointed out by [other guidelines](https://layer9.wordpress.com/2010/07/28/ipsec-vpn-mit-strongswan-und-fritzbox-fon-wlan-7270-v3/). In our scenario, the configuration would look like this:

```
vpncfg {
	connections {
		enabled = yes;
		conn_type = conntype_lan;
		name = "Site2Site VPN";
		always_renew = no;
		reject_not_encrypted = no;
		dont_filter_netbios = yes;
		localip = 0.0.0.0;
		local_virtualip = 0.0.0.0;
		remoteip = 0.0.0.0;
		remote_virtualip = 0.0.0.0;
		remotehostname = "localsite.dyndns.org";
		localid {
			fqdn = "remotesite.dyndns.org";
		}
		remoteid {
			fqdn = "localsite.dyndns.org";
		}
		mode = phase1_mode_aggressive;
		phase1ss = "all/all/all";
		keytype = connkeytype_pre_shared;
		key = "wBN4HKbOL4SrbQf9MCa2HLWh6raiPi";
		cert_do_server_auth = no;
		use_nat_t = yes;
		use_xauth = no;
		use_cfgmode = no;
		phase2localid {
			ipnet {
				ipaddr = 192.168.0.0;
				mask = 255.255.255.0;
			}
		}
		phase2remoteid {
			ipnet {
				ipaddr = 192.168.1.0;
				mask = 255.255.255.0;
			}
		}
		phase2ss = "esp-all-all/ah-none/comp-all/pfs";
		accesslist = "permit ip any 192.168.1.0 255.255.255.0";
	}
	ike_forward_rules = "udp 0.0.0.0:500 0.0.0.0:500",
	"udp 0.0.0.0:4500 0.0.0.0:4500";
}
```

You now have to replace `phase1_mode_aggressive` with `phase1_mode_idp`. Otherwise, there will be no connection later. Even worse: there will be no error messages about incompatible settings.

# OpenWrt Setup

There are multiple IPSec implementations on OpenWrt. I sticked to StrongSwan in combination with Charon because the OpenWrt has [a pretty good article](https://wiki.openwrt.org/doc/howto/vpn.ipsec.basics) about it. I only had to install the package `strongswan-default`. I followed the configuration convention of the article and used the mentioned scripts. For my scenario, the following configuration was sufficient.

**/etc/config/ipsec**

```
config 'ipsec'
  option 'zone' 'vpn'
  option 'debug' '1'
  list 'listen' 'eth0.2'

config 'remote' 'Site2Site'
  option 'enabled' '1'
  option 'gateway' 'remotesite.dyndns.org'
  option 'exchange_mode' 'main'
  option 'local_identifier' '@localsite.dyndns.org'
  option 'remote_identifier' '@remotesite.dyndns.org'
  option 'authentication_method' 'psk'
  option 'pre_shared_key' 'wBN4HKbOL4SrbQf9MCa2HLWh6raiPi'
  list   'p1_proposal' 'pre_g2_aes_sha1'
  list   'tunnel' 'site2site_lan'

config 'p1_proposal' 'pre_g2_aes_sha1'
  option 'encryption_algorithm' 'aes128'
  option 'hash_algorithm' 'sha1'
  option 'dh_group' 'modp1024'

config 'tunnel' 'site2site_lan'
  option 'local_subnet' '192.168.1.0/24'
  option 'remote_subnet' '192.168.0.0/24'
  option 'p2_proposal' 'g2_aes_sha1'

config 'p2_proposal' 'g2_aes_sha1'
  option 'pfs_group' 'modp1024'
  option 'encryption_algorithm' 'aes128'
  option 'authentication_algorithm' 'sha1'
```

**/etc/init.d/ipsec**

I tried to use the script from the [wiki article](https://wiki.openwrt.org/doc/howto/vpn.ipsec.basics#ike_daemon) (backup on [Gist](https://gist.github.com/seiferma/55adb1ee73dd202a6acc/61aebdd8e02622e26d9d8221c2502821a908f268)), but experienced an error because of incorrect handling of hostnames:  
`Error: an inet prefix is expected rather than "remotesite.dyndns.org"`.

I replaced the line  
```
LocalGateway=`ip route get $RemoteGateway | awk -F"src" '/src/{gsub(/ /,"");print $2}'`
```
with

```bash
ip route get $RemoteGateway > /dev/null
if [ $? -eq 0 ]; then
  RemoteGatewayIp=$RemoteGateway
else
  RemoteGatewayIp=`nslookup $RemoteGateway|sed 's/[^0-9. ]//g'|tail -n 1|awk -F " " '{print $2}'`
fi
LocalGateway=`ip route get $RemoteGatewayIp | awk -F"src" '/src/{gsub(/ /,"");print $2}'`
```

To support dynamic IP addresses on the remote site, the ipsec starter can be used. It updates the configuration after a given time period if the IP addresses behind the hostnames in the configuration have changed. I replaced the line
```/usr/sbin/ipsec start```
with 
```/usr/sbin/ipsec starter --auto-update 65```

The script including all adjustments is available on [Gist](https://gist.github.com/seiferma/55adb1ee73dd202a6acc/212569052b31bef2d3def270d2db177a2d03241f) as well.

**/etc/hotplug.d/iface/100-ipsec**

[Hotplug scripts](https://wiki.openwrt.org/doc/techref/hotplug) are executed after changes in interfaces and networks. I use them to handle changes of the local WAN IP and to start the IPSec tunnel in general. If the state of the WAN interface (eth0.2) changes, the IPSec tunnel is restarted. This is not harmful as the tunnel would not work anyway after such a change. The script is based on the firewall script in the same folder and only considers up/down state changes and changes of the associated IP address.

```bash
#!/bin/sh

[ "$ACTION" = ifup -o "$ACTION" = ifupdate ] || exit 0
[ "$ACTION" = ifupdate -a -z "$IFUPDATE_ADDRESSES" -a -z "$IFUPDATE_DATA" ] && exit 0
[ "$DEVICE" = eth0.2 ] || exit 0

/etc/init.d/ipsec-custom restart
logger -t ipsec "Restarting ipsec tunnel due to $ACTION of $INTERFACE ($DEVICE)"
```

**/etc/firewall.user**

Every tutorial handles the firewall configuration in a differnt way. I prefer [a simple and pragmatic approach](https://www.mundhenk.org/blog/fritzbox-openwrt-vpn#Firewall) by using the user configuration. The firewall has to allow IPSec packets (UDP ports 500 and 4500, ESP) to reach the device. After the tunnel has been established, NAT has to be disabled for connections to the remote network. The LAN access is configured via the zone configuration in the next step.

```bash
iptables -A input_rule -p esp -j ACCEPT 
iptables -A input_rule -p udp --dport 500 -j ACCEPT 
iptables -A input_rule -p udp --dport 4500 -j ACCEPT
iptables -t nat -A postrouting_rule -d 192.168.8.0/24 -j ACCEPT
```

**Firewall Configuration**

According to the [firewall guideline](https://wiki.openwrt.org/doc/howto/vpn.ipsec.firewall) of the OpenWrt wiki, I created a new VPN zone, that rejects packets in the input and forwarding chain and accepts packets in the output chain. Additionally, I allowed forwarding from and to the LAN zone. The firewall script did not work for me, so I moved the remaining configuration into the firewall user script (see above).

# Results

After the setup, you can restart the router. You should see a message in the system log that indicates that IPSec has been restarted because of changes on the eth0.2 interface. You can use `ipsec status` via SSH to see if the connection has been established. The output should look like this

```
Security Associations (1 up, 0 connecting):
  ipsec-site2site_lan[12]: ESTABLISHED 25 minutes ago, 198.51.100.112[remotesite.dyndns.org]...203.0.113.56[localsite.dyndns.org]
  ipsec-site2site_lan{13}:  INSTALLED, TUNNEL, reqid 1, ESP SPIs: bad70c18_i fd15f52d_o
  ipsec-site2site_lan{13}:   192.168.1.0/24 === 192.168.0.0/24
```

Finally, you should be able to ping remote hosts.