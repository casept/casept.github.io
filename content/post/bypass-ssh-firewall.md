+++
title = "Bypassing pesky SSH-blocking firewalls"
date = "2017-12-26T7:24:28+01:00"
categories = ["debian", "linux", "tor"]
tags = ["debian" , "linux", "tor"]
description = "Bypassing fascist firewalls so you can push your stuff to github"
author = "casept"
+++

I'm writing this on a bus with onboard Wi-Fi. The network works (and is decently fast!), but the provider blocks SSH for some reason, which means I can't push my code to github without reconfiguring my remotes to use HTTP.    
Fortunately, bypassing the firewall is fairly simple:  

1. Install your tools:  

```shell
sudo apt install tor proxychains
# Disable tor service if you don't want to start it on boot
sudo systemctl disable tor
```

In my case the provider also blocked the default tor relays, so I had to obtain some bridges from `bridges.torproject.org` ...which was also blocked, crap! So I used some seedy web proxy to access the site, and got myself some tor bridges.

2. Add the bridges (if needed):
```shell
sudo su
echo 'Bridge <One of the lines you got from the torproject's site>' >> /etc/tor/torrc"
```

3. Start tor:  
```shell
# Run this in a separate terminal/tmux pane etc. as it needs to run in the background
tor
```

4. Do your git thing:   
`proxychains` is already preconfigured to use tor, so you don't need to do much here.
```shell
proxychains git push
```

Note that this should work for most other shell commands as well.
