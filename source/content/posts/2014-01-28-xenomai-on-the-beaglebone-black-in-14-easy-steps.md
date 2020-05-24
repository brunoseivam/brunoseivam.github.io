---
title: Xenomai on the Beaglebone Black in 14 easy steps
author: Bruno Martins
date: 2014-01-28T21:35:48+00:00
url: /xenomai-on-the-beaglebone-black-in-14-easy-steps/
dsq_thread_id:
  - 2696647300
categories:
  - beaglebone
  - linux

---
##### **EDIT: Mark wrote an updated guide [here](http://elinux.org/EBC_Xenomai "New instructions!").**

The [BeagleBone Black](http://beagleboard.org/products/beaglebone%20black "BeagleBone Black") is an amazingly cheap and powerful development platform that is being used by many people in a lot of projects. That was intentionally vague, because I know that if you ended up here you already know what a BeagleBone Black is.

In this post I'll explain how I got [Xenomai](http://www.xenomai.org/ "Xenomai") to run on my BeagleBone.

First of all I tried [these](https://github.com/DrunkenInfant/beaglebone-xenomai "Failed instructions") instructions, but couldn't get past the kernel compilation step. I believe that this is due to the instructions being six months old, which are like two and a half centuries in computer time. So I continued searching and found a [post in a Japanese blog](http://yuina822.blogspot.com.br/ "Japanese blog"). Using ~~my fluent Japanese~~ Google Translator I could understand what was going on and could successfully reproduce the steps and get Xenomai up and running (big thanks to the author!). Here I'll reproduce the steps. I'm assuming that you are on a computer running Ubuntu (like mine) and are familiar with the command line.

# Getting the tools

**Step 0:** Get all the tools that will be needed (cross-compiler and dev libraries).

``` bash
sudo apt-get install build-essential libncurses5{,-dev} gcc-arm-linux-gnueabi
```

# Building the Kernel

**Step 1:** First of all, make a directory to hold all of our development files. I'll call mine `bbb`.

``` bash
mkdir bbb
cd bbb
export BBB=$(pwd)
```

**Step 2:** Get the Linux kernel for the BeagleBone and the Xenomai sources. This might take a while.

``` bash
git clone http://github.com/beagleboard/kernel.git beagle-kernel
wget http://download.gna.org/xenomai/stable/xenomai-2.6.3.tar.bz2
tar xvjf xenomai-2.6.3.tar.bz2
```

**Step 3:** Checkout kernel 3.8 version branch. Apply BeagleBone's patches.

_Note: In this step I revert to a specific commit because newer ones are known to cause problems._

``` bash
cd beagle-kernel
git checkout origin/3.8 -b 3.8
git reset --hard eae56c3
./patch.sh
```

**Step 4:** Get a firmware that the kernel config will need (I'm not sure whether this firmware is really needed).

``` bash
wget "http://arago-project.org/git/projects/?p=am33x-cm3.git;a=blob_plain;f=bin/am335x-pm-firmware.bin;hb=HEAD" -O kernel/firmware/am335x-pm-firmware.bin
```

**Step 5:** Copy the BeagleBone default config as the running config.

``` bash
cp configs/beaglebone kernel/.config
```

**Step 6:** Apply I-pipe patches to the BeagleBone kernel.

``` bash
cd kernel
patch -p1 &lt; ../../xenomai-2.6.3/ksrc/arch/arm/patches/beaglebone/ipipe-core-3.8.13-beaglebone-pre.patch
patch -p1 &lt; ../../xenomai-2.6.3/ksrc/arch/arm/patches/ipipe-core-3.8.13-arm-3.patch
patch -p1 &lt; ../../xenomai-2.6.3/ksrc/arch/arm/patches/beaglebone/ipipe-core-3.8.13-beaglebone-post.patch
```

**Step 7:** Run the Xenomai `prepare-kernel`  script for the BeagleBone kernel.

``` bash
cd ../../xenomai-2.6.3/scripts
./prepare-kernel.sh --arch=arm --linux=../../beagle-kernel/kernel
```

**Step 8:** Configure the kernel to be built.

``` bash
cd ../../beagle-kernel/kernel
make ARCH=arm menuconfig
```

Under `CPU Power Management --->  CPU Frequency scaling`, disable `[ ] CPU Frequency scaling`. _(Note: Don't know if it's better to leave it enabled, read the comments!)_

Under `Real-time sub-system  ---> Drivers ---> Testing drivers`, enable everything.

**Step 9:** Compile the kernel.

_Note: I chose 16 to the <span class="lang:default decode:true  crayon-inline ">-j</span>  option, because my computer has 8 cores. Choose a value appropriate to your computer. I read somewhere that 2 times the number of cores is a good number._

_Note: If there were errors in the compilation, the messages will probably be lost among all other output. To see them, simply run the command again._

``` bash
make -j16 ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- LOADADDR=0x80008000 uImage dtbs modules
```

# **Preparing an SD Card**

Now let's get an SD Card ready with the Angstrom distribution and our kernel. If you want to use this kernel with the distribution on the eMMC memory, just put it in the appropriate place.

**Step 10:** Download and copy the default Angstrom distribution to your SD Card. Replace `/dev/sdX`  with the path to your SD Card. `sudo fdisk -l`  is your friend. _Note: I used a SanDisk 4GB SD Card._

**CAUTION: YOU WILL LOSE ALL YOUR PREVIOUS DATA ON THE DEVICE `/dev/sdX` !**

``` bash
cd $BBB
wget https://s3.amazonaws.com/angstrom/demo/beaglebone/Angstrom-Cloud9-IDE-GNOME-eglibc-ipk-v2012.12-beaglebone-2013.06.20.img.xz
sudo -s
xz -dkc Angstrom*img.xz &gt; /dev/sdX
exit
```

**Step 11:** Mount the Angstrom partition. Copy kernel and kernel modules (thanks for your comment, Jurg Lehni!), Xenomai modules and source folder to that partition. Replace `/dev/sdX2` with your actual path to the partition.

``` bash
mkdir sd
sudo mount /dev/sdX2 sd
sudo cp beagle-kernel/kernel/arch/arm/boot/uImage sd/boot/uImage-3.8.13
cd beagle-kernel/kernel
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- INSTALL_MOD_PATH=$BBB/sd modules_install
cd -
mkdir sd/home/root/xeno_drivers
cp beagle-kernel/kernel/drivers/xenomai/testing/*.ko sd/home/root/xeno_drivers/
cp -r xenomai-2.6.3 sd/home/root
sudo umount sd
```

# Testing

Now put the SD card on the BeagleBone, boot it, ssh into it and test Xenomai.

**Step 12:** Configure the date and compile Xenomai

_Note: an example for the date command would be_ `date -s "21 May 2014 13:25 GMT-3"` 

``` bash
date -s "DD MMM YYYY HH:MM TZ"
cd ~/xenomai-2.6.3
./configure CFLAGS="-march=armv7-a -mfpu=vfp3" LDFLAGS="-march=armv7-a -mfpu=vfp3"
make
make install
```

**Step 13:** Load the testing driver

``` bash
cd ~/xeno_drivers
insmod xeno_klat.ko
```

**Step 14:** Run some tests!

User-mode latency:

``` bash
# /usr/xenomai/bin/latency
== Sampling period: 1000 us
== Test mode: periodic user-mode task
== All results in microseconds
warming up...
RTT|  00:00:01  (periodic user-mode task, 1000 us period, priority 99)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|      6.916|      7.083|     11.708|       0|     0|      6.916|     11.708
RTD|      6.874|      8.041|     22.708|       0|     0|      6.874|     22.708
RTD|      8.749|      9.041|     28.333|       0|     0|      6.874|     28.333
RTD|      8.791|      9.041|     22.583|       0|     0|      6.874|     28.333
RTD|      8.791|      9.083|     26.499|       0|     0|      6.874|     28.333
RTD|      8.791|      9.041|     24.541|       0|     0|      6.874|     28.333
RTD|      8.708|      8.999|     25.208|       0|     0|      6.874|     28.333
RTD|      8.833|      9.041|     24.041|       0|     0|      6.874|     28.333
RTD|      8.749|      9.041|     26.291|       0|     0|      6.874|     28.333
RTD|      8.791|      8.999|     22.416|       0|     0|      6.874|     28.333
^C---|-----------|-----------|-----------|--------|------|-------------------------
RTS|      6.874|      8.708|     28.333|       0|     0|    00:00:10/00:00:10
```

In-kernel Latency:

``` bash
# /usr/xenomai/bin/klatency
== Sampling period: 100 us
== Test mode: in-kernel periodic task
== All results in microseconds
warming up...
RTT|  00:00:00  (in-kernel periodic task, 100 us period, priority 99)
RTH|-----lat min|-----lat avg|-----lat max|-overrun|----lat best|---lat worst
RTD|      -3.250|      -0.846|       3.791|       0|      -4.250|       5.167
RTD|      -0.917|      -0.800|       4.041|       0|      -4.250|       5.167
RTD|      -0.917|      -0.801|       4.333|       0|      -4.250|       5.167
RTD|      -1.084|      -0.806|       1.499|       0|      -4.250|       5.167
RTD|      -1.043|      -0.788|       3.999|       0|      -4.250|       5.167
RTD|      -3.335|      -0.794|       3.624|       0|      -4.250|       5.167
RTD|      -0.918|      -0.744|       4.707|       0|      -4.250|       5.167
RTD|      -0.919|      -0.803|       4.581|       0|      -4.250|       5.167
RTD|      -0.877|      -0.797|       4.581|       0|      -4.250|       5.167
RTD|      -0.919|      -0.804|       3.373|       0|      -4.250|       5.167
RTD|      -0.919|      -0.802|       4.622|       0|      -4.250|       5.167
RTD|      -0.920|      -0.804|       3.372|       0|      -4.250|       5.167
^CRTD|      -0.920|      -0.801|       3.622|       0|      -4.250|       5.167
---|------------|------------|------------|--------|-------------------------
RTS|      -4.250|      -0.744|       5.167|       0|    00:00:06/00:00:06
```

Poke around:

``` bash
# cat /proc/xenomai/version
2.6.3
# cat /proc/ipipe/version
3
```

Change parameters:

``` bash
# cat /proc/xenomai/latency
4999
# echo "0" &gt;&gt; /proc/xenomai/latency
# cat /proc/xenomai/latency
0
# rmmod xeno_lat
# cd ~/xeno_drivers
# insmod xeno_klat.ko
# /usr/xenomai/bin/klatency
== Sampling period: 100 us
== Test mode: in-kernel periodic task
== All results in microseconds
warming up...
RTT|  00:00:01  (in-kernel periodic task, 100 us period, priority 99)
RTH|-----lat min|-----lat avg|-----lat max|-overrun|----lat best|---lat worst
RTD|       4.041|       4.190|       9.958|       0|       4.041|      10.166
RTD|       3.582|       4.205|      10.374|       0|       3.582|      10.374
RTD|       4.082|       4.191|       8.957|       0|       3.582|      10.374
RTD|       4.082|       4.182|       9.665|       0|       3.582|      10.374
RTD|       4.082|       4.197|       8.373|       0|       3.582|      10.374
RTD|       3.623|       4.193|       8.457|       0|       3.582|      10.374
RTD|       3.915|       4.184|       9.040|       0|       3.582|      10.374
RTD|       4.081|       4.184|       8.206|       0|       3.582|      10.374
RTD|       0.456|       4.180|       8.664|       0|       0.456|      10.374
RTD|       0.455|       4.196|       8.831|       0|       0.455|      10.374
RTD|       4.080|       4.182|       8.997|       0|       0.455|      10.374
RTD|       4.080|       4.197|       9.622|       0|       0.455|      10.374
RTD|       4.038|       4.187|       9.913|       0|       0.455|      10.374
RTD|       4.079|       4.194|       8.788|       0|       0.455|      10.374
RTD|       4.079|       4.181|       8.538|       0|       0.455|      10.374
RTD|       4.079|       4.175|       8.579|       0|       0.455|      10.374
^CRTD|       0.704|       4.195|       9.079|       0|       0.455|      10.374
---|------------|------------|------------|--------|-------------------------
RTS|       0.455|       3.967|      10.374|       0|    00:00:17/00:00:17
```

Great, huh? Now go develop something real time =)

{{ template "_internal/disqus.html" . }}
