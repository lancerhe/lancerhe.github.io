---
title: AWS EC2 通过 resize2fs 修改 EBS Volume
author: 谇雨
layout: post
permalink: /amazon-ec2-drive-not-EBS-volume-size.html
categories:
  - AWS
tags:
  - AWS
  - EC2
---

> 本文转载：[aws ec2 硬盘 resize2fs](http://www.netingcn.com/aws-ec2-resize2fs-storage.html){:target="_blank"}

在申请 `AWS EC2` 时，按照向导，在选择存储的时候默认硬盘大小是 8G，这时候可以根据自己的需要输入一个合适的数字，例如100。完成向导并启动 EC2 instance 后登陆机器。使用命令：

    df -hT

发现硬盘的大小不是自己的设定的值，而还是 8G，使用`fdisk`、`mkfs`来分区和格式化后，还是无法增大其空间。反复折腾多次，包括重启机器，问题依旧，后来发现其实很简单，只需要使用一条命令`resize2fs`就可以搞定。

    resize2fs /dev/xvde

注意：`/dev/xvde` 根据自己的实际情况可能会不一样。使用`fdisk`或`df`命令都可以获知具体的设备号。 如果执行上述命令收到 The filesystem is already 2096896 blocks long. Nothing to do! 的错误，那么需要先做如下操作：

    <<1>> Look at the filesystem, it is 6G
    <<2>> Look at the disk and the partition, the disk is 21.5 GB but the partition is 6 GB (6291456 blocks)
    <<3>> Start fdisk for that disk (xvda, so not the partition xvda1)
    <<4>> Switch to sector display.
    <<5>> Print the partition(s), and remember the start sector (2048 in the example).
    <<6>> Delete the partition.
    <<7>> Create a new partition.
    <<8>> Make it primary.
    <<9>> First partition.
    <<10>> Enter the old start sector, do NOT make any typo here!!! (2048 in the example) 
    <<11>> Hit enter to accept the default (this is the remainder of the disk)
    <<12>> Print the changes and make sure the start sector is ok, if not restart at <<6>>
    <<13>> Make the partition bootable. do NOT forget this!!!
    <<14>> Enter your partition number (1 in the example)
    <<15>> Write the partition info back, this will end the fdisk session.
    <<16>> Reboot the server, and wait for it to come up (this may take longer than usual).
    <<17>> Verify the filesystem size.
    <<18>> If the filesystem is not around 20Gb as expected, you can use this command.

    # df -h  <<1>>

    Filesystem      Size  Used Avail Use% Mounted on
    /dev/xvda1      6.0G  2.0G  3.7G  35% / 
    tmpfs            15G     0   15G   0% /dev/shm

    # fdisk -l  <<2>>

    Disk /dev/xvda: 21.5 GB, 21474836480 bytes
    97 heads, 17 sectors/track, 25435 cylinders
    Units = cylinders of 1649 * 512 = 844288 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x0003b587

        Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1   *           2        7632     6291456   83  Linux

    # fdisk /dev/xvda  <<3>>

    WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
             switch off the mode (command 'c') and change display units to
             sectors (command 'u').

    Command (m for help): u  <<4>>
    Changing display/entry units to sectors

    Command (m for help): p  <<5>>

    Disk /dev/xvda: 21.5 GB, 21474836480 bytes
    97 heads, 17 sectors/track, 25435 cylinders, total 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x0003b587

        Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1   *        2048    12584959     6291456   83  Linux

    Command (m for help): d  <<6>>
    Selected partition 1

    Command (m for help): n  <<7>>
    Command action
       e   extended
       p   primary partition (1-4)
    p  <<8>>
    Partition number (1-4): 1  <<9>>
    First sector (17-41943039, default 17): 2048  <<10>>
    Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): <<11>>
    Using default value 41943039

    Command (m for help): p <<12>>

    Disk /dev/xvda: 21.5 GB, 21474836480 bytes
    97 heads, 17 sectors/track, 25435 cylinders, total 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x0003b587

        Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1            2048    41943039    20970496   83  Linux

    Command (m for help): a  <<13>>
    Partition number (1-4): 1  <<14>>


    Command (m for help): w  <<15>>
    The partition table has been altered!

    Calling ioctl() to re-read partition table.

    WARNING: Re-reading the partition table failed with error 16: ...
    The kernel still uses the old table. The new table will be used at
    the next reboot or after you run partprobe(8) or kpartx(8)
    Syncing disks.

    # reboot  <<16>>



    # df -h  <<17>>
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/xvda1       20G  2.0G   17G  11% / 
    tmpfs            15G     0   15G   0% /dev/shm

    # resize2fs /dev/xvda1  <<18>>
    resize2fs 1.41.12 (17-May-2010)
    Filesystem at /dev/xvda1 is mounted on /; on-line resizing required
    old desc_blocks = 1, new_desc_blocks = 2
    Performing an on-line resize of /dev/xvda1 to 5242624 (4k) blocks.
    The filesystem on /dev/xvda1 is now 5242624 blocks long.

    root@vs120 [~]#  df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/xvda1       20G  7.8G   11G  42% /
    tmpfs           498M     0  498M   0% /dev/shm
    /usr/tmpDSK     399M   11M  368M   3% /tmp

更多信息可以参考这里：[http://serverfault.com/questions/414983/ec2-drive-not-ebs-volume-size](http://serverfault.com/questions/414983/ec2-drive-not-ebs-volume-size){:target="_blank"}