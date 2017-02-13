+++
title = "Installing snapcraft on debian sid"
date = "2017-02-13T7:24:28+01:00"                                                                                             
categories = ["debian", "linux", "snap"]
tags = ["debian" , "linux", "snap"]
description = "How to install snapcraft on debian sid"
author = "casept"
+++

Canonical's snapcraft tool unfortunately currently isn't packaged for debian. In this post I'll describe how I got a build of it to work on my machine™.
Note that this is a messy process which will give any of the "don't make a frankendebian!" guys eye cancer, and for good reason.
### What you'll need   
* patience   
* free time    
* a machine running debian sid (stretch/jessie might work, haven't tested)     

### How to    
#### Step 1: Install (some of the) dependencies          
```
sudo apt update && sudo apt install -y build-essential dpkg-dev wget git debhelper pkg-config python3-docopt python3-fixtures python3-jsonschema python3-libarchive-c python3-lxml python3-magic python3-progressbar python3-pymacaroons python3-requests-toolbelt python3-responses python3-simplejson python3-setuptools python3-tabulate python3-testscenarios python3-testtools python3-xdg python3-yaml squashfs-tools xdelta3 

```

#### Step 2: Clone the latest snapcraft      
The stable version (2.26) had some issues when I tried to use it, so we'll build from git.
```
mkdir snapcraft && cd snapcraft
git clone https://github.com/snapcore/snapcraft
```

#### Step 3: Build a frankendebian
The packages `python3-petname` and `python3-pysha3` are currently not packaged in debian, so we'll download them from ubuntu.
This *should* be low risk, given that those packages have few dependencies and are generally not tightly coupled with other packages.

```
wget http://de.archive.ubuntu.com/ubuntu/pool/main/p/python-petname/python3-petname_2.0-0ubuntu1_all.deb
sudo dpkg -i python3-petname_2.0-0ubuntu1_all.deb
wget http://de.archive.ubuntu.com/ubuntu/pool/universe/p/pysha3/python3-pysha3_1.0.0-0ubuntu1_amd64.deb
sudo dpkg -i python3-pysha3_1.0.0-0ubuntu1_amd64.deb
sudo apt -f install
```

#### Step 4: Build the package
```
cd snapcraft
#Tests are currently broken, disable them
export DEB_BUILD_OPTIONS=nocheck
dpkg-buildpackage -us -uc
```

#### Step 5: Install
```
sudo dpkg -i ../snapcraft_*.deb
```

Profit!
