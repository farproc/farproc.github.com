---
layout: post
title: New firmware for DLink DIR-835
---

### {{ page.title }}

I've been having some stability problems with my router since I bought it a few months ago. They'd kind of settled down and I stopped tinkering with it - but unfortently half the cool features are currently disabled. I've been waiting for a firmware update to hopefully fix all my problems and one turned up today!

I was running:

    Current Firmware Version :  1.01    Date :   18 Jan 2012

and the new image is labeled:

    Latest Firmware Version :   1.01    Date :   29 Feb 2012

Although I'm sure it hasn't been availbable since that date and its not linked off the dlink website either. The router itself found it though its magic __check now__ button (tools -> firmware).

Anyway - so I downloaded "DIR835A1\_FW101B07.bin" and uploaded it to the router (via a wired connection like it says) and upgraded the router. Awesome - hopefully everything will be sorted out. Upgrade went soothly and seem to keep all my configuration as well.

While waiting for all that to happen i started poking into the firmware upgrade.

    silver:router tim$ file DIR835A1_FW101B07.bin 
    DIR835A1_FW101B07.bin: u-boot legacy uImage, Linux Kernel Image, Linux/MIPS, OS Kernel Image (lzma), 1280031 bytes, Wed Jan 18 08:57:55 2012, Load Address: 0x80002000, Entry Point: 0x802A9E70, Header CRC: 0x7AC9A8E4, Data CRC: 0x8A46998D

Well that all seems to make sense - MIPS Linux kernel image. Its no big surprise it running linux, its got a TTL of 64 as well. Then I discovered a cool tool called [binwalk](https://code.google.com/p/binwalk/). It basically runs though every offset in the file and fires it though libmagic (the _magic_ behind the _file_ command).

    silver:router tim$ binwalk DIR835A1_FW101B07.bin 

    DECIMAL     HEX         DESCRIPTION
    -------------------------------------------------------------------------------------------------------
    0           0x0         uImage header, header size: 64 bytes, header CRC: 0x7AC9A8E4, created: Wed Jan 18 08:57:55 2012, image size: 1280031 bytes, Data Address: 0x80002000, Entry Point: 0x802A9E70, data CRC: 0x8A46998D, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: Linux Kernel Image
    64          0x40        LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 3752448 bytes
    1310720     0x140000    Squashfs filesystem, little endian, version 4.0, size: 1903145727 bytes, 1074 inodes, blocksize: 0 bytes, created: Sun Jun 30 19:55:44 2030
    1523130     0x173DBA    JFFS2 filesystem (old) data big endian, JFFS node length: 1

OK - so this is looking pretty good. I'm guessing:

    0       ->        63    (64)        uImage Header
         64 ->   1310719    (1310656)   Linux Kernel (compressed)
    1310720 ->   1523129    (212410)    initrd ram disk
    1523130 ->  16318464    (14995334)  root file system

So lets extract those sections

    dd bs=1 if=DIR835A1_FW101B07.bin skip=64 count=1310656 of=./kernel
    dd bs=1 if=DIR835A1_FW101B07.bin skip=1310720 count=212410 of=./initrd
    dd bs=1 if=DIR835A1_FW101B07.bin skip=1523130 of=./rootfs

So lets see what we've got:

    lzma -dc kernel |strings

Looks kernel like....

Now lets try and mount that rootfs (or so i think?!)

    sudo modprobe mtdram total_size=32768 erase_size=256
    sudo modprobe mtdblock
    sudo modprobe mtdchar
    # sudo mknod /dev/mtdblock0 b 31 0
    # dd if=image-3.jffs2 of=/dev/mtdblock0
    # mount -t jffs2 /dev/mtdblock0 /mnt/disk

I stopped here - but hopefully I'll come back and finish this off once I've got a little more time. Let me know if you get further than me...
