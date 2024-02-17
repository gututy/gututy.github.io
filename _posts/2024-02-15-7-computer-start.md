---
title: 计算机系统启动过程分析
tags: 计算机体系结构 学习笔记
permalink: /notes/ca/computer-start
aside:
    toc: ture
sidebar:
    nav: notes
---

> 第七章：计算机启动过程分析
<!--more-->

# 一句话要点
```markdown
- 系统启动
  - 从复位到系统可用
- 初始化是什么
  - 将系统各种寄存器状态从不确定设置为确定，将一些模块状态从无序强制为有序的一个过程
- 什么东西需要初始化
  - CPU、内存、各类IO接口
- 怎么初始化
  - 按照从核内到核外，从片内到片外的次序进行
```
![系统启动](/assets/ca/第七章图/系统启动.png)

# 处理器核初始化
## 处理器复位
- 从芯片引脚输入电平信号，将处理器内部的部分寄存器等状态置为预设值
  - 开始取指(LoongArch 处理器的第一条指令为0x1C000000)
- 复位输入是一个硬件初始化动作，至少需要完成最基本的芯片状态初始化(保证第一条指令取指执行)
  - 控制寄存器状态初始化，如处于核心态
  - 核内程序计数器(PC)初始化为0x1C000000
  - 片内对于0x1Cxxxxxx段地址的路由通路初始化
  - 清空流水线有效位避免混乱
  - **为什么不初始化更多硬件？**
    - 减轻软件负担，缩短系统启动时间，将更多硬件交由硬件自动处理
- 复位后，CPU内部只有取指通路一丝光亮，其余漆黑一片

### 首条指令
- 初始化控制器存期(用于控制处理器行为，如中断)
  - 模式信息CRMD(0x0)
  - 例外配置ECFG(0x4)
  - 例外入口地址EENTRY(0xC)
  - TLB重填例外入口地址TLBRENTRY(0x88)
- 初始化软件约定使用的通用寄存器
  - 栈指针：BIOS用的堆栈
  - 全局指针：BIOS用的地址空间
  - Stack、_gp为代码标号，编译时使用实际地址替换

```nasm
;手写代码
dli       t0, (0x7 << 16)
csrxchg   zero, t0, 0x4       ;设置中断入口间距
dli       t0, 0x1c001000
csrwr     t0, 0xc             ;设置普通例外入口地址
dli       t0, 0x1c001000
csrwr     t0, 0x88            ;设置TLB重填例外入口地址
dli       t0, (1 << 2)
csrxchg   zero, t0, 0x0       ;关中断执行
la        sp, stack           ;初始化栈指针

la        gp, _gp             ;初始化全局指针
```
```nasm
;编译器代码
lu12i.w       $r12, 0x70
csrxchg       $r0, $r12, 0x4
lu12i.w       $r12, 0x1c001
csrwr         $r12, 0xc
lu12i.w       $r12, 0x1c001
csrwr         $r12, 0x88
ori           $r12, $r0, 0x4
csrxchg       $r0, $r12, 0x0
lu12i.w       $r3, 0x90400
lu32i.d       $r3, 0
lu52i.d       $r3, $r3, 0x900
lu12i.w       $r2, 0x90020
ori           $r2, $r2, 0x900
lu32i.d       $r2, 0
lu52i.d       $r2, $r2, 0x900
```

### 模式信息控制寄存器
- 四种特权等级，CRMD(0x0)
  - 复位后默认为最高特权等级
![CRMD.PLV](/assets/ca/第七章图/CRMD.PLV.png)
- 例外使能控制，CRMD(0x0)
![CRMD.IE](/assets/ca/第七章图/CRMD.IE.png)

### 例外相关控制寄存器
- 例外入口支持向量化，ECFG(0x4)
  - VS=0，使用统一入口
