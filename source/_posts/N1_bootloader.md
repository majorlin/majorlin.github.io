---
title: 斐讯N1 Bootloader
date: 2019-03-10 18:16:07
tags: MCU
---
最近把家里的N1系统给玩坏了，后面重装系统的时候，系统一直装不到EMMC上，这个可真的能逼死我这个强迫症，无奈只好将N1拆机捣鼓。

先说下系统玩坏的过程，也就是闲的没事干，想把N1上的linux升级一下，因为最新的内核好像也可以支持WIFI和蓝牙了，在升级的过程中发现4.19不能直接装到EMMC上，因为4.19上枚举不出来``/dev/data``，没办法，只好回到3.19, 本来直接回去装个驱动就好了，但是手贱的我直接把dtb给升级替换了，不巧，新的dtb并不好用，所以系统就起不来了，好歹还能从U盘启动，本来吧，直接启动完把dtb换回来就好了，我偏偏又重装了一下，熟练重启，系统崩溃。

拆机连上UART发现log上面也米有什么有用的东西，最后把分区重新格式化，重新装，莫名其妙的就启动起来了，这个。。。。我也不知道说什么才好，总之EMMC上系统跑起来还是很快的，折腾就折腾了。

<!--more-->

在斐讯N1的bootloader中，启动的命令是start\_autoscript：

```
start_autoscript=\
if usb start ; 
    then run start_usb_autoscript; 
fi; 
if ext4load mmc 1:c 1020000 /boot/s905_autoscript;
    then autoscr 1020000; 
fi;
start_mmc_autoscript=if fatload mmc 0 1020000 s905_autoscript; then autoscr 1020000; fi;
start_usb_autoscript=\
    if fatload usb 0 1020000 s905_autoscript; 
    then autoscr 1020000; 
fi; 
    if fatload usb 1 1020000 s905_autoscript; 
    then autoscr 1020000; fi; 
    if fatload usb 2 1020000 s905_autoscript; then autoscr 1020000; fi; 
    if fatload usb 3 1020000 s905_autoscript; then autoscr 1020000; fi;
```

上面是从uboot env中得到的启动脚本的内容，可以看到这个启动过程首先会尝试从USB上启动，启动不成功才会转到EMMC，启动过程是通过fatload或者ext4load直接从USB或者EMMC load启动脚本的，知道了这个命令格式，我们可以通过如下的命令查看emmc里面的内容：

```
gxl_p230_v1#ext4ls mmc 1:c
<DIR>       4096 .
<DIR>       4096 ..
<DIR>      16384 lost+found
<DIR>          0 bin
<DIR>          0 boot
<DIR>       4096 dev
<DIR>       4096 etc
<DIR>          0 home
<DIR>          0 lib
<DIR>       4096 media
<DIR>       4096 mnt
<DIR>       4096 opt
<DIR>       4096 proc
<DIR>       4096 root
<DIR>       4096 run
<DIR>       4096 sbin
<DIR>       4096 selinux
<DIR>       4096 srv
<DIR>       4096 sys
<DIR>       4096 tmp
<DIR>       4096 usr
<DIR>          0 var
```
上面就是一个读取目录的命令，我们可以根据这个命令的结果查看分区里面的内容是否正确，上面``/boot``的内容完全为空，估计这就是启动失败的原因，不过在U盘启动的linux中，我们可以看到这个分区的内容是完全正确的，这个里面的原因我到现在还没能弄明白。

反正最后不知道怎么系统就好了，本来都打算开始重新刷android的镜像了，一下看到这个log就出来了，还挺灵异的。

