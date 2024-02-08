---
title: 软硬件协同
tags: 计算机体系结构 学习笔记
permalink: /docs/ca/soft-hardware-together
aside:
    toc: ture
sidebar:
    nav: computer_architecture
---

# 软硬件协同
## 应用程序二进制接口(Application Binary Interface, ABI)
### 为什么要有ABI
> ABI定义了应用程序二进制代码中数据结构和函数模块的格式及其访问方式
- 计算机是一个复杂系统
  - 多层次的软件：引导程序、系统内核、动态链接库、应用……
  - 各种开发工具：汇编、C、C++
  - 对于如何在指令系统基础上构建软件系统，必须要有一些规范
- 规范些什么？
  - 处理器基础数据类型的大小、布局和对其要求
  - 寄存器使用约定
  - 函数调用约定
  - 目标文件和可执行文件格式
  - 栈布局
  - 程序装载和动态链接相关信息
  - 系统调用和标准库接口定义
  - 开发环境和执行环境
  - 等等
- 不同ABI差异的原因
  - 操作系统差异
  - 应用领域差异
  - 软硬件的发展需要
### 不同指令系统的ABI
#### MIPS ABI
- O32
  - 传统的MIPS约定，仍广泛用于嵌入式工具链和32位Linux中
- N64
  - 在64位处理器编程中使用新的正式ABI，改变了指针和long型整数的宽度，并改变了寄存器使用的约定和参数传递方式
- N32
  - 在64位处理器上执行的32位程序
  - 与N64的区别是指针和long型整数宽度为32位
  - 与O32相比可以使用64位寄存器，浮点寄存器数量从16个变32个
```c++
printf("%d %d %d %d",               // O32
        sizeof(int),                // 4 4 8 4
        sizeof(long),               // N64
        sizeof(long long),          // 4 8 8 8 
        sizeof(int*));              // N32
                                    // 4 4 8 4
```
#### LoongArch ABI
- LP32
  - 32位处理器上执行的32位程序(指针和数据都是32位)
- LPX32
  - 64位处理器上执行的64位程序，但指针和long型宽度为32位(指针32位、数据64位)
- LP64
  - 64位处理器上执行的64位程序(指针和数据都是64位)
#### X86 ABI
- i386
  - IA-32
- x86-64
- x32
  - 64位处理器上执行的32位程序
### 寄存器约定
- MIPS ABI寄存器约定
  - a*：函数参数
  - v*：函数返回值
  - t*：临时变量，子函数可改
  - s*：子函数不改的变量
  - k*：内核保留
  - gp：全局指针
  - sp：栈顶指针
  - fp：栈帧指针
  - ra：返回地址

![MIPS_ABI寄存器约定](./第四章图/mips_abi寄存器.png)

- LoongArch ABI寄存器约定
  - a0-a7：函数参数
  - v0/v1：函数返回值，是a0/a1别名
  - t*：临时变量，子函数可改
  - s*：子函数不改的变量
  - ra：返回地址
  - tp：线程指针
  - sp：栈顶指针
  - fp：栈帧指针

![Loong_Arch寄存器约定](./第四章图/LoongArch_ABI寄存器.png)

- LoongArch vs. MIPS
  - LoongArch 的3个ABI约定一致，MIPS O32背历史包袱
  - LoongArch给编译器留下更多可用寄存器
    - 取消了预留给内核的专用寄存器(\$k0/\$k1)，用便签寄存器来高效暂存数据
    - 取消了\$gp寄存器，用基于PC的运算指令实现重定位和动态链接
    - 取消了汇编暂存器(\$at)，宏指令不需要中转寄存器
    - 复用参数寄存器和返回值寄存器，\$a0/\$a1两个参数寄存器也被用于返回值寄存器
  - LoongArch 提供线程寄存器\$tp
    - 高效实现线程私有存储(Thread Local Storge, 简称TLS)