![ECFG.VS](/assets/ca/第七章图/ECFG_VS.png)
- 例外入口地址，EENTRY(0xC)
![EENTRY](/assets/ca/第七章图/EENTRY.png)
- TLB重填例外入口地址，TLBENTRY(0x88)
![TLBENTRY](/assets/ca/第七章图/TLBENTRY.png)

## 外部调试接口初始化
> - 外部调试接口本身不是处理器核初始化的内容，但调试时需要尽早开始人机交互，方便软件调试
> - 输出
>   - 蜂鸣器
>   - 数码管显示
> - 输入输出
>   - 串口控制器
> - 显示器这样的设备相比执行是比较复杂的接口，在启动的后期才会使用
### 串口初始化
- 建立主机与被调试机之间的通信协议
  - 其中GS3_UART_BASE是串口控制器内部的寄存器
- **所谓驱动就是读写IO设备控制器**
  - **注意读写IO设备与读写存储器的不同**
```nasm
LEAF(inintserial)
li      a0, GS3_UART_BASE       ;加载串口设备基地址

li      t1, 128                 ;线路控制寄存器，写入0x80(128)表示后续的寄存器访问为分频寄存器
sb.b    t1, a0, 3               ;寄存器访问

li      t1, 0x12                ;配置串口波特率分频，当串口控制器输入频率为32MHz，并将串口通信
sb.b    t1, a0, 0               ;速率设置为115200时，分频方式为33000000/16/0x12=114583
li      t1, 0x12                ;由于串口通信有固定的起始格式，能够容忍传输两端一定的速率差异，
sb.b    t1, a0, 1               ;只要将传输两端的速率保持在一定范围内就可以保证传输的正确性

li      t1, 0x3
sb.b    t1, a0, 3               ;设置传输字符宽度为8，同时将后续的寄存器访问设置为正常寄存器

li      t1, 0
sb.b    t1, a0, 1               ;不使用中断模式

li      t1, 71
sb.b    t1, a0, 2
jirl    ra
END(initserial)
```

- 部分寄存器定义

![串口初始化寄存器1](/assets/ca/第七章图/串口初始化1.png)
![串口初始化寄存器2](/assets/ca/第七章图/串口初始化2.png)

#### 串口字符输入输出
```nasm
字符输出
LEAF(tgt_putchar)
dli     a1, GS3_UART_BASE       ;加载串口设备基地址
1:
ld.bu   a2, a1, 0x5             ;读取线路状态寄存器中的发送FIFO空标志
andi    a2, a2, 0x20
                                ;FIFO非空时等待
beqz    a2, 1b                  ;FIFO空时将通过a0传入的字符写入数据寄存器
st.b    a0, a1, 0
jirl    zero, ra, 0
END(tgt_putchar)
```
```nasm
字符输入
LEAF(tgt_getchar)
dli     a0, GS3_UART_BASE       ;加载串口设备基地址
1:
ld.bu   a1, a0, 0x5             ;读取线路状态寄存器中的接收FIFO有效标志
andi    a1, a1, 0x1
                                ;接收FIFO为空时等待
beqz    a1, 1b                  ;FIFO非空时将数据读出并放在a0寄存器中返回
ld.b    a0, a0, 0
jirl    zero, ra, 0
END(tgt_getchar)
```

## TLB初始化
- TLB用于虚实地址转换
- 从TLB来看，LoongArch的地址空间分为窗口映射和TLB映射
  - 在窗口映射中命中的地址使用窗口直接映射的方式
  - 其他未命中的地址全部使用TLB映射
  - **对于BIOS，基本使用非映射空间，无需TLB转换**
- 对于特别的地址转换需求，会借助TLB映射
  - 0xC0000000→0x40000000：对应显存等大IO空间

### LoongArch的虚拟地址空间
![LoongArch虚拟地址空间](/assets/ca/第七章图/LoongArch虚拟地址空间.png)

### TLB初始化代码
- 逐一清空每一个TLB项
  - 需要使用时再填入正确映射(**通过TLB Refill例外**)
