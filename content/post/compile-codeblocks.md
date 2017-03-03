+++
date = "2017-03-03T17:04:28+01:00"
title = "Building code::blocks on ubuntu 16.04" 
categories = ["linux"]
tags = ["linux"]
description = "Building code::blocks on ubuntu 16.04"
author = "casept"
+++

I'm currently creating a snap package for the code::blocks IDE.
On the way, I ran into a little snag, namely the build failing with:  
```
make -j2
CDPATH="${ZSH_VERSION+.}:" && cd . && /bin/bash /home/david/dev/snap/snap-codeblocks/parts/codeblocks/build/missing aclocal-1.13 -I m4
/home/david/dev/snap/snap-codeblocks/parts/codeblocks/build/missing: line 81: aclocal-1.13: command not found
WARNING: 'aclocal-1.13' is missing on your system.
         You should only need it if you modified 'acinclude.m4' or
         'configure.ac' or m4 files included by 'configure.ac'.
         The 'aclocal' program is part of the GNU Automake package:
         <http://www.gnu.org/software/automake>
         It also requires GNU Autoconf, GNU m4 and Perl in order to run:
         <http://www.gnu.org/software/autoconf>
         <http://www.gnu.org/software/m4/>
         <http://www.perl.org/>
Makefile:505: recipe for target 'aclocal.m4' failed
make: *** [aclocal.m4] Error 127
Command '['/bin/sh', '/tmp/tmpophagygp', 'make', '-j2']' returned non-zero exit status 2
```

The fix is surprisingly simple - all you have to do is run `./bootstrap` beforehand. In a snapcraft.yaml it would look something like this:  
```
  your-part:                                                                                                                         
    plugin: autotools                                                                                                                 
    source: http://example.com/your-part.tar.gz                                 
    prepare: ./bootstrap                                                                                                              
```