### 函数调用约定
#### MIPS ABI函数调用(N64)
- 定点参数经扩展后放到a0-a7，超过8个的通过栈传递
- 浮点参数通过浮点寄存器传递，定点+浮点共8个
- 结构体视为双字序列，通过寄存器传递
- 超过两个双字的返回值用指针(a0)通过栈传递
- 在栈中预留8个参数槽，对应8个参数，必要时用于参数存储，宽度固定为64位(N64)
- 栈16字节对齐，指针增量为16的倍数
#### LoongArch ABI函数调用

### 进程虚拟地址空间
- 地址空间划分
  - 主程序和动态库的代码与数据位于底端(但不为0, 地址0在多数操作系统中会被设为不可访问的地址，以便捕获空指针访问)
  - 堆空间自底向上增长，管理程序运行中的动态分配的内存数据
  - 用户栈自顶向下增长，存放函数中的临时变量和数据(临时变量、子函数参数、返回地址)
  - 动态链接库位于堆和栈之间

![虚拟地址空间](./第四章图/虚拟地址空间.png)

### 栈帧布局
- 栈——代码的例化
  - 保存上下文
  - 传递参数
  - 保存临时变量、非静态局部变量
- 使用分类
  - 简单叶子函数：可以不用栈
  - 编译时可确定栈大小的：只用sp
  - 运行时改变栈大小的：fp+sp
- 栈帧构成
  - 参数区+临时变量区+子函数参数
  - 编译器优化后不一定保留参数区

![栈帧布局](./第四章图/栈帧布局.png)

### LoongArch相关示例
#### 参数传递
```c
extern void abort(void);
int fun(double a1, double a2, double a3, double a4, double a5, double a6, double a7, double a8, double a9, int a10, double a11, int a12){
    if(a9 != a11) abort;
    return 0;
}
```
对应的LoongArch汇编代码
```
fun:
    movgr2fr.d $f0, $a0                 //$f0是参数a9，从$a0获得
    movgr2fr.d $f1, $a2                 //$f1是参数a11，从$a2获得
    fcmp.ceq.d $fcc0, $f1, $f0          //比较a9与a11
    bceqz      $fcc0, .L8
    move       $a0, zero
    jr         $ra
.L8
    addi.d     $sp, $sp, -16
    st.d       $ra, $sp, 8
    bl         %plt(abort)
    ld.d       $ra, $sp, 8
```

![LoongArch参数传递](./第四章图/LoongArch参数传递.png)

#### 可变参数传递
```c
struct Ss{
    char c1, c2;
}a3 = {3, 4};
int fun(double a1, ...);                //可变参数根据整形调用规范传参
int test(){
    return fun(1, (float)2, a3, (long double)5, (float)6, (short)7, (int)8, (float)9, (int)10);
}
```

![LoongArch可变参数传递](./第四章图/LoongArch可变参数传递.png)

#### 栈帧生成
- 简单栈帧
```c
int simple(int a, int b){
    return((a & 0xff) + b);
}
```
> gcc -O2 -fno-omit-frame-pointer -S 编译后
```
simple:
    addi.d      $sp, $sp, -16           // 设立一个16字节的栈帧
    st.d        $fp, $sp, 8             // 在偏移8的位置保存$fp寄存器
    addi.d      $fp, $sp, 16            // 把$fp指向刚进入函数时的$sp
    ld.d        $fp, $sp, 8             // 恢复$fp
    bstrpick.w  $a0, $a0, 7, 0          // 将$a0的0-7位0扩展到32位
    add.w       $a0, $a0, $a1
    addi.d      $sp, $sp, 16            // 释放栈帧
    jr          $ra
```
> gcc -O2 -S 编译后
```
simple:
    bstrpick.w  $a0, $a0, 7, 0          
    add.w       $a0, $a0, $a1
    jr          $ra
```

