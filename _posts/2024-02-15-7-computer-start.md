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