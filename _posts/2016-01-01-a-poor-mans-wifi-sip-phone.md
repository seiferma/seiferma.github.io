---
id: 51
title: 'A Poor Man&#8217;s Wifi SIP Phone'
last_modified_at: 2016-01-01T19:56:50+01:00
permalink: /2016/01/01/a-poor-mans-wifi-sip-phone/
categories:
  - Configuration
tags:
  - android
  - csipsimple
  - regular expression
  - sip
  - voip
  - wifi
---
After replacing my old FritzBox, I needed a SIP phone that handles at least two accounts, allows settings dialing rules and connects via Wifi. Supporting two SIP accounts is important because I use two providers with different pricings. Unfortunately, I could not find a cheap and available phone that fulfills these requirements. Therefore, I decided to build my own wifi SIP phone. <!--more-->All you need for building this phone is

  * old Android smartphone (>= Android 1.6)
  * CSipSimple app ([Google Play Store](https://play.google.com/store/apps/details?id=com.csipsimple) / [F-Droid](https://f-droid.org/repository/browse/?fdid=com.csipsimple))
  * SIP account(s)

I reset the phone and did not log in into the Google services. I do not need the Google services on this phone and do not want to make the data of my account available on this device. I connected the phone to the wifi network and installed the apk of the CSipSimple app. The Android integration and the call receiving feature has to be activated in the configuration wizard. Configuring the SIP accounts is straight forward. It is, however, important to active the STUN server in the application preferences as explained in [the troubleshooting section](https://code.google.com/p/csipsimple/wiki/FAQ#The_other_party_can_hear_me_but_I_can_not_hear_them) of the app's documentation if NAT is present. This is usually the case if the phone uses an IPv4 connection to your provider via a router or cellular network. For local accounts such as FritzBox accounts, this setting can be disabled per account in the account settings.

**Update:** I used a VPN connection to reach the Voip Server. I had to ensure that no NAT is involved. So, in the iptables configuration of the VPN endpoints, I had to remove the rules that masquerade IPs and use simple forwarding rules instead:  
```bash
iptables -I FORWARD -i tun0 -j ACCEPT
iptables -I FORWARD -i eth0 -j ACCEPT
```

Up to here, this was very easy and actually did not require a tutorial. The more interesting part is about automatically selecting the appropriate provider for each call. CSipSimple provides [filter rules](https://code.google.com/p/csipsimple/wiki/UsingFilters) to achieve this. Filter rules apply after the phone number has been dialed in the Android dialer and a selection dialog for the available providers appears. The selection criteria for my scenario is as follows:

  * Provider A shall be used for fixed network calls
  * Provider B shall be used for cellular network calls and calls in foreign countries

I added _directly call_ rules to the accounts that use regular expressions to match the phone number. The regular expressions according to the [German telephony network](https://de.wikipedia.org/wiki/Telefonvorwahl_%28Deutschland%29) are as follows:

  * Cellular network (country code is 49 and area code starts with 15, 16, or 17)  
    `^(((\+|00)49)|0)1[5-7][0-9]+`
  * Fixed network (country code is 49 and area code does not start with 1 or 900)  
    `^(((\+|00)49)|0)(([2-8][0-9]+)|(9(([1-9][0-9])|([0-9][1-9]))))[0-9]+`
  * Foreign countries (area code exists and is not 49)  
    `^((\+|00)(([0-35-9][0-9])|(4[0-8])))[0-9]+`

Additionally, I added a _rewrite_ rule that adds the country and area code to every phone number that does not include one or both of these. The rewrite rule has to be placed before the other rules and uses the regular expression `^[1-9][0-9]+` to find phone numbers without country (00xx or +xx) or area code (0xx). The according action is to _prefix_ the number by `+49721`, which es the country code of Germany and the area code of Karlsruhe. This rule allows to omit the area code for calls to the same area. This memes the behavior of a real wired phone connected to a local service provider.

With this setup, using the phone is very intuitive as all complexity of VOIP is hidden behind the Android stock dialer. To make using the phone even more comfortable, I connected my CardDav account to the phone that makes my contacts available.