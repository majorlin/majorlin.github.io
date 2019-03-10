---
title: python3 操作Jlink
date: 2019-03-10 18:16:07
tags: python
---
JLINK是一种常见的ARM调试器，在MCU的测试中我们有时候需要通过脚本和JLINK交互，所以这里就把python3和JLINK交互的一些技巧做一个简单的总结。

## 打开DLL

JLINK实际是有一套完整的API的，这个API可以通过DLL进行调用，我们来看看python3打开API调用的方式：

```python
import ctypes, os, sys
print(os.environ.get("JLINK_ARM_PATH"))
# "C:\Program Files (x86)\SEGGER\JLink_V634e\JLinkARM.dll" for windows
jlink = ctypes.cdll.LoadLibrary(os.environ.get("JLINK_ARM_PATH")
```

上面就是通过ctypes调用JLINK API的方式，这种方式在linux上也可以运行，不过需要注意的是32位的python只能调用32位的DLL，所以这里一定要保持一致。

<!--more-->

## API列表

下面是一些常见的API的一个列表：

|函数名称|含义|
|:--|:--|
|JLINKARM_Open()|打开JLINK|
|JLINKARM_Sleep(int ms)|延时|
|JLINKARM_Close()|关闭设备|
|JLINKARM_Reset()|系统复位|
|JLINKARM_GoAllowSim()|-|
|JLINKARM_Go()|执行程序|
|JLINKARM_Halt()|中断程序|
|JLINKARM_Step()|单步执行|
|JLINKARM_ClrError()|清除错误|
|JLINKARM_SetSpeed(int speed)|设置接口速度|
|JLINKARM_SetMaxSpeed()|设为最高速度|
|JLINKARM_GetVoltage()|读取电压|
|JLINKARM_IsHalted()|查询程序是否暂停|
|JLINKARM_IsOpen()|查询设备是否打开|
|JLINKARM_ClrBP(UInt32 index)|取消断点|
|JLINKARM_SetBP(UInt32 index, UInt32 addr)|设置断点|
|JLINKARM_SetBPEx(UInt32 addr, BP_MODE mode)|设置程序断点|
|JLINKARM_ClrBPEx(int handle)|取消程序断点|
|JLINKARM_WriteReg(ARM_REG index, UInt32 dat)|写寄存器|
|UInt32 JLINKARM_ReadReg(ARM_REG index)|读寄存器|
|JLINKARM_WriteMem(UInt32 addr, UInt32 size, byte[] buf)|写内存|
|JLINKARM_ReadMem(UInt32 addr, UInt32 size, byte[] buf)|读内存|
|UInt32 JLINKARM_GetDLLVersion()|读DLL版本号|
|UInt32 JLINKARM_GetHardwareVersion()|读固件版本|
|UInt32 JLINKARM_GetSN()|读SN|
|UInt32 JLINKARM_GetId()|读MCU ID|

## 举个栗子

内部Memory的读操作可以通过以下方式完成:

```python
def read_mem(jlink, address, count):
    buftype=ctypes.c_ubyte * int(count)
    buf=buftype()
    status = jlink.JLINKARM_ReadMemU8(address, count, buf, 0)
    print("Read status:", status)
    return buf
```

写操作：

```python
def write_mem(jlink, address, buf):
    jlink.JLINKARM_WriteMem.argtypes = [ctypes.c_int, ctypes.c_int, ctypes.c_char_p]
    jlink.JLINKARM_WriteMem.restype = ctypes.c_int
    status = jlink.JLINKARM_WriteMem(address, len(buf), buf)
    if status < 0:
        print("FAIL: download error")
        sys.exit(1)
    print("write status:", status)
```

写寄存器：

```python
def write_reg(jlink, r, val):
    if not isinstance(val, ctypes.c_ulong):
        val = ctypes.c_ulong(val)
    jlink.JLINKARM_WriteReg(r, val)

# set sp
write_reg(jlink, 13, sp)
# set pc
write_reg(jlink, 15, pc + 1)
```

## 选择JLINK

对于批量测试，我们常常会在一台电脑上挂多个JLink，这个时候直接运行脚本的时候就会出现一个选择Jlink的窗口，这个窗口会直接打断我们的测试脚本，这个时候就需要通过指定SN的方式去选择我们需要的Jlink, 指定Jlink的方法如下：

```
jlink.JLINKARM_EMU_SelectByUSBSN(621000000)
```

## 烦人的Device selection

搞定了前面的问题之后，终于开始连CORE的时候，却碰到了如下的提示：

![Jlink Error][1]

这个提示是说没有指定相应的Device，其实这个没有什么必要，不过好像Jlink特别喜欢跳这个东东，这个窗口一跳出来就会打断我们正在执行的脚本，所以这个问题也要搞定。其实搞定这个也没那么难，就是要查很多地方，毕竟自己驱动jlink的人还是少数，不过后来我还是找到了一个方法，就是直接connect之前指定目标core:

```
    jlink.JLINKARM_TIF_Select(1)
    jlink.JLINKARM_CORE_Select(0x60000FF)  # cortex_m0+
```

这里TIF select是指定调试接口类型，这里用的是SWD，后面一句是指定core的类型。

## 总结

其实这一个系列走下来，就相当于自己搞了一个SDK， Segger官方本来是有SDK的，不过因为要收费，所以只能自己慢慢搞上一个了。

  [1]: /images/jlink_error.png