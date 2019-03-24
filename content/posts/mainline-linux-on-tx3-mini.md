---
title: "TX3 Mini: Mainline Linux on an Android TV Box"
date: 2019-03-23T21:23:25Z
draft: true
---


I've always found Raspberry Pi's cases and the Pi's connector locations to be rather aesthetically unpleasing, with the power connector coming out from the side and the ethernet from the back (or vice versa). Imagine my excitement when I was browsing the [github mirror of Linux][1] (as you do) and I ran into a dts for an Android TV box called Tanix TX3 Mini.

<!--more-->

{{% zoom-img src="/images/posts/mainline-linux-on-tx3-mini/tx3-mini-hero.jpg" %}}

It has the following specs:

* Amlogic S905W chipset
  * Quad Core 1.2GHz 64bit Cortex-A53
  * Mali-450
* 1 or 2 GiB of DDR3 memory
* 8 or 16 GiB of eMMC flash
* 10/100Mbit Ethernet
* HDMI output up to 4K30P

It's also cheap enough so that I wouldn't feel bad about bricking it in my attempt to get Mainline Linux running on it. I got the 2 GiB RAM and 16 GiB eMMC model. They usually go for about $30 on AliExpress or Ebay and that's including a case (duh!), power brick, HDMI cable and an IR remote (I'm using my to control my TV soundbar, how great!?).

There are images with LibreELEC and armbian out there on forums, but they require you to hold down a small button on the board to boot from the sd card and also runs the same kernel as the original Android OS, I believe. That simply won't do!

# U-Boot

This TV box ships with Android installed and has a rather old version of Linux and U-boot installed. Most TV boxes that share the same chipset are based on the same reference design and include an unsoldered serial header on the board, how handy!

{{% zoom-img src="/images/posts/mainline-linux-on-tx3-mini/tx3-mini-board.jpg" %}}

After you've soldered a pin header to the serial port and connected to it you'll see that it's running U-Boot v2015.01, yikes! This version can't boot a modern Linux image, we're gonna have to compile our own!

This proved to be more complicated than I imagined...

### Boot Loader stages what!?

Ok look, I'm no expert but apparently aarch64 has secure boot and it doesn't just boot U-Boot directly but in stages... 4 stages! That's 4 bootloader stages that run before U-Boot runs and in turn boots Linux. Although, one of those stages (stage 3-2) is optional and not used on our board, but 4 sounded better for dramatic effect, give me a break Karen!

If you're interested you can read more about it [here][2], in a lot of detail!

### Missing defconfig

So, the U-Boot source code was missing any defconfig resembling tx3-mini, f%#@! But after navigating the source for a while I found a board that sounds a lot like mine!

![The P212](/images/posts/mainline-linux-on-tx3-mini/p212.gif#img-center)

The P212 has Amlogic S905X chipset which is a little different from the TX3 Mini's S905W. Now, it's been a few months on and off that I've worked on this so I don't remember exactly what steps I took to figure out that TX3 Mini is referenced from P281, but I did!

I simply made a copy of all the files for p212 and renamed them to p281, most importantly to make `CONFIG_DEFAULT_DEVICE_TREE` equal `meson-gxl-s905w-p281` as its [dtb][3] is upstream and will therefor ship with distros. No need to compile a dtb manually! I have a [fork of U-Boot][4] that includes this on branches called `<version>-tx3-mini`.

With that done we can simply run:
```bash
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- p281_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-
```

> aarch64-linux-gnu is the name of the aarch64 toolchain in Arch Linux' repos, your distro might call it something else.

### Let's get back to that secure boot thing

So here we are, we've compiled a bootloader that we can't boot yet!

The reason it was so great to find the P212 in U-Boot's source is that it has a great [README][5] on how to create the complete bootloader image. The only problem is that there's no defconfig for p281 in the repo listed in the instructions, again! I did however find another one that [did!][6]

The Makefile hardcodes the `CROSS_COMPILE` variable so you'll need [this][7] toolchain (or you know, fix the Makefile) but if you're running on a recent kernel some header files won't contain something something needed. I ended up compiling it on CentOS 7 as that kernel is ancient. 

I'm also a generous guy and I have pre-compiled it for anyone that trusts me, [here][8] (only for the 2 GiB version, I'll get back to that one).

### Wiping the eMMC

To wipe the eMMC we're gonna need the [aml-flash][9] tool! This tool is meant for burning the Android image to it but it does have a handy `--destroy` flag that wipes the bootloader from eMMC. The reason we want this is that the soc looks for the bootloader on eMMC before it looks at the sd card, so we need it gone! You will be able to go back to Android if you obtain an image from [here][10] and use this tool to flash it.

All you have to do to wipe the eMMC is to go through the install instructions in the repo find a USB Type-A to Type-A cable (stop looking, you don't have it, why would you?), plug it into the USB slot closer to the sd card on the board, the other into your computer and run:

`aml-flash --destroy`

I made my cable by soldering a USB Type-B PCB socket I had laying around to a cut USB cable. Then I could simply use a USB Type-A to USB Type-B cable to connect to it.

{{% zoom-img src="/images/posts/mainline-linux-on-tx3-mini/cable.jpg" %}}

You could also just solder two of them together or order one online, turns out they do exist!

### That's not the memory I was promised!

Here we are, we have compiled U-Boot and created a bootloader image. Only thing left is to burn it to an sd card. You'll find the following in the README for P212 to do so:

```bash
DEV=/dev/your_sd_device
dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=512 skip=1 seek=1
dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=1 count=444
```

Now pop that sd card into your board, plug in the serial port (baud rate 115200) and look at that thing boot U-Boot! Wait, we're only seeing 1 GiB of RAM (unless you just downloaded my pre-compiled fip thingy)!?

After looking through the [uboot-amlogic][6] repo for a while I found a handy variable called [CONFIG_DDR_SIZE][11]. Its value is in MiB and is currently set to `1024`, now this is promising! After changing it to `2048` and re-compiling, it boots and detects 2 GiB!

That's why the pre-compiled file will only works on 2 GiB models (I think). "Why didn't you just set it to 0 you idiot!? It clearly says in the comment that it will auto-detect that way!" you ask, and to that I have to say, first of all that's rude and second because I didn't, did I!?

But in all seriousness when it started reporting the correct amount of RAM I was too excited to go back and try it with 0.

# Linux

[1]: https://github.com/torvalds/linux
[2]: https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/firmware-design.rst
[3]: https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/amlogic/meson-gxl-s905w-p281.dts
[4]: https://github.com/arnarg/u-boot/tree/v2019.01-tx3-mini
[5]: https://github.com/u-boot/u-boot/blob/master/board/amlogic/p212/README.p212
[6]: https://github.com/Stane1983/uboot-amlogic
[7]: https://releases.linaro.org/archive/13.11/components/toolchain/binaries/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz
[8]: /files/gxl-p281-fip-2g.tar.gz
[9]: https://github.com/Stane1983/aml-linux-usb-burn
[10]: http://www.tanix-box.com/download-view/tanix-tx3-mini-firmware-full-image-20170829/
[11]: https://github.com/Stane1983/uboot-amlogic/blob/master/board/amlogic/configs/gxl_p281_v1.h#L259
