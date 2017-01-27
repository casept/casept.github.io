+++
title = "Fixing chromium in debian unstable"
date = "2017-01-27T7:24:28+01:00"                                                                                             
categories = ["debian", "linux"]
tags = ["debian" , "linux"]
description = "The latest versions of chromium shipped in debian unstable have disabled support for third-party extensions"
author = "casept"
+++

For some (in my opinion pants-on-head-retarded) reason the maintainers of the chromium package in debian have chosen to *disallow loading of extensions from the chrome store*.
They claim that this will pave the way towards "distro-packaged" extensions, but as of yet I haven't seen a single one in the repos, not even ublock.
What's worse, the only notice you get is if you upgrade from a version before this "fix" was applied.
Users on the mailing list [ain't happy either](https://lists.alioth.debian.org/pipermail/pkg-chromium-maint/2017-January/008959.html).

### How to unfuck this ###  
The best way I have found is to edit the .desktop file at `/usr/share/applications/chromium.desktop` and replace the line `Exec=/usr/bin/chromium %U` with `Exec=/usr/bin/chromium --enable-remote-extensions %U`.
I certainly hope the maintainers come to their senses, as doing this after every update is ridicilously inconvenient and it breaks the chromium ecosystem.
