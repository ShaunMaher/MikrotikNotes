![Product Logos: RouterOS, Wireguard and Android](Wireguard/RouterosPlusWireguardPlusAndroid.png)
# RouterOS + Wireguard + Android
## Overview
This document is still a work in progress.

TODO:
- [ ] Networking is broken if device is on home Wifi (i.e. same network as the Wireguard server)
- [ ] No IPv4 support
- [ ] Create a public DNS AAAA record for the RouterOS Peer
- [ ] Generally the language below is poor and could be improved
- [ ] Wireguard Peer screen shot contains the peer public key in the title bar

I'm trying to make all parts of my network IPv6 native.  All devices that
support it are dual stack IPv4 and IPv6.  My ISP provides an IPv6 /56 subnet
and, with a little tinkering, my mobile data provider will assign my device a
single IPv6 address.  Otherwise, both of my internet connections are behind
CG-NAT.

My internal home/lab networks are all IPv6 enabled (again, dual stack).

My goals are twofold:
* Secure remote access to services on my home network (without just allowing public internet access to said services)
* Route all traffic from my mobile device through my home ISP for privacy so
that, for example, a public open Wifi access point can be used without my data
being harvested.

## Prerequisites
### RouterOS 7
Mikrotik RouterOS 7 is required as it is the first version of RouterOS to
support Wireguard.

At the time of writing, none of my Mikrotik hardware devices support RouterOS 7.
To work around this, I have created a dedicated Wireguard RouterOS instance in a
VM.

### Android: Wireguard client
Get it from the Play Store.

### Accurate time
Wireguard complains if your peer clocks are a bit different.  Your mobile device
will probably keep itself up to date from the data network but your router might
benefit from having NTP configured and enabled.

```
/system ntp client
set enabled=yes
/system ntp client servers
add address=0.pool.ntp.org
```

## RouterOS 7 VM
If you are using an existing device on your network, you probably don't need anything in this section.  Please skip ahead.

### On your core RouterOS router (the one that gets an IPv6 address and prefix from your ISP)

You will need your core router to be able to assign a subnet (or prefix) to the
RouterOS VM instance.

Assuming your DHCP client configuration is creating an address pool called
"LAN" and your LAN interface is called "brLAN" (which for me is a bridge between
two physical interfaces):

![DHCPv6](Wireguard/DHCPv6-01.png)

```
/ipv6 dhcp-server
add address-pool=LAN interface=brLAN lease-time=1h name=LANv6
```

### On the RouterOS VM
![DHCPv6](Wireguard/DHCPv6-02.png)

```
/ipv6 dhcp-client
add add-default-route=yes interface=ether1 pool-name=LAN pool-prefix-length=72 request=prefix use-interface-duid=yes
```

Once it says "Status: bound", head back to your core router

### On your core RouterOS router
As super depressing as it is and as much as I hate to do it, we are going to be
hard coding some IPv6 IP addresses.  The DHCPv6 subnet we just assigned to the
VM is currently dynamic which means that it might change if the VM is restarted
(for example).  We will make it a static reservation on the DHCPv6 server (the
  core router) so that it becomes persistent.

![DHCPv6](Wireguard/DHCPv6-03.png)

```
/ipv6 dhcp-server binding
add address=2403:5800:3200:1403::/64 duid=0x5254008d284b iaid=1 life-time=1h server=LANv6
```

Your core router should already be configured to be very restrictive as to the
inbound IPv6 traffic it allows through to your LAN (where the VM resides).  You
need to make a small hole for the Wireguard traffic to come in from the internet
and make it to your RouterOS VM.

![DHCPv6](Wireguard/Firewall-01.png)![DHCPv6](Wireguard/Firewall-02.png)

```
/ipv6 firewall filter
add action=accept chain=forward comment="Wireguard listening port on isr2" dst-address=2403:5800:3200:1403:300::1/128 dst-port=13231 log=yes log-prefix=WG protocol=udp
```

## Wireguard Terminology
Before we get into the wireguard setup itself, I found some of the fields in
both the Android client and the RouterOS configuration to be a little ambiguous.

### RouterOS: peer
* Allowed Address(es): This is NOT a list of public IP addresses that are
allowed to connect.  This is NOT a list of LAN IP addresses that the peer is
allowed to connect to.  This IS a list of LAN IP addresses ON THE PEER that are
allowed to send packets and have packets sent back to them.  This makes no sense
if the peer is a single device on a public network with a single IP but, when
you look at the android client, it will make more sense.

### Android: Interface
* Addresses: Not to be confused with the "Allowed IPs" field in the "Peer"
section.  This is a private IP address that will be assigned to the wireguard1
virtual network interface.  This is essentially a pretend LAN IP for the device
that isn't really on a LAN.  This field should match the "Allowed Address(es)"
field on the peer on the RouterOS end.

## Addressing
We need to decide what virtual IP addresses to assign to the peer's wireguard
interfaces.

We can (though I gave this limited testing) use IP addresses from
the IPv6 private IP range.  The advantage here is that if our ISP changes our
IPv6 prefix we don't need to reconfigure our network.  The down side is that you
would need to setup static routes on (at least) your core router to send packets
to wireguard clients via the wireguard interface (which might be on another
RouterOS device).  https://dnschecker.org/ipv6-address-generator.php

The alternative, which is what I have implemented, is to use IPs out of the
prefix that was assigned to the wireguard1 interface from DHCPv6.  The advantage
being that everything on my network knows how to route that prefix already.

So, for me, my wireguard1 interface on my RouterOS VM is:
2403:5800:3200:1403:300::1 and I'm going to use 2403:5800:3200:1403:300::5 as
the first IP I assign to a device.

## Wireguard
When creating the Wireguard interface, leave the "Private Key" and "Public Key"
fields blank.  Keys will be generated when you save.
![RouterOS Wireguard Interface Settings](Wireguard/RouterOSWireguard.png)
![RouterOS Wireguard Peer Settings](Wireguard/RouterOSWireguardPeer.png)

## Android Client
![Android Client Settings](Wireguard/AndroidWireguard.png)
