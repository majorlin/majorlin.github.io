---
layout: DSC核下的时钟调试
title: debug
date: 2019-03-12 15:38:27
tags: MCU
---

从ARM核转到DSC核下面各种不适，今天就这些不适的症状做一个简单的介绍。

## 工具的各种不适

DSC现在使用的还是古老的``Eclipse``编辑器，界面古老且使用各种不便，安装也是各种复杂，为了支持DSC还要安装相应的升级扩展包。另外DSC使用的是``P&E``的调试器，这个调试器真是不负其名，使用之前要经过各种调试，不过好歹还是可以用了，据说是P&E CEO亲自上阵解的bug。曾几何时我也是``Eclipse``的粉丝呀，当年还在大学里面学习Java,不过现在看起来，这个IDE真是各种臃肿，界面也明显跟不上时代了，调试的时候各种窗口让人眼花缭乱，不过既然是唯一的选择，我也没得选择。

## C语言的移植性

曾经天真的我认为C语言的移植问题就是杞人忧天，C语言这么低级有啥好移植的，不都可以直接跑的嘛。知道我遇到了DSC我才知道自己有多天真，曾经将int类型是4个byte这种基础奉为天理，但是如今回首确实各种不堪。DSC让你的int类型变成2byte，甚至地址编址都不再是按照byte编址，而是直接按照16bit编址，这真是超出我的想象，看来我依然只是一只井底之蛙，不过我宁愿待在井底信奉我的真理，奈何现实还是将我无情捞起。从此我也来是老老实实的写可以移植的C语言了，类型定义什么的我也开始使用下面的这种方式了：

```c
/***********************************************************************/
/*
 * The basic data types
 */
typedef unsigned char		uint8;  /*  8 bits */
typedef unsigned short int	uint16; /* 16 bits */
typedef unsigned long int	uint32; /* 32 bits */

typedef char			    int8;   /*  8 bits */
typedef short int	        int16;  /* 16 bits */
typedef long	            int32;  /* 32 bits */

typedef volatile int8		vint8;  /*  8 bits */
typedef volatile int16		vint16; /* 16 bits */
typedef volatile int32		vint32; /* 32 bits */

typedef volatile uint8		vuint8;  /*  8 bits */
typedef volatile uint16		vuint16; /* 16 bits */
typedef volatile uint32		vuint32; /* 32 bits */
```

## 输出一个时钟有这么难

测试时钟的时候也很难呀，比如最开始就碰到了类型的问题，以前一直使用``int``来表征时钟频率，一般的频率也足够了，但是DSC里面不行，这玩意只有16位，一不小心就溢出了，溢出也没啥警告，辛辛苦苦调试半天才找到原因。

另外GPIO输出也是挺奇怪的，这玩意和SIM关系挺大哎，也不多说了，直接码上说事：

```c
int32_t enable_clock_out(int32_t target, clock_out_opt_t clk, int32_t div){
    SIM->CLKOUT &= ~(SIM_CLKOUT_CLKODIV_MASK);
    SIM->CLKOUT |= SIM_CLKOUT_CLKODIV(div);
    if (target){
        /* for clock out1 */
        SIM->CLKOUT &= ~(SIM_CLKOUT_CLKOSEL1_MASK);
        SIM->CLKOUT |= (SIM_CLKOUT_CLKOSEL1(clk));
        SIM->CLKOUT &= ~(SIM_CLKOUT_CLKDIS1_MASK);
        SIM->GPSFL &= ~(SIM_GPSFL_F1_MASK);
        SIM->PCE0 |= SIM_PCE0_GPIOF_MASK;
        SIM->GPSFL |= SIM_GPSFL_F1(0);
        printf("Routing clock out 1 to F1 pin");
        GPIOF->PER |= (1 << 1);
    } else {
        /* for clock out0 */
        SIM->CLKOUT &= ~(SIM_CLKOUT_CLKOSEL0_MASK);
        SIM->CLKOUT |= (SIM_CLKOUT_CLKOSEL0(clk));
        SIM->CLKOUT &= ~(SIM_CLKOUT_CLKDIS0_MASK);
        /* GPIO clock gate */
        SIM->PCE0 |= SIM_PCE0_GPIOG_MASK;
        /* GPIO function selection */
        SIM->GPSGH &= ~(SIM_GPSGH_G11_MASK);
        SIM->GPSGH |= SIM_GPSGH_G11(1);
        /* select function pin */
        GPIOG->PER |= (1 << 11);
        printf("Routing clock out 0 to G11 pin");
    }
    return ((1 << div) + 1);
}
```
看到了吧，开时钟、选功能、GPIO功能选择，一个都不能少，虽说也挺正常，但是还是不习惯，不过好在是可以正常搞起来了。

![Clock Wave](/images/clock_wave.png)

最后看到clock波形的时候还是挺开心的，终于可以正常测试了。
