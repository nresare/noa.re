---
title: "Configuring Hyperoptic IPv6 with your own router"
author: "Noa Resare"
date: "2023-04-30"
---

This page contains some details on how I configured IPv6 on my own internet gateway instead
of the [Nokia device](https://www.hyperoptic.com/wp-content/uploads/2020/10/Nokia-HA-140W-B-admin-manual.pdf)
provided by Hyperoptic.

# IPv4

First off it should be stated that regular IPv4 worked for me without any special tricks.
There is a DHCP-server that responds to DHCP Offer messages sent on the ethernet network
available from the [fibre termination hardware](https://guidessimo.com/document/1942159/nokia-ont-g-010g-r-quick-reference-manual-2.html)
with the only caveat being that by default the IP address that the remote DHCP server hands
out is a private address that gets translated by a [Carrier-grade NAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT)
device before reaching the public internet.

# IPv6

IPv6 connectivity, however, was another matter. The Hyperoptic router hands out publicly
routable IPv6 addresses to clients via the
[SLAAC](https://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration)
mechanism and there was a
[Hyperoptic help page](https://www.hyperoptic.com/faq/posts/static-ip-addresses/) that
gave some information about how to configure your third-party router, but still it
would not work for me at first.

The details below is based on my setup, Ubuntu Linux 22.04, but hopefully it contains enough
detail to be useful in other contexts as well.

## Reconfigure MAC

I found the first clue needed to configure IPv6 on
[Naz Markuta's excellent write-up](https://markuta.com/pfsense-ipv6-hyperoptic/)
with instructions on how to configure pfSense firewalls with Hyperoptic, the IPv6
configuration will only succeed for devices with the same
[MAC address](https://en.wikipedia.org/wiki/MAC_address) as the Nokia router provided
by Hyperoptic. So the first step is to configure the MAC address of the ethernet card
connecting to Hyperoptic.

I figured out the MAC address of the external interface of the provided router by connecting
it to a computer and capturing its network traffic, but I believe the same information is
available in the admin interface of the router.

To change the MAC on the currently running system, I used
`ifconfig enp1s0 down && ifconfig enp1s1 hw ether NOKIA_MAC && ifconfig enp1s0 up` (you will
need to substitute your ethernet device for `enp1s0`) and to make the change permanent I added
a file `/etc/systemd/network/enp1s0.link` with the content
```
# IPv6 from Hyperoptic neeed the mac address from
# their hardware to respond to DHCPv6 requests
[Match]
# This is the original mac address of ROUTER's enp1s0
MACAddress=00:11:22:33:44:55

[Link]
# This is the address from the Nokia hardware
MACAddress=55:44:33:22:11:00
NamePolicy=kernel database onboard slot path
```

## Configure networkd to act as a DHCPv6 client

networkd, part of systemd, supports all the technologies needed: DHCPv6,
responding to messages to configure other clients using the SLAAC mechanism,
emitting and receiving ICMPv6 Router Advertisement (RA) messages.
However, configuring was not very straight forward.

From the right MAC address, Hyperoptic will respond to DHCPv6 requests for
Prefix Delegation, handing out a publicly routable prefix with a 56 bit netmask. However,
a request for a non-temporary address will yield a `NoAddrsAvail` response, indicating
that no addresses are available.

In other words, the DHCPv6 Solicit message will need to contain the
Identity Association for Prefix Delegation (IA_PD) option but not the Identity Association for
Non-temporary Address (IA_NA) option. The way to configure systemd to do this is to, somewhat
counter-intuitively, enable prefix delegation on the inner interface. My network configuration
files look like this:

```
$ cat /etc/systemd/network/enp1s0.network
[Match]
# This is the interface connected to the Hyperoptic mediaconverter
Name=enp1s0

[Network]
# This enables the DHCP client for both IPv4 and IPv6
DHCP=yes

[DHCPv6]
# This setting ensures that the DHCPv6 client will start before a Router Advertisment
# is being received. This is needed because it seems Hyperoptic only sends RA messages
# when the Prefix Delegation is reserved.
WithoutRA=solicit
# We don't want to attempt to Request a non-temporary Address
UseAddress=no
```

```
$ cat /etc/systemd/network/enp2s0.network
# managed by puppet
[Match]
# This is the interface connected to the internal network
Name=enp2s0

[Network]
# Hard coded IPv4 address
Address=192.168.168.1/24
# This setting will enable the IA_PD option on the external interface and set this interface
DHCPv6PrefixDelegation=yes
# This setting will send Router Adveritsement messages on the local network with the option
# to enable SLAAC autoconfiguration to allow other clients to configure IPv6 addresses with
# the delegated prefix.
IPv6SendRA=yes
```

## Receiving default route configuration

I have not exhaustively experimented with this, but it seems no Router Advertisement messages
are sent from Hyperoptic unless the Prefix Delegation has been reserved via DHCPv6.
Moreover, it seems it takes a fairly long time before the RA messages gets sent, I have
observed between 5 and 15 minutes. Maybe this is because the Router Solicitation messages
are blocked in some way and the Router Advertisement messages that gets sent is a periodic
resend.

It would be interesting to figure out why this is, since the provided Nokia router seems to
get its route set up faster than that, but for now I am just letting it take its time.

Hopefully this has been useful to someone. For comments or corrections please email me
at n (at) noa.re.
