---
layout: post
title: "Using OpenBSD's in-kernel WireGuard support to route some local hosts over a VPN"
---

This post is a follow-up to [Routing some local hosts over a WireGuard VPN on an OpenBSD router](/posts/wireguard-go-openbsd), which detailed how to use the [userland implementation of WireGaurd, `wireguard-go`](https://git.zx2c4.com/wireguard-go/about/) on OpenBSD to route a specific local host or subnet over a WireGuard VPN. I've finally upgraded to OpenBSD 6.8, so this post will cover how to use the new in-kernel WireGuard support from the `wg(4)` driver to accomplish the same goal.

These man pages were helpful:

* [`wg(4)`](https://man.openbsd.org/OpenBSD-6.8/wg)
* [`ifconfig(8)`](https://man.openbsd.org/OpenBSD-6.8/ifconfig)
* [`pf.conf(5)`](https://man.openbsd.org/OpenBSD-6.8/pf.conf)
* [`rdomain(4)`](https://man.openbsd.org/OpenBSD-6.8/rdomain)

# Prerequisites

You'll need a WireGuard keypair (public and private keys) for your OpenBSD router and a WireGuard public key from your VPN provider. You can generate your own private key and upload the public key to your VPN provider, or let them generate everything for you (for example, [Mullvad has a page to generate a configuration file](https://mullvad.net/en/download/wireguard-config/) that generates everything for you and also gives you the ability to upload a key).

You also of course need an OpenBSD router with PF. I tested these steps on OpenBSD 6.8.

# Configure the WireGuard interface

Create a new file called `/etc/hostname.wg2`, or `wg0`, `wg1`, etc. I'll use `wg2` for the rest of this post. Populate it with the following content, making the following replacements:

* Replace `<rdomain>` with the number of the routing domain you want to put the `wg` interface in (I use `2` to match the interface number)
* Replace `<private_key>` with your WireGuard private key
* Replace `<public_key>` with the public key of your WireGuard peer
* Replace `<endpoint>` with the IP address of your peer and `<port>` with the peer's listening port
* Replace `<local_ip>` with the IP address of your `wg` interface (from the `Address` line in a Mullvad configuration file)

```
description "WireGuard"
rdomain <rdomain>
wgkey <private_key>
wgpeer <public_key> wgendpoint <endpoint_ip> <port> wgaip 0.0.0.0/0
inet <local_ip> 255.255.255.255
!route -T 2 -n add -inet default -iface <local_ip>
!route -T 2 -n add <endpoint_ip> -gateway $(route -T 0 -n get default | grep gateway | awk '{print $2}')
```

If you want to restrict what source IP addresses traffic from your peer is allowed you have, you can optionally change the `0.0.0.0/0` value. If your local WireGuard IP address has a different subnet value, you should change the `255.255.255.255` netmask accordingly.

The `route -T 0 -n get default` command ensures that traffic to the WireGuard VPN peer is not routed over the VPN. The `grep` and `awk` commands extract your default route table's default gateway IP address and the full command adds a route to the WireGuard endpoint or peer IP via that default gateway.

Save the file and then set the proper permissions and bring up the new interface:

```
chmod 640 /etc/hostname.wg2
chown root:wheel /etc/hostname.wg2
sh /etc/netstart wg2
```

You should now be able to see some information about the interface and statistics about VPN traffic by running `ifconfig wg2`.

# Configure PF

These PF rules will match any incoming traffic from a local device with IP address `10.0.0.100`, route that traffic using the routing table associated with the alternate rdomain you created, and NAT the traffic to the IP address of your `wg` interface. This assumes you already have a PF macro called `int_if` that defines the interface that the traffic from the local device will be entering on. Add these lines to your `/etc/pf.conf` file:

```
match in on $int_if inet from 10.0.0.100 to any rtable 2
match out on wg2 inet from !(wg2:network) to any nat-to (wg2:0)
```

You can also route all the traffic on an interface (a physical or VLAN interface) over the WireGuard VPN. For example, if you want to route all traffic in VLAN 200 over the VPN, the first `match` rule would be:

```
match in on vlan200 from vlan200:network to any rtable 2
```

Save your changes and reload the ruleset to apply them:

```
pfctl -f /etc/pf.conf
```

Any local devices, interfaces, or subnets you matched with your PF rules should have all of their traffic routed over the WireGuard VPN now. You can [check for DNS leaks and confirm your external IP address with dnsleaktest.com](https://dnsleaktest.com). 

# DNS

Mullvad has a handy function where they [hijack all DNS traffic and route it to a Mullvad DNS server](https://mullvad.net/en/help/terms-service/), but your VPN provider may not do this for you. For good measure (or if you're not using Mullvad or another VPN provider that does this for you) you should do something like manually set the DNS server on your local device, or configure your DHCP server with the desired DNS server address for your local subnet. For example, if your OpenBSD router is your DHCP server and you want to configure Mullvad's DNS server for the entire 10.0.0.0/24 subnet, include something like this in `/etc/dhcpd.conf`:

```
subnet 10.0.0.0 netmask 255.255.255.0 {
    option routers 10.0.0.1;
    option domain-name-servers 193.138.218.74;
    range 10.0.1.100 10.0.1.199;
}
```

[The Mullvad DNS server address is available here](https://mullvad.net/en/help/dns-leaks/).

[If you use Firefox you may also want to disable DNS over HTTPS (DoH)](https://support.mozilla.org/en-US/kb/dns-over-https-doh-faqs#w_will-users-be-able-to-disable-doh).

# That's it!

Thanks for reading.

# Bonus

Here's a kind of gross script I quickly wrote to convert a Mullvad configuration file into `hostname.if` format. The first argument should be the path to the Mullvad configuration file and the second argument should be the routing domain number you plan to use. If you save it as `mullvad.sh` and make it executable, you can run it with a command like `./mullvad.sh mlvd.conf 2`:

```
#!/bin/sh

if [ -z "$1" -o -z "$2" ]; then
    printf "First argument should be a Mullvad config file and second argument a routing domain number.\\n"
    exit 1
fi

rdomain="$2"
wgkey=$(cat "$1" | grep PrivateKey | awk '{print $3}')
wgpeer=$(cat "$1" | grep PublicKey | awk '{print $3}')
wgendpoint=$(cat "$1" | grep Endpoint | awk '{print $3}' | sed 's/:/ /')
wgaip=$(cat "$1" | grep AllowedIPs | awk '{print $3}')
inet=$(cat "$1" | grep Address | awk '{print $3}')

printf \
"description \"WireGuard\"\\n\
rdomain ${rdomain}\\n\
wgkey ${wgkey}\\n\
wgpeer ${wgpeer} wgendpoint ${wgendpoint} wgaip ${wgaip}\\n\
inet ${inet}\\n\
!route -T ${rdomain} -n add -inet default -iface $(echo "$inet" | cut -d '/' -f 1)\\n\
!route -T ${rdomain} -n add $(echo "$wgendpoint" | cut -d ' ' -f 1) -gateway \$(route -T 0 -n get default | grep gateway | awk '{print \$2}')\\n"
```
