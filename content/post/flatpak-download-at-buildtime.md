+++
date = "2017-01-26T17:04:28+01:00"
title = "Downloading files from the internet during a flatpak build"
categories = ["flatpak", "linux"]
tags = ["flatpak" , "linux"]
description = "Downloading files from the internet during a flatpak build"
author = "casept"
+++

Flatpak's [official documentation](http://flatpak.org/developer.html) explains how to automate the building of flatpaks.
However, what it does not mention is how the build sandbox is configured not to allow "unfiltered" access to the network at build time.
This means that it can be very tricky to build some applications that rely on pulling in dependencies after you have cloned the repo or downloaded the tarball.
An example for this are Node.js projects, which require the usage of `npm` in order to install dependencies.
The build fails with symptoms such as:

```
npm install
npm ERR! fetch failed https://github.com/beeman/node-df/tarball/master
npm WARN retry will retry, error on last attempt: Error: getaddrinfo EAI_AGAIN github.com:443
``` 

The best way to work around this limitation is by adding

```
"build-options" : { 
   "build-args": [ 
      "--share=network"                                                                                   
      ] 
   },
``` 

to your `name.of.your.app.json` file, right below the `name:` field of the module.

Do note that there's a reason for this behaviour in flatpak: without it reproducible builds are not possible, as the build script could fetch any number of versions of dependencies from the internet.
