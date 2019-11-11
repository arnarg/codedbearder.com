---
title: "Running NixOS on a Terramaster NAS"
date: 2019-11-10T23:29:37Z
draft: true
categories:
- Linux
tags:
- linux
- nas
- nixos
- plex
---

I have an old whitebox server running a Xeon E5 (v1) that's running a handful of VMs on Proxmox. Most of them I can get rid of by moving their services to Kubernetes, but I wasn't sure what I wanted to do with my NAS VM and Plex server. The latter you can easily run in Kubernetes but I'm not planning on having particularly powerful nodes and I need transcoding.

When I was looking at potential consumer NAS boxes I noticed in a teardown of [Terramaster F2-221][1] that there's just an internal USB header running the OS which gave me hope of running any OS that I want on it (that and that it has a standard x86_64 Intel CPU). This may very well be the case with other NASes from other manufacturers but I'm not too familiar with it.

### The Hardware

The Terramaster F2-221 has an Intel Celeron J3355 CPU and 2GB of DDR3 but has an unpopulated SODIMM slot so you can upgrade the memory. The J3355 has Intel HD Graphics 500 with Quick Sync support which makes it capable of Hardware-Accelerated Transcoding in Plex. Terramaster claims that it can do two simultaneous 4K streams but I have not put that to the test.

Servicing it is relatively simple, there's just 4 screws in the back of the unit that need to be unscrewed, even the screwdriver is included. After that you can unplug the fan and slide the aluminum shroud off the unit, revealing the motherboard. From there you can pop in extra RAM without unscrewing the motherboard. Terramaster does sell some "compatible" RAM on Amazon but I popped in a DDR3 1600MHz 4GB stick I had on hand and it worked perfectly fine.

### The Software

The USB stick plugged into the internal header doesn't actually run the OS as it turns out. It just boots a minimal environment exposing a web server which you can navigate to to actually install Terramaster OS (TOS) by going through a fairly user friendly wizard.

It partitions your hard drives installed in the unit and raids them using mdadm, one for the operating system and one for the data storage (I'm writing this from memory, but that's the gist of it). On the raided data partition it uses btrfs.

Now, I already have a couple of drives that are formatted with btrfs in raid1 configuration. That means that I would need to wipe them to install TOS on there and then migrate my files back on to them. The installation wizard does display a warning about this.

For those reasons and that TOS is not that great for my requirements I decided to roll my own NAS OS (this was my plan before buying the unit). I do however realize that I'm probably not the average user that would buy this and TOS does look fairly user friendly.

## Improving the software

[1]: https://www.terra-master.com/global/products/homesoho-nas/f2-221.html
