+++
title = "Setting up a wireguard server running on an OpenWRT router"
date = "2018-07-21T18:05:50+02:00"
categories = ["openwrt", "linux", "vpn", "wireguard"]
tags = ["openwrt" , "linux", "vpn", "wireguard"]
description = "Because I couldn't be arsed to debug OpenVPN"
author = "casept"
+++

# Motivation

I've been trying to find a decent DIY VPN solution for quite a while now.
They all seem easy to configure at first, but the weirdness and workarounds just kept piling up until I realized that I've been to busy cursing at $VPN_SOLUTION to get any sleep.

Wireguard is a relative newcomer to the scene, having been widely known for only about a year now.
It's approach to building a VPN is rather unique in that it doesn't force you to set up an entire PKI just to connect your phone to the VPN (looking at you, OpenVPN...). Instead, it only focuses on creating a secure tunnel between machines, leaving key managment and other such intricacies to you. This means that you're much more free in choosing the security vs. complexity tradeoff that applies in your particular case. After all, you probably just want to enjoy your Linux ISOs on the go, not run a F500 company's IT department.


Unfortunately, it's also not the most well-doumented piece of software out there. This post took a few days of scraping together information from way too many forum threads and blog posts to write. Even though the software is fairly new, I'm still surprised that nobody has done a (somewhat) comprehensive writeup of this before...

# Disclaimer

I'm not particularly experienced with networking or OpenWRT, so this setup took me quite a while to figure out and might be flawed in (not-so) subtle ways (I lost my internet connection several times while figuring out this setup). Implement at your own risk.

# Prerequisites

You'll need:

- A router with a recent-ish version of OpenWRT or LEDE installed
- A few megs of flash space on the router for packages and config
- SSH access to the router
- At least a /64 IPv6 prefix, if you want to use IPv6 tunnels. (I recommend getting a 6in4 tunnel from Hurricane Electric if your ISP doesn't offer native IPv6.)
- The client's network should also be IPv6-capable, or v6 tunneling won't work.
- A public IPv4 address (If your carrier doesn't do CGNAT you probably have one).

# Setup

### Installing packages

`ssh` into your router and install the needed packages:

```shell
opkg update
opkg install luci-proto-wireguard luci-app-wireguard wireguard kmod-wireguard wireguard-tools
```

Reboot your router now, as some models will give you trouble when creating the interface if you don't.

### Punching a hole in the firewall

You need to configure the builtin firewall so that the wireguard port is exposed:

```shell
uci add firewall rule
uci set firewall.@rule[-1].src="*"
uci set firewall.@rule[-1].target="ACCEPT"
uci set firewall.@rule[-1].proto="udp"
uci set firewall.@rule[-1].dest_port="1234"
uci set firewall.@rule[-1].name="Allow-Wireguard-Inbound"
uci commit firewall
/etc/init.d/firewall restart
```

### Creating and setting up a separate Wireguard firewall zone

We're going to place the wireguard interface in it's own firewall zone:

```shell
# Add the firewall zone
uci add firewall zone
uci set firewall.@zone[-1].name='wg'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].masq='1'
# Add the WG interface to it
uci set firewall.@zone[-1].network='wg0'
# Forward WAN and LAN traffic to/from it
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wg'
uci set firewall.@forwarding[-1].dest='wan'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wg'
uci set firewall.@forwarding[-1].dest='lan'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wg'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wan'
uci set firewall.@forwarding[-1].dest='wg'
uci commit firewall
/etc/init.d/firewall restart
```

This step is probably optional (you could just add the interface to the `lan` zone). I perform it here because it allows for finer-grained firewall rules in the future.

### Setting up the wireguard interface

In order for wireguard to work you have to generate a keypair for the server (like with SSH):

```shell
# Generate the private key, setting it to only be readable by the current user and group.
umask 077 && wg genkey > privkey
# Derive the public key from it
cat privkey | wg pubkey > pubkey
```

Now that we have the keypair we can begin initializing the interface:

```shell
# wg0 is the name of the wireguard interface, replace it if you wish.
uci set network.wg0="interface"
uci set network.wg0.proto="wireguard"
uci set network.wg0.private_key="$(cat privkey)"
# You may change this port to your liking, ports of popular services get through more firewalls.
# Just remember it for when you have to configure the firewall later.
uci set network.wg0.listen_port="1234"
uci add_list network.wg0.addresses='<A /64 (or greater) IPv6 subnet for client use>'
uci add_list network.wg0.addresses='<An IPv4 subnet for client use, in CIDR notation>'
# Save the changes
uci commit network
/etc/init.d/network reload
```

### Configuring the (Android) client

I only cover the Android client here because the desktop clients are (relatively well) documented on the project's site. On the other hand, the Android client required me poking at some IP fields for a while to figure it out (which is why I'll document it here).

  1. Install the app from [google play](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en_US) or [f-droid](https://f-droid.org/en/packages/com.wireguard.android/).

  2. Create a profile by clicking on the "+" and on "create from scratch".

  3. Think of a name for your connection (it's arbitrary).

  4. Hit the "generate" button to create a public/private keypair for your client.

  5. Set "DNS servers" to your DNS server (typically your router's private address).

  6. Enter "0.0.0.0/0, ::/0" into the "allowed IPs" field. This way, traffic to any IP will be forwarded through the tunnel.

  7. Enter "25" in the "persistent keepalive" field so that the connection doesn't get dropped by some NAT setups.

  8. Enter your router's public IP address and WG port into the "Endpoint" field in the format ip:port.

  9. Enter a subnet of the IPv4 subnet you configured on the server, followed by a comma and an IPv6 subnet (also a subnet of the IPv6 subnet you configured on the server). For example, if the IPv4 subnet is `10.14.0.0/16` and the IPv6 subnet is `eeee:1234:8790::/60` then a valid entry would be `10.14.0.3/32, eeee:1234:8790::/64`. Just make sure each client gets it's own range, or you might get some nasty collisions.

You'll also need to set the client up on your server:

```shell
# Change all occurences of "wireguard_wg0" to something else (like wireguard_wg1, wireguard_wg2 and so on) for subsequent clients after the 1st
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].public_key="<your client's pubkey>"
# Allow the client to forward traffic to any IP through the tunnel
uci set network.@wireguard_wg0[-1].route_allowed_ips="1"
uci add_list network.@wireguard_wg0[-1].allowed_ips="0.0.0.0/0"
uci add_list network.@wireguard_wg0[-1].allowed_ips="::/0"
# Enable sending of keepalive packets so NAT routers don't terminate the connection. WG recommends a value of 25.
uci set network.@wireguard_wg0[-1].persistent_keepalive='25'
# What you want your client to show up as in the UI
uci set network.@wireguard_wg0[-1].description='<client name>'
uci commit network
/etc/init.d/network reload
```

Repeat for all clients you wish to add.

You might also need to restart your router again (at least that was the case for me, before the restart WG didn't recognize any clients).

Aaand you're done! (Hopefully...)

#### References:

All these posts helped me tremendously in figuring out this setup:

  1. https://danrl.com/blog/2016/travel-wifi/ - extremely helpful post by the guy who implemented WG support in OpenWRT.

  2. https://wireguard.com - The WG project site.

  3. https://github.com/Chadster766/McWRT/wiki/Hurricane-Electric-Tunnelbroker-IPv6-Setup - Guide for setting up a 6in4 tunnel.

  4. https://danrl.com/blog/2017/luci-proto-wireguard/ - This post covers setting up the server, but doesn't go into lots of detail.

  5. https://wiki.openwrt.org/doc/uci/firewall#forwarding_ipv6_tunnel_traffic - Documents setting up forwarding for a tunnel.

  6. https://forum.lede-project.org/t/solved-wireguard-dns-leak-on-the-mobile-device/14640/9 - Helped me figure out what addresses to enter on the client