- 常规栈帧
```c
extern int nested(int a, int b, int c, int d, int e, int f, int g, int h, int i);
int normal(void){
    return nested(1, 2, 3, 4, 5, 6, 7, 8, 9);
}
```
> gcc -O2 -S
```
normal:
    addi.d  $sp,$sp,-32           # 分配栈帧
    addi.w  $t0,$zero,9           # 0x9
    stptr.d $t0,$sp,0             # 第9个参数存入栈
    addi.w  $a7,$zero,8           # 0x8
    addi.w  $a6,$zero,7           # 0x7
    addi.w  $a5,$zero,6           # 0x6
    addi.w  $a4,$zero,5           # 0x5
    addi.w  $a3,$zero,4           # 0x4
    addi.w  $a2,$zero,3           # 0x3
    addi.w  $a1,$zero,2           # 0x2
    addi.w  $a0,$zero,1           # 0x1
    st.d    $ra,$sp,24            # 保存ra
    bl      %plt(nested)
    ld.d    $ra,$sp,24            # 恢复ra
    addi.d  $sp,$sp,32            # 释放栈帧
    jr  $ra                                  
```

- 需要$fp的栈帧
```c
#incldude<alloca.h>
extern long nested(long a, long b, long c, long d, long e, long f, long g, long h, long i);
long dynamic(void){
    long *p = alloca(64);
    p[0] = 0x123;
    return nested((long)p, p[0], 3, 4, 5, 6, 7, 8, 9);
}
```
> gcc -O2 -S
```
dynamic:
    addi.d   $sp,$sp,-32
    st.d     $fp,$sp,16        # 保存fp
    st.d     $ra,$sp,24        # 保存ra
    addi.d   $fp,$sp,32        # fp = sp in(fp指向入口时的sp)
    addi.d   $sp,$sp,-64       # alloca修改sp
    addi.d   $a0,$sp,16        # [sp+16,sp+80] 为分配的alloca空间      
    addi.w   $t0,$zero,291     # 0x123
    stptr.d  $t0,$a0,0
    addi.w   $t0,$zero,9       # 0x9
    stptr.d  $t0,$sp,0         # sp到sp+16为调子函数参数区
                               # 
    addi.w   $a7,$zero,8       # 0x8
    addi.w   $a7,$zero,7       # 0x7
    addi.w   $a7,$zero,6       # 0x6
    addi.w   $a7,$zero,5       # 0x5
    addi.w   $a7,$zero,4       # 0x4
    addi.w   $a7,$zero,3       # 0x3
    addi.w   $a1,$zero,291     # 第二个参数0x123
    bl       %plt(nested)      # 修改ra
    addi.d   $sp,$fp,-32
    ld.d     $ra,$sp,24
    ld.d     $fp,$sp,16
    addi.d   $sp,$sp,32
    jr  $ra                                                                   
```
#### 大结构体返回

```c
typedef struct vint8 {
    int v[8];
}vint8_t;
extern vint8_t get_vint8(int v);
int sum_vim8(int v){
    vint8_t r;
    int sum = 0;
    r = get_vint8(v);
    for (int i = 0; i < 8; i++){
        sum += r.v[i];
    }
    return sum;
}
```

```
sum_vint8:
    addi.d      $sp, $sp, -48
    or          $a1, $a0, $zero         # v变成第二个参数
    or          $a0, $sp, $zero         # 第一个参数为返回值
    st.d        $ra, $sp, 40
    bl          %plt(get_vint8)         # 调用get_vint8
    or          $t0, $sp, $zero         # $t0指向结构体
    addi.d      $t2, $sp, 32            
    or          $t1, $zero, $zero       # $t1 = sum = 0
    .align      3, 54525952, 4
.L2:                                    # for循环
    ldptr.w     $a0, $t0, 0             # 取出r.v[i]
    addi.d      $t0, $t0, 4             # 下一个v[i]地址
    add.w       $a0, $a0, $zero         # sum += r.v[i]
    or          $t1, $a0, $zero         
    bne         $t0, $t2, .L2           # 没到末尾就循环
    ld.d        $ra, $sp, 40
    addi.d      $sp, $sp, 48
    jr          $ra
```