- LoongArch中支持更简洁的初始化指令
  - INVTLB 0
- 越来越多的处理器不再需要软件初始化TLB

```nasm
LEAF(CPU_TLBClear)
dli     a3, 0                       ;循环控制变量
dli     a0, (1<<31) | (12<<24)      ;设置页大小为4K，31位为1表示无效
li      a2, 64                      ;TLB表项树
1:
csrwr   a0, 0x10                    ;将表项写入编号为0x10的TLBIDX寄存器
addi.d  a0, 1                       ;增加TLBIDX中的索引号
addi.d  a3, 1                       ;增加循环变量
tlbwr                               ;写TLB表项
bne     a3, a2, 1b
jirl    zero, ra, 0
END(CPU_TLBClear)
```
- TLB表项
  - 全相联
  - 32-64项
  - **关注E这一项**

![TLB表项](/assets/ca/第七章图/TLB表项(7).png)

- TLB控制寄存器
  - **关注NE这一项**

![TLB控制寄存器](/assets/ca/第七章图/TLB控制寄存器.png)
![TLB控制寄存器2](/assets/ca/第七章图/TLB控制寄存器2.png)

## Cache初始化
- 复位后BIOS刚开始启动时，处理器从非缓存空间开始执行
  - **上千拍**完成一条指令的取指和执行
  - Cache初始化后，跳转到Cache空间执行，全速流水
  - 尽早完成Cache初始化，使用Cache地址执行能够大大提升启动速度
- 从Cache来看，逻辑地址空间分为非缓存和缓存
  - LoongArch使用窗口进行映射，一般情况下按以下的设置方式
  - 0x90000000_00000000 - 0x9FFFFFFF_FFFFFFFF ：Cache访问方式
  - 0x80000000_00000000 - 0x8FFFFFFF_FFFFFFFF ：Uncache访问方式
  - 上述两个通过窗口来映射的空间，虽然访问的方式不同，但后面所映射的物理地址是相同的。区别在于用Cache方式访问时，如果在Cache中失效会自动到实际的物理位置取回数据；而用Uncache方式，则不会在Cache中查找。Uncache一般用于IO访问
  - 为什么用一个Cache地址进行写操作后，用同样的Uncache地址无法得到写入的值？
### 一级Cache初始化代码
```nasm
LEAF(godson2_cache_init)
li        a2, (1<<14)       ;64KB/4路，为Index的实际数量
li        a0, 0x0           ;a0表示当前的Index
1:
CACOP     0x0, a0, 0x0      ;对4路Cache分别进行写TAG操作
CACOP     0x0, a0, 0x0
CACOP     0x0, a0, 0x0
CACOP     0x0, a0, 0x0
addi.d    a0, a0, 0x40      ;每个Cache行大小为64字节
bne       a0, a2, 1b
jirl      ra
END(godson2_cache_init)
```
- 片上Cache容量越来越大，使得初始化事件越来越长
- 使用硬件资源在复位时对TLB、Cache等结构的初始化
  - 大幅减少系统启动的时间
  - 龙芯3A2000以后的处理器都支持硬件初始化Cache

### Cache指令及寄存器
- Cache在功能上对用户程序是透明的，但对OS是可见的
  - 以前硬件一般不对Cache做初始化，刚上电时Cache内容是乱的
  - 需要软件把Cache初始化成任何访问都不命中
- LoongArch的Cache指令，只能在核心态下使用
  - LoongArche的Cache指令通过OPCODE来表示具体的操作
  - 包括初始化和维护一致性的操作

### Cache算法配置
- 两位决定Cache算法
  - 0-Uncached、1-Cached、2-Uncache ACC
  - 对于窗口映射方式，由窗口配置寄存器的MAT域决定
  - 其他部分由TLB表项决定

![Cache算法1](/assets/ca/第七章图/Cache算法1.png)
![Cache算法2](/assets/ca/第七章图/Cache算法2.png)

