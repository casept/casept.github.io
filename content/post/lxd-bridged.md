+++
title = "Using a bridged LXD network"
date = "2018-08-25T20:22:50+02:00"
categories = ["linux", "lxd", "networking"]
tags = ["linux", "lxd", "networking"]
description ="Configuring LXD to expose the containers directly on the network"
author = "casept"
+++

The regular network configuration created when simply running `lxd init --auto` is quite simple and usable. Unfortunately, it has one major drawback - NAT. Because LXD creates it's own private network you have to deal with port forwarding, and some services (`pi-hole` for example) take issue with not being able to use an address that is part of the network their clients are on. In addition, this setup makes the containers impossible to easily reach over IPv6.

The fix for these issues is fairly strightforward - expose the containers to the host's network directly, so that they get an IP from the router, and every device on the LAN can talk to them directly. The most common way to do it is to use a so-called "bridge interface". Basically, a bridge interface is an ethernet interface that simply passes along layer 2 traffic from the containers to the host network, with the container interface's MAC address intact. This way the router recognizes each as a different machine, and gives them IP addresses.
Note that this only works with ethernet, most wireless drivers/network setups don't support it.

Several tutorials already cover this setup, but none include the "gotchas" I encountered.

# Prerequisites

* A host running ubuntu 17.10 or later
* Root access
* No docker installed on the host (explanation below)

# Setup

First, back up all your containers, just to be safe.

Next, create a bridge interface using `netplan`. Essentially, `netplan` unifies network configuration in one place on newer ubuntu versions.

Create a new file under `/etc/netplan/99-lxd-bridge.yaml` (Note: __.yaml, not .yml!__ Also, the file has to be run once the physical interface has been configured, hence the `99-`.):

```yaml
network:
  version: 2
  renderer: networkd
  bridges:
    br0:
      dhcp4: yes
      interfaces:
        # Replace this with your ethernet interface
        - enp0s25
```

Then, apply the configuration:

```shell
sudo netplan --debug apply
```

You should see `br0` mentioned in the output.
Also, this command will kill your SSH connection if you're connected via the physical interface.

If you have configured some kind of static DHCP lease or anything else that depends on the physical interface's MAC address you might have to replace it with the bridge's MAC address.

Next, configure LXD to use your bridge:

```shell
# Answer "no" when prompted whether to create a new bridge
# When asked whether to use an existing bridge, answer "yes" and enter "br0" at the next prompt.
sudo lxd init
```

At this point you *should* be able to create a container using your bridge:

```shell
sudo lxc launch ubuntu:18.04 test
sudo lxc exec test bash
ifconfig
```

You *should* see an IPv4 address that's within your host's subnet, and several IPv6 addresses.
Unfortunately, in my case IPv4 simply didn't work - Everything worked as far as IPv6 was concerned, though.

After a frustrated morning I figured out the cause of my issues - docker! I didn't investigate further, but I'd guess that the way docker configures it's bridge doesn't sit well with LXD for some reason. I'm sure that it can be worked around, but I wanted to transfer docker into an LXD container anyways, so I simply removed it from the host.