![大结构体返回](./第四章图/大结构体返回.png)

> 寄存器a0存放指向在栈中的返回结构，此例中为局部变量r
> 
> 如果返回值要写到全局变量，则先写到栈中，再搬



## 六种常见的上下文切换场景
### 函数调用
- 用户主动发起的上下文改变
  - 与普通跳转的差异：调用函数后要能返回原上下文
- 保存恢复的内容
  - 部分寄存器：\$sp/\$fp等栈帧相关寄存器，ABI约定应保存的内容(如\$s0\~\$s8这种)
- 软硬件协同实现
  - 通过ABI约定哪些寄存器的值在调用一个函数后不会发生变化，哪些可能发生变化，编译器根据约定生成调用、返回代码以及用于保存恢复寄存器的函数头和函数尾代码
  - 不同架构可提供不同的硬件支持
#### 函数调用的硬件支持
- RISC处理器常见做法
  - 硬件只提供最小必要支持：跳转，保存返回地址到寄存器
    - LoongArch：
    - `call: jirl $ra, <target addr reg>, offset`
    - `return: jirl $zero, $ra, 0`
- X86的实现
  - 提供Call/Ret指令，Call指令除了跳转，还将返回地址保存在栈上，执行调用权限检查等
  - 提供push/pop类指令方便操作栈内容
- Sparc的实现
  - 寄存器窗口机制
    - 8个全局寄存器
    - 2-32个寄存器窗口
      - 每个窗口16个寄存器
    - 任何时候可见32个
      - 8个全局寄存器
      - 8个局部(当前窗口)
      - 8个输入(当前窗口)
      - 8个输出(下个窗口)
  - 通过save/restore指令滑动窗口
    - save时当时函数的输出变成下一个函数的输入，函数返回时用restore恢复原窗口
### 例外和中断
- (通常)对用户透明：保存和恢复**所有可能被改变的**用户可见的上下文内容
  - 可能改变的寄存器(通用寄存器、处理器状态寄存器)
  - 发生异常的地址、中断屏蔽位、当前指令、特权状态等现场信息(以应付嵌套异常)
  - 特定异常的相关信息，比如访存异常涉及的虚拟地址等
- 软硬件协同实现
  - 硬件需提供必要支持：不同特权等级状态以及切换、中断和异常的检测、优先级和屏蔽、异常相关信息；不同架构可提供更多支持来提高处理效率
  - 软件(一般由操作系统内核管理)：进行异常原因识别、上下文保存、具体异常和中断处理程序调用、上下文恢复和返回等
#### 异常和中断的硬件支持
- 异常中断的来源识别
  - 常规做法：OS通过读取相关状态寄存器和外设寄存器找到确切的原因，直接保存所有可能被内核修改的上下文状态，然后调用相应的处理函数，最后再恢复所有状态
  - 向量中断：硬件能为不同的异常和中断自动跳转到不同入口，避免慢速的IO读
- 高频异常或者中断的支持
  - 提供特殊入口和处理函数，尽量降低上下文保存开销
  - MIPS预留两个通用寄存器给TLB refill，后者10余条指令完成处理，省去保存和恢复寄存器的开销
  - X86完全由硬件处理TLB refill(page walker)
  - LoongArch通过便签寄存器来支持高频异常，避免占用通用寄存器
### 系统调用
#### 系统调用简介
- 内核给用户程序提供的服务子程序
  - 进程控制：fork, clone, execve, exit, getpid, pause, wait等
  - 文件操作：create, open, close, read, write, access, chdir, chmod等
  - 系统控制：ioctl, ioperm, gettimeofday等
  - 内存管理：brk, mmap, sync等
  - 网络管理：gethostname, socket, connect, send等
  - 用户管理：getuid, getgid等
  - 进程间通信：signal, msgsnd, pipe, semctl, shmctl等
  - 用strace命令可以跟踪一个进程的系统调用
