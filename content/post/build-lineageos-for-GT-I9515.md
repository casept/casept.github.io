+++
title = "Building LineageOS 14.1 for the S4 value edition (GT-I9515 or jfvelte)"
date = "2017-05-16T7:24:28+01:00"                                                                                             
categories = ["android"]
tags = ["android"]
description = "I'll try to explain how I got LineageOS to build for the GT-I9515."
author = "casept"
+++

While there's plenty of documentation on how to build LineageOS for officially supported devices, there's hardly any documentation on how to build for an unofficial device and/or on systems which have <16GB RAM and a slower CPU. **A lot of the workarounds might be useful for other devices as well.**
I'll assume that you're using Ubuntu 16.04, though I'm sure other distros will work with slight modifications.      

### Installing dependencies     
Install some packages:     
```
sudo apt-get install repo bc bison build-essential curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-6-dev lib32z1-dev libesd0-dev liblz4-tool libncurses5-dev libsdl1.2-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop pngcrush schedtool squashfs-tools xsltproc zip zlib1g-dev openjdk-8-jdk phablet-tools
```

### Preparing your source directory
Next, find a drive with at least ~150GB of free space (preferably a mechanical one, as constant rebuilds will kill a SSD) and a recent filesystem with case sensitivity and unix permissions support (Such as ext3 or ext4).
`cd` to your drive and create a folder to hold all android source code:
```
mkdir lineage-14.1
cd lineage-14.1
```
Initialize your local repo:
```
repo init -u https://github.com/LineageOS/android.git -b cm-14.1
```

Now come the custom parts. First, create the file `roomservice.xml`:
```
mkdir -p .repo/local_manifests/
vim .repo/local_manifests/roomservice.xml
```

Paste the following into that file:
```
<?xml version="1.0" encoding="UTF-8"?>                                                                                                      
<manifest>                                                                                                                                  
  <project name="jfvelte-dev/android_device_samsung_jfvelte" path="device/samsung/jfvelte" remote="github" revision="cm-14.1"/>             
  <project name="jfvelte-dev/proprietary_vendor_samsung" path="vendor/samsung" remote="github" revision="cm-14.1"/>                         
  <project name="jfvelte-dev/android_kernel_samsung_jf" path="kernel/samsung/jf" remote="github" revision="cm-14.1"/>                       
  <project name="LineageOS/android_device_qcom_common" path="device/qcom/common" remote="github" revision="cm-14.1"/>                       
  <project name="LineageOS/android_device_samsung_qcom-common" path="device/samsung/qcom-common" remote="github" revision="cm-14.1"/>       
  <project name="LineageOS/android_external_stlport" path="external/stlport" remote="github" revision="cm-14.1"/>                           
  <project name="LineageOS/android_hardware_samsung" path="hardware/samsung" remote="github" revision="cm-14.1"/>                           
  <project name="LineageOS/android_packages_resources_devicesettings" path="packages/resources/devicesettings" remote="github" revision="cm-14.1"/>                                                                                                                                     
</manifest>
```
and save.      

Now checkout the source code:
```
repo sync
```
This will take a few hours and download ~50GB.     

Once the download has finished, run
```
source build/envsetup.sh
```
to initialize your build environment.

### A (posibly unnecessary) step
You might have to fix a missing dependency declaration. Open the file `device/samsung/jfvelte/lineage.dependencies`:
```
vim device/samsung/jfvelte/lineage.dependencies
```
And paste in the following before the final "`]`":
```
  {                                                                                                                                         
    "repository": "android_packages_resources_devicesettings",                                                                              
    "target_path": "packages/resources/devicesettings"                                                                                      
  }
```

### Enabling ccache
Create a directory in a suitable location with ~100GB of free space (You can use the same drive as for the sources). Then enable `ccache` and point it towards that directory:
```
export USE_CCACHE=1
export CCACHE_DIR=/PATH/to/YOUR/Directory/.ccache
```

And tell it how much space to use:
```
prebuilts/misc/linux-x86/ccache/ccache -M 100G
```


### Additional configuration
If you wish to have root baked into your build, run
```
export WITH_SU=true
```

**If your PC has ~2 cores or less** you need to build some components manually first due to broken dependency resolution (I guess the bug hasn't been caught yet because most LineageOS devs have beefier machines than that.) Run:
```
breakfast jfvelte
mka org.cyanogenmod.platform-res
```
and wait for it to finish.

**If you have <16GB RAM** you also need to tweak the jack build server (it's used to compile the java parts, such as apps) to use less memory. Let it generate config files by running
```
prebuilts/sdk/tools/jack-admin start-server && prebuilts/sdk/tools/jack-admin kill-server
```

Then edit it's config file:
```
vim ~/.jack-settings
```
and add a line:
```
SERVER_NB_COMPILE=1
```
If you have more RAM you can try increasing that number to 2, though I wouldn't recommend risking it, as guessing wrong will cause the build server to run out of memory and fail. Because there's no `ccache` equivalent for java all the java stuff will have to be recompiled on failure, which can take hours. You also have to tell jack how much memory it can use and start it manually:
```
export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4g"
prebuilts/sdk/tools/jack-admin kill-server && prebuilts/sdk/tools/jack-admin start-server
```

### Building
You should finally be good to go!      
Run `brunch jfvelte` and hope for the best. A flashable zip will be waiting for you in `out/target/product/jfvelte/`. Flash that and you should be finished!     
Just don't forget to kill the jack server:
```
prebuilts/sdk/tools/jack-admin kill-server
```
or you won't be able to cleanly unmount the drive.