## 处理器核初始化后
CPU内部一片光亮，除了串口和BIOS接口，多数“门窗”都没开

# 总线接口初始化
## 内存接口初始化
- 到此为止，BIOS从Flash读程序，写IO或控制寄存器
  - 相比Flash设备，内存接口的访问性能大大提升
- 内存接口的初始化与核内部件的初始化的差别
  - Cache/TLB的初始化主要是将内容设置为无效
  - 内存接口的初始化针对接口控制
- 内存接口初始化内容
  - 通过内存的SPD(并非必需)获取内存大小，类型，频率，延迟等各种信息
  - 根据所得到的内存信息对控制器和内存进行设置
  - 可能还有对于时序配合的信号训练
  - 对内容并不关心(ECC内存除外，不初始化内容会导致ECC错)

### 内存控制器配置代码
- 将预先定义好的值写入内存控制器的相应寄存器中
- 内存控制器将自动对内存进行初始化配置
  - 主要设置延迟、匹配阻抗等
- 初始设置后，会再对内存信号进行训练

```nasm
ddr2_config:
  add.d     a2, a2, s0          ;a2为调用该程序时传入的参数，与s0之和用于表示初始化参数在FLASH中的基地址
  dli       t1, DDR_PARAM_NUM   ;t1用于表示内存参数的个数
  addi.d    v0, t8, 0x0         ;t8用于表示内存参数的个数
1:
  ld.d      a1, 0x0(a2)         ;初始化的过程就是从FLASH中取数再写入内存控制器中的寄存器的过程
  st.d      a1, 0x0(v0)         
  addi.d    t1, t1, -1
  addi.d    a2, a2, 0x8
  addi.d    v0, v0, 0x8
  bnez      t1, 1b
```

## IO总线初始化
- 根据不同IO总线的需求进行针对性的初始化
- 通过初始化配置消除与具体实现相关的总线特性，保证软件兼容性
  - 信号定义，硬件实现各不相同：HyperTransport、PCIE等
  - 软件协议兼容PCI：配置空间、IO空间及Memory空间
- 龙芯3号中使用HyperTransport接口
  - 地址划分：确定配置访问、IO访问及Memory访问的地址
  - 设定DMA访问地址
  - 对总线频率、宽度进行重新设置

## 总线接口初始化后
- CPU内部一片光亮，“门窗”也已打开
- 但窗外还是一片漆黑

# 设备探测及驱动加载
## PCI协议下的工作流程
- 上世纪九十年代提出，软件结构被PCIE和HT等继承
  - 配置空间、IO空间、Memory空间
- 通过配置空间探测总线设备
  - 配置空间大部分情况下是对设备的属性进行刻画
  - 在完成设备探测之后基本不再使用配置空间
- 对总线上的各个设备所需的地址空间进行分配
  - IO、Memory空间是真正使用设备功能时的地址空间
  - 对于IO和内存统一编址的CPU，两者区别不大
- 使用分配好的空间对各个设备进行控制
  - 主要是驱动程序加载
- 特点：
  - 总线设备发生变化时无需修改软件
  - 同一总线支持多个相同设备不会导致冲突

## 设备探测
- 使用配置空间：以HT的配置空间为例
  - 高位地址访问不同设备，FDFE和FDFF是HT协议规定的
  - 通过总线号、设备号及功能号可以遍历总线设备
  - Offset是设备内地址偏移，索引设备空间寄存器
   ![HT总线配置访问](/assets/ca/第七章图/HT总线配置访问.png)
  - 通过规定的寄存器得到设备的唯一标识、设备类型等
    - Device ID、Vendor ID
  - 通过基址寄存器(BAR)获取并对设备空间进行配置
    - IO设备需要使用的内存空间

### 设备配置空间寄存器分布
- 一般放在IO控制器上(如PCIE控制器)
  - 所有设备的空间分布都一致