- 两个基本要求
  - **安全性**：核心态下运行，所有参数都应检查，避免受攻击，保证自身的安全
  - **兼容性**：避免改应用，保护生态
- 约定
  - 调用号放在\$a7寄存器中
  - 至多7个参数通过\$a0\~\$a6寄存器进行传递
  - 返回值存放在\$a0/\$a1寄存器
  - 系统调用保存\$s0\~\$s8寄存器的内容，不保证保持参数寄存器和暂存寄存器的内容
#### 系统调用例子

![LoongArch系统调用](./第四章图/LoongArch系统调用.png)

```
//hello.S
    .section .rodata
    .align 3
.hello:
    .ascii "Hello World!\012\000"
    .text
    .align 3
    .global main
main:
    li          $a7, 64         # write的系统调用号64
    li          $a0, 1          # fd == 1 是stdout的文件描述符号
    la.local    $a1, .hello     # 字符串地址
    li          $a2, 14         # 字符串长度
    syscall     0x0
    jr          $ra             # 返回
```
纯汇编的HelloWorld
- 将字符串写到标准输出
  - a0：文件描述符
  - a1：写的内容
  - a2：写的字节数
  - a7：write系统调用号
  - 调用完a0返回实现写的数量
- 退出进程
  - a0：返回值
  - a7：exit系统调用
#### 系统调用的实现
> 作为一种特殊的异常处理，有类似的流程
> - syscall指令触发异常，硬件将CPU切换到核心态，设置相应状态，跳转到处理程序
> - 保存通用寄存器、sp寄存器指向内核用的栈地址
> - 根据系统调用号查表得到相应的系统调用函数，执行
> - 通过异常返回流程(ERTN)返回用户态

### 进程
- 进程是程序在特定数据集合上的一个运行实例
  - 由**程序**、**数据集合**和**进程控制块**三部分组成
    - 进程控制块记录每个进程运行过程中虚拟内存地址、打开文件、锁和信号等资源的情况
  - 进程切换为每个进程提供虚拟的独立的CPU和内存地址空间
- 保存恢复的内容
  - 不能多进程共享的处理器状态信息：通用寄存器、处理器模式和状态、页表基址、PC、eflags等用户态专用寄存器
  - 内存有备份的非共享状态信息可直接丢弃：Flush TLB
- 软硬件协同实现
  - 主要以软件为主，硬件可协助提高效率
- 硬件支持
  - 硬件资源实现多进程共享
    - TLB项增加进程标志位来实现多进程共享，避免切换时flush
  - 硬件进程切换
    - X86的TS段和硬件切换支持，未被现代系统采用
  - lazy切换
    - 部分资源可在使用到时再切换：LoongArch的浮点和向量可关闭和使能，关闭时使用触发异常
### 线程
- 线程是程序代码的一个执行路径
  - 同一个进程的**多个线程共享地址空间**，所以线程切换时共享内存不切换
  - 一份计算资源虚拟化为多份
- 保存恢复的内容
  - 通用寄存器(包括栈指针等)、线程私有存储(TLS)基址
- 软硬件协同实现
  - OS和/或线程库负责切换
  - 根据情况实现内核级、用户级或者混合线程
- 硬件支持
  - 线程私有存储(TLS)的切换支持
    - MIPS通过系统调用设置TLS，通过rdhwr触发异常来读取TLS
    - LoongArch预留一个$tp寄存器指向线程的TLS
### 虚拟机
- 虚拟机模拟一台物理电脑
  - 将一台物理电脑虚拟成若干虚拟电脑
- 保存恢复的内容
  - 物理资源少于虚拟资源时需要保存恢复
  - 除了用户态的通用寄存器，可能还包括部分特权态资源
