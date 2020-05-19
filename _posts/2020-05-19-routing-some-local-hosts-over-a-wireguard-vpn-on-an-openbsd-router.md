---
layout: post
title:  "Routing some local hosts over a WireGuard VPN on an OpenBSD router"
---

# Overview

I was inspired to write this by a friend who wanted to be able to automatically tunnel traffic from specific devices on their local network over a WireGuard VPN. A few other resources were helpful in writing this:

* [toying with wireguard on openbsd](https://flak.tedunangst.com/post/toying-with-wireguard-on-openbsd)
* [WireGuard on OpenBSD](https://blog.jasper.la/wireguard-on-openbsd.html)
* [OpenBSD Router : VPN](https://lipidity.com/openbsd/wireguard/)
* [`pf.conf` man page](https://man.openbsd.org/OpenBSD-6.6/pf.conf.5)
* [`rdomain` man page](https://man.openbsd.org/OpenBSD-6.6/rdomain.4)

# Prerequisites

You need a WireGuard configuration file from your VPN provider. For example, [Mullvad has a page to generate a config file here](https://mullvad.net/en/download/wireguard-config/). [There is an interesting post here about why Mullvad supports WireGuard](https://mullvad.net/en/blog/2017/9/27/wireguard-future/).

You also need an OpenBSD router/firewall with PF running. It shouldn't matter if the OpenBSD system is directly connected to the internet or not. These steps were tested successfully on version 6.6 of OpenBSD.

# Install WireGuard

This will install WireGuard from packages. [`wireguard-go` is a userland implementation of WireGuard written in Go](https://git.zx2c4.com/wireguard-go/about/). [`wireguard-tools` provides the `wg` and `wg-quick` command-line tools](https://git.zx2c4.com/wireguard-tools/about/). These steps will only use the `wireguard_go` daemon and the `wg` command.

```
pkg_add wireguard-go wireguard-tools
```

# Set up WireGuard to run as a non-root user

From what I can tell, [WireGuard merged a patch to support running as a non-root user](https://lists.zx2c4.com/pipermail/wireguard/2019-July/004308.html), as long as you set the MTU on the `tun` interface instead of relying on the WireGuard daemon to do it. The default install from OpenBSD packages does not set this up for us, so we'll set it up ourselves.

```
useradd -s /sbin/nologin -c "WireGuard" -d /var/empty _wireguard
```

Now you can set the `tun` interface you want WireGuard to use (we'll use `tun2` just because; you can use any unused `tun` device number you want, but the rest of the steps here will assume you're using `tun2`), set the user it will run as, and enable it.

```
rcctl set wireguard_go flags tun2
rcctl set wireguard_go user _wireguard
rcctl enable wireguard_go
```

# Set up the config file

This will put your WireGuard config file into `/etc/wireguard` and recursively set permissions that allow the new user you created to access it. This assumes the config file is named `mullvad-se12.conf` and is in your current working directory (you can copy it from wherever you downloaded it to).

```
mkdir /etc/wireguard
cp mullvad-se12.conf /etc/wireguard/client.conf
chown -R _wireguard:_wireguard /etc/wireguard
chmod -R 600 /etc/wireguard
```

For the config file to work with `wg` (instead of `wg-quick`, which doesn't allow for the level of customization we're doing here) **you have to delete both the `Address` and `DNS` lines from the `Interface` section**. Copy these values somewhere else for now since you'll need them later.

# Set up the network interface

Create the file `/etc/hostname.tun2` with the following contents. Replace `10.32.4.3` with the IPv4 address and `c00:bbbb:bbbb:bb01::1:7c21 128` with the IPv6 address, both from the `Address` line that you previously deleted from `/etc/wireguard/client.conf` (note that the `/32` in the IPv4 address is replaced with the equivalent in dot-decimal notation and the IPv6 address is separated from its prefix with a space instead of a forward slash).

```
description "WireGuard"
inet 10.32.4.3 255.255.255.255
inet6 fc00:bbbb:bbbb:bb01::1:7c21 128
mtu 1420
up
```

Bring up the interface.

```
sh /etc/netstart tun2
```

# Configure a routing table

This will configure routes in an alternate routing table that can be used for the traffic we want to be routed over the VPN. I initially also put the `tun2` interface in an alternate `rdomain`, which worked fine, but I don't think it's strictly required here and we can get away with just a dedicated routing table (we'll use PF later to select the routing table for specific incoming traffic).

We'll use an ID of `2` for the routing table but you can use any value up to `255` (the default routing table is ID `0`). Remember to replace `10.32.4.3` and `fc00:bbbb:bbbb:bb01::1:7c21` here with the actual addresses you configured in the `hostname.tun2` file!

```
route -T 2 -n add -inet default -iface 10.32.4.3
route -T 2 -n add -inet6 default -iface fc00:bbbb:bbbb:bb01::1:7c21
```

One more route is needed to ensure that traffic to the WireGuard VPN peer/server is not routed over the VPN. Replace `185.65.134.128` with the IP address from the `Endpoint` line in `/etc/wireguard/client.conf` (make sure to remove the `:51820` port specification) and replace `172.15.1.1` with your internet connection's default gateway. If your OpenBSD system is connected directly to the internet, this is the `gateway` value in the output of the `route -T 0 -n get default` command.

```
route -T 2 -n add 185.65.134.128 -gateway 172.15.1.1
```

# Start WireGuard

...gotta fix non-root.

# Apply the client configuration to the interface

```
wg setconf tun2 /etc/wireguard/client.conf
```

# Configure PF

* Add these lines to your `pf.conf` file **before** your other `nat-to` rule (assuming you have an existing `nat-to` rule for all egress traffic)
* Replace the IP address `10.0.0.100` with the private IP address of the local device you want to be routed over the VPN (you could also specify a range here)
* Replace `em1` with whatever interface your local device's traffic will be entering on
* Note that these will not match IPv6 traffic (maybe I'll fix that later)

```
match in on em1 inet from 10.0.0.100 to any rtable 2
match out on tun2 inet from !(tun2:network) to any nat-to (tun2:0)
```

* Reload pf configuration to apply changes

```
pfctl -f /etc/pf.conf
```

* Your local device with IP address `10.0.0.100` should have all of its traffic routed over the WireGuard VPN now!
* Read the DNS section to deal with DNS leaks

# DNS

* **Nothing in these instructions covers setting the DNS server on the local device! Without updating your local device's DNS settings it could still be sending DNS queries to your ISP, some other third party, or whatever your default DNS server is.**
* Set the local device's DNS server manually (set the local device's DNS server to the IP address from the `DNS` line that you noted before) or you could specify a custom DNS server in a DHCP reservation for the local device

# Diagram (could be improved with IPv6 info)

![wireguard_openbsd_router](https://user-images.githubusercontent.com/35312055/77459122-226add80-6df7-11ea-9899-b6863956636b.png)