- 软硬件协同实现
  - 可以完全由软件实现
  - OS内核和硬件配套可以提高效率：现代虚拟机效率接近物理机的100%
### 六种上下文切换保存和恢复的内容

![六种上下文切换对比](./第四章图/六种上下文切换.png)

## 同步和通讯
> 现代操作系统的关键特性：**多任务**
>
> 当同一块数据被多段代码并发访问时需要同步
>
> 多数代码隐含原子性实现的假定
>   - 读-改-写
>   - 队列插入、删除
>   - 红黑树插入、删除
>
> 并发访问可能来自
> - 另一个CPU核上的线程(多处理器系统中同时运行的多个代码块)
> - 处于中断上下文的线程(由于中断引起并发代码段)
> - 由于线程调度引起的多线程并发执行
### 基于锁的同步
> - 将可能并发访问同一块数据的代码(对数据进行原子操作的程序段)定义为临界区
> - 进入临界区前先申请锁(并加锁)
> - 申请失败者被阻塞
> - 持有锁的线程离开临界区后释放锁
#### 锁的类型
##### 自旋锁(selfspin)
```assembly
    la.local            $a0, lock
selfspin:
    ll.w                $t0, $a0, 0
    bnez                $t0, selfspin
    li                  $t1, 0x1
    sc.w                $t1, $a0, 0
    beqz                $t1, selfspin
    <Critical section>
    st.w                $zero, lock
    ...
```

- 循环等待直至得到锁
- 在多处理器环境下使用
- 传统的自旋锁不够公平
  - 不体现先后次序
  - 可能出现饿死的情况
- 排队自旋锁
  - 增加ticket核serving_now
  - 原子性的取号
  - 根据当前服务号判断是否抢到
  - 解锁时将服务号+1

##### 互斥锁(mutex)
```c
//include/linux/mutex.h
struct mutex{
    atomic_t             count;
    spinlock_t           wait_lock;
    struct list_head     wait_list;
    struct task_struct   *owner;   
    ...
}

//include/asm-generic/mutex-dec.h
static inline void
_mutex_fastpath_lock(atomic_t *count, void(*fail_fn)(atomic_t *)){
    if(atomic_dec_return(count) < 0){
        fail_fn(count);
    }    
}
static inline void
_mutex_fastpath_unlock(atomic_t *count, void(*fail_fn)(atomic_t *)){
    if(atomic_inc_return(count) <= 0){
        fail_fn(count);
    }
}
```
- 由count决定锁的状态
  - =1：可用
  - =0：已锁定
  - <0：已锁定，多个加锁请求
- 没取到锁则睡眠等待
- 解锁时检查等待队列
##### 其他锁
- 读写锁(rwlock)
  - 允许一个写线程或多个读线程访问共享数据，但读写之间互斥
  - 可允许多个读同时访问，使读数据的效率更高
  - 适用于共享数据频繁被多个线程同时读但很少修改的情况
- 顺序锁(seqlock)
  - 写优先(spinlock不分读写、rwlock读优先)
  - 写进入和退出时对锁变量作原子性递增
  - 读开始和结束时取锁变量进行判断，相等且不为奇数时读不需要重试
- RCU锁(Read Copy Update)
  - 共享数据通过指针引用，每个写都要创建一个副本
  - 旧的副本没有在引用后删除
  - 相比顺序锁，不需要让读进程重试
### 非阻塞的同步
- 锁的问题
  - 若持有锁的线程死亡、阻塞或死循环，其他等待锁的线程可能会永远等待下去
  - 获取锁和释放锁的代价
  - 并发度受限，细粒度锁设计难度大，持有锁的线程因时间片中断或页错误而被取消调度时其他线程需要等待
- 事务内存(Transactional Memory)的思想
  - 把临界区代码定义为事务
  - 尝试性的执行事务代码
  - 执行过程中动态检测冲突
  - 根据冲突检测结果提交或取消事务
  - LL/SC可以理解为大小仅一个word的事务操作
