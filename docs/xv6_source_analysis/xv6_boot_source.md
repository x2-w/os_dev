---
icon: lucide/layers
---

本文对 xv6 中的启动部分代码进行分析，涉及文件 `entry.S `, `start.c` 和 `main.c`

## 启动文件 entry.S

```asm
        # qemu -kernel loads the kernel at 0x80000000
        # and causes each hart (i.e. CPU) to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
.global _entry
_entry:
        # set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + ((hartid + 1) * 4096)
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        # jump to start() in start.c
        call start
spin:
        j spin

```

这是 xv6 操作系统的 **入口汇编代码**（`kernel/entry.S`），负责完成从 QEMU 加载到跳转到 C 代码的过渡

### 注释说明

```asm
# qemu -kernel loads the kernel at 0x80000000
# and causes each hart (i.e. CPU) to jump there.
# kernel.ld causes the following code to
# be placed at 0x80000000.
```

- **QEMU 加载行为**：`qemu-system-riscv64 -kernel kernel` 会将内核文件加载到物理内存地址 `0x80000000`（RISC-V 规范定义的 RAM 起始地址）。

- **多核启动**：`each hart` 指所有硬件线程（Hardware Thread，即 CPU 核心）。QEMU 会让所有 CPU 同时从该地址开始执行。

- **链接器保证**：`kernel.ld` 链接脚本通过 `. = 0x80000000;` 和 `.text : { kernel/entry.o(_entry) ... }` 确保 `_entry` 位于 `0x80000000`（见链接脚本 kernel.ld 分析）。

### 段和符号定义

```asm
.section .text
```

- **代码段声明**：将后续代码放入 `.text` 段（可执行代码段）。链接脚本会将 `.text` 放在 `0x80000000` 起始处。

```asm
.global _entry
```

- **全局符号导出**：让链接器能够看到 `_entry` 符号。链接脚本中的 `ENTRY(_entry)` 依赖这个符号来确定入口点。

```asm
_entry:
```

- **入口标签**：QEMU 跳转的目标地址。所有 CPU 核启动后都从这里开始执行。


### 栈设置（C 语言运行环境准备）

```asm
# set up a stack for C.
# stack0 is declared in start.c,
# with a 4096-byte stack per CPU.
```

- **C 语言依赖**：C 代码需要栈来存储局部变量、返回地址等。汇编阶段必须手动设置栈指针（`sp` 寄存器）。

- **栈定义**：`stack0` 是在 `start.c` 中定义的数组（`char stack0[4096 * NCPU]`），为每个 CPU 分配 4KB 栈空间。

```asm
# sp = stack0 + ((hartid + 1) * 4096)
la sp, stack0
```

- **加载栈基地址**：`la`（Load Address）伪指令将 `stack0` 的地址加载到 `sp`（Stack Pointer）寄存器。

- **RISC-V 寄存器**：`sp` 是 x2 寄存器，专门用作栈指针。

```asm
li a0, 1024*4
```

- **加载栈大小**：`li`（Load Immediate）将 `4096`（4KB）加载到 `a0` 寄存器（函数参数/临时寄存器）。

- **计算**：`1024*4` 是汇编时常量，等于 4096。

```asm
csrr a1, mhartid
```

- **读取 CPU ID**：`csrr`（CSR Read）从 `mhartid`（Machine-mode Hart ID）控制状态寄存器读取当前 CPU 的 ID。

- **多核区分**：CPU 0 得到 0，CPU 1 得到 1，以此类推。这样每个 CPU 可以使用不同的栈，避免冲突。

```asm
addi a1, a1, 1
```

- **计算偏移倍数**：`hartid + 1`。例如：

    - CPU 0 → 1

    - CPU 1 → 2

- **原因**：栈向**低地址增长**（RISC-V 调用约定），所以栈顶应该是高地址。通过 `+1`，CPU 0 的栈顶是 `stack0 + 4096`，CPU 1 是 `stack0 + 8192`，以此类推。

```asm
mul a0, a0, a1
```

- **计算偏移量**：`a0 = 4096 * (hartid + 1)`。

- **结果**：

    - CPU 0：偏移 4096

    - CPU 1：偏移 8192

    - CPU 2：偏移 12288


```asm
add sp, sp, a0
```

- **设置最终栈顶**：`sp = stack0 + 4096*(hartid+1)`。

- **栈布局示意**：

  ```text
  高地址
  stack0 + 12288 [CPU 2 栈顶 (sp)]
            ...    [CPU 2 栈空间 (4KB)]
  stack0 + 8192  [CPU 1 栈顶 (sp)]
            ...    [CPU 1 栈空间 (4KB)]
  stack0 + 4096  [CPU 0 栈顶 (sp)]
            ...    [CPU 0 栈空间 (4KB)]
  stack0         [数组起始]
  低地址
  ```

- **保护**：`stack0` 本身不被用作栈，而是作为第 0 个 CPU 的栈底保护（或留给特殊用途）。

### 跳转到 C 代码

```asm
# jump to start() in start.c
call start
```

- **调用 C 函数**：`call` 指令完成两件事：

    1. 将返回地址存入 `ra`（x1，Return Address）寄存器

    2. 跳转到 `start` 函数的地址

- **start.c**：这是第一个 C 语言编写的初始化函数，负责：

    - 设置页表（虚拟内存）

    - 初始化各种硬件（PLIC、CLINT 等）

    - 创建第一个进程（init）

    - 进入调度器


### 安全保护（无限循环）

```asm
spin:
        j spin
```

- **死循环保险**：如果 `start()` 函数意外返回（理论上不应发生，因为 `start` 最后会调用调度器永不返回），CPU 会执行到这里。

- **防止跑飞**：`j spin`（jump）是无限循环，避免 CPU 继续执行随机内存内容导致不可预测行为。


### 执行流程总结

```text
QEMU 加载内核到 0x80000000
        │
        ▼
所有 CPU 同时从 _entry (0x80000000) 开始执行
        │
        ▼
每个 CPU 读取自己的 hartid
        │
        ▼
计算专属栈顶：stack0 + (hartid+1)*4096
        │
        ▼
设置 sp 寄存器
        │
        ▼
call start ──► 进入 C 代码初始化
        │
        ▼
   start() 永不返回
   （进入调度器循环）
        │
        ▼
   若意外返回 ──► spin 死循环（保险）
```

### 关键 RISC-V 概念

| 概念 | 说明 |
|:--:|:--|
| **hart** | Hardware Thread，硬件线程，即 CPU 核心。xv6 支持多核（SMP）。 |
| **mhartid** | Machine-mode Hart ID，每个核唯一的 ID（从 0 开始）。 |
| **sp (x2)** | Stack Pointer，栈指针，指向栈顶（通常是已分配栈空间的最高地址）。 |
| **栈向下增长** | RISC-V 调用约定规定 `sp` 减少是压栈，增加是出栈。 |
| **M-mode** | Machine Mode，RISC-V 的最高特权级。QEMU 启动时 CPU 处于此模式。 |
| **call** | 伪指令，等价于 `auipc ra, ...` + `jalr ra, ...`，保存返回地址并跳转。 |

这段代码是操作系统启动的"桥梁"——从汇编的裸机环境过渡到 C 语言的高级抽象环境。

## 初始化 start.c

```c
// entry.S needs one stack per CPU.
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];

// entry.S jumps here in machine mode on stack0.
void
start()
{
    // set M Previous Privilege mode to Supervisor, for mret.
    unsigned long x = r_mstatus();
    x &= ~MSTATUS_MPP_MASK;
    x |= MSTATUS_MPP_S;
    w_mstatus(x);

    // set M Exception Program Counter to main, for mret.
    // requires gcc -mcmodel=medany
    w_mepc((uint64)main);

    // disable paging for now.
    w_satp(0);

    // delegate all interrupts and exceptions to supervisor mode.
    w_medeleg(0xffff);
    w_mideleg(0xffff);
    w_sie(r_sie() | SIE_SEIE | SIE_STIE);

    // configure Physical Memory Protection to give supervisor mode
    // access to all of physical memory.
    w_pmpaddr0(0x3fffffffffffffull);
    w_pmpcfg0(0xf);

    // ask for clock interrupts.
    timerinit();

    // keep each CPU's hartid in its tp register, for cpuid().
    int id = r_mhartid();
    w_tp(id);

    // switch to supervisor mode and jump to main().
    asm volatile("mret");
}
```

这是 xv6 内核启动的 C 语言初始化函数（`kernel/start.c`），负责**从机器模式（M-mode）切换到监管者模式（S-mode）**，并完成硬件初始化

### start 函数入口

```c
void
start()
{
```

- **入口函数**：由 `entry.S` 中的 `call start` 跳转过来。此时 CPU 仍处于 **M-mode（机器模式）**，这是 RISC-V 的最高特权级。

### start 函数功能

#### 1. 设置前一个特权级（为 mret 做准备）

```c
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
```

- **读取 mstatus**：`r_mstatus()` 是内联函数，读取 `mstatus`（Machine Status）寄存器。该寄存器保存处理器的状态信息，包括当前和前一个特权级。

```c
  x &= ~MSTATUS_MPP_MASK;
```

- **清除 MPP 位**：`MSTATUS_MPP_MASK` 是掩码（通常是位 11:12）。`MPP`（Machine Previous Privilege）记录进入 M-mode 之前的特权级。这里先清零，准备设置新值。


```c
  x |= MSTATUS_MPP_S;
```

- **设置 MPP 为 S-mode**：`MSTATUS_MPP_S` 将 MPP 设置为 **Supervisor（监管者模式，值 01）**。这是关键步骤：当执行 `mret` 时，CPU 会根据 MPP 的值决定返回到哪个特权级。

```c
  w_mstatus(x);
```

- **写回 mstatus**：`w_mstatus()` 将修改后的值写回寄存器。


**作用**：欺骗 CPU，让它以为我们是从 S-mode "陷入"到 M-mode 的，这样执行 `mret` 时会"返回"到 S-mode。

#### 2. 设置异常程序计数器（跳转目标）

```c
  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);
```

- **设置 mepc**：`mepc`（Machine Exception Program Counter）是异常返回地址寄存器。`w_mepc()` 将 `main` 函数的地址写入其中。

- **mret 行为**：执行 `mret` 时，PC（程序计数器）会被设置为 `mepc` 的值，即跳转到 `main()` 函数。

- **编译器要求**：注释提到需要 `-mcmodel=medany`，因为 `main` 地址在 0x80000000 以上（高位地址），需要编译器生成正确的地址加载指令。


#### 3. 禁用分页（虚拟内存）

```c
  // disable paging for now.
  w_satp(0);
```

- **关闭 MMU**：`satp`（Supervisor Address Translation and Protection）是 S-mode 的页表基址寄存器。写入 0 表示禁用虚拟内存，此时 S-mode 使用物理地址（直接映射）。

后续在 `main()` 中会重新设置页表，这里先关闭以避免地址翻译混乱。


#### 4. 委托中断和异常（关键步骤）

```c
  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
```

- **委托异常**：`medeleg`（Machine Exception Delegation）控制哪些异常委托给 S-mode。`0xffff` 表示将所有 16 种异常（如页错误、非法指令、断点等）都委托给 S-mode 处理。

- **如果不委托**：异常会在 M-mode 处理，需要 M-mode 的异常处理程序，增加复杂性。

```c
  w_mideleg(0xffff);
```

- **委托中断**：`mideleg`（Machine Interrupt Delegation）控制哪些中断委托给 S-mode。`0xffff` 表示将所有中断（软件中断、定时器中断、外部中断等）委托给 S-mode。

```c
  w_sie(r_sie() | SIE_SEIE | SIE_STIE);
```

- **启用 S-mode 中断**：`sie`（Supervisor Interrupt Enable）是 S-mode 的中断使能寄存器。

    - `SIE_SEIE`：外部中断使能（External Interrupt Enable）

    - `SIE_STIE`：定时器中断使能（Timer Interrupt Enable）

该语句是读取当前 `sie` 值，置位这两个中断位，写回。

#### 5. 配置物理内存保护（PMP）

```c
  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
```

- **PMP 地址寄存器**：`pmpaddr0` 定义第 0 个 PMP 区域的上界地址。`0x3fffffffffffff`（即 2^54 - 1）覆盖了整个 64 位地址空间（RISC-V SV39 只使用低 39 位，但这里设置最大值以确保覆盖所有物理内存）。

- **作用**：定义 PMP 区域的结束地址。

```c
  w_pmpcfg0(0xf);
```

- **PMP 配置寄存器**：`pmpcfg0` 配置第 0 个 PMP 区域的属性。`0xf`（二进制 1111）通常表示：

    - 位 0（R）：读允许

    - 位 1（W）：写允许

    - 位 2（X）：执行允许

    - 位 3（A）：地址匹配模式（TOR - Top of Range）

- **效果**：允许 S-mode 和 U-mode 读写执行整个物理内存。没有 PMP 配置，S-mode 可能无法访问某些内存区域。


#### 6. 初始化定时器

```c
  // ask for clock interrupts.
  timerinit();
```

- **定时器初始化**：调用函数配置 CLINT（Core Local Interruptor）的定时器，设置第一次时钟中断，并启用定时器中断。

- **抢占式多任务**：时钟中断是操作系统进行进程调度的基础（时间片轮转）。


#### 7. 保存 CPU ID

```c
  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
```

- **读取 Hart ID**：`r_mhartid()` 读取 `mhartid` 寄存器，获取当前 CPU 的硬件线程 ID（0, 1, 2...）。

```c
  w_tp(id);
```

- **写入 tp 寄存器**：`tp`（Thread Pointer，x4）是通用寄存器，通常用于存储线程本地数据。这里用来保存 CPU ID，方便后续 `cpuid()` 函数快速获取当前 CPU 编号，而不需要频繁访问 CSR。


#### 8. 切换到 S-mode 并跳转

```c
  // switch to supervisor mode and jump to main().
  asm volatile("mret");
```

- **mret 指令**：机器模式返回（Machine-mode Return）。

- **执行效果**：

    1. **特权级切换**：根据 `mstatus.MPP` 的值（之前设置为 S-mode），将当前特权级切到 S-mode

    2. **跳转**：PC（程序计数器）设置为 `mepc` 的值（之前设置为 `main`）

    3. **中断状态**：根据 `mstatus.MPIE` 恢复中断使能状态

- **volatile**：防止编译器优化掉这条指令。


**执行后**：

- CPU 现在运行在 **S-mode（监管者模式）**

- 执行流跳转到 `main()` 函数（`kernel/main.c`）

- 继续内核初始化（控制台、页表、调度器等）


### 执行流程总结

```text
entry.S (M-mode, 0x80000000)
    │
    ▼
start() (M-mode)
    |
    |-> 设置 mstatus.MPP = S-mode（准备降级）
    |-> 设置 mepc = main（设置返回地址）
    |-> 关闭页表（satp = 0）
    |-> 委托所有中断/异常给 S-mode
    |-> 配置 PMP（允许 S-mode 访问全部内存）
    |-> 初始化定时器（时钟中断）
    |-> 保存 hartid 到 tp 寄存器
    |
    ▼
mret 指令
    |
    |
    |-> 特权级：M-mode ──► S-mode
    |-> PC 跳转：start ──► main()
    |
    ▼
main() (S-mode, 继续初始化)
```

### 关键 RISC-V 概念

|寄存器/指令|全称|作用|
|:--|:--|:--|
|**mstatus**|Machine Status|保存全局状态，包含 MPP（前一个特权级）|
|**mepc**|Machine Exception PC|异常/中断返回地址|
|**mret**|Machine Return|从 M-mode 返回，根据 MPP 切换到对应特权级|
|**medeleg/mideleg**|Machine Exception/Interrupt Delegation|将异常/中断委托给低特权级处理|
|**satp**|Supervisor Address Translation and Protection|S-mode 页表基址|
|**PMP**|Physical Memory Protection|M-mode 的内存访问控制|
|**tp**|Thread Pointer (x4)|线程指针，这里复用为 CPU ID 存储|

这段代码是 **RISC-V 操作系统启动的标准模板**，完成了从 Bootloader（M-mode）到内核（S-mode）的关键特权级转换。

## 主函数 main.c

```c
volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();
}
```

`main` 函数主要负责**多核 CPU 的协调初始化**和**内核子系统**的启动。

### 全局同步变量

```c
volatile static int started = 0;
```

- **`static`**：限制作用域在本文件，防止外部链接。

- **`volatile`**：告诉编译器不要优化对此变量的访问。多核环境下，一个 CPU 写入 `started`，其他 CPU 需要立即看到变化，不能被编译器缓存到寄存器。

- **用途**：作为主核（CPU 0）与从核（其他 CPU）之间的**同步标志**。主核完成初始化后置为 1，通知从核可以继续。


### main 函数入口

```c
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
```

- **执行上下文**：由 `start()` 通过 `mret` 跳转到此处，此时：

    - 特权级：**S-mode（Supervisor）**

    - 环境：分页已关闭（物理地址），中断已委托

    - **所有 CPU 同时进入**（每个 hart 独立执行）


### 主核（CPU 0）初始化分支

```c
  if(cpuid() == 0) {
```

- **区分主从**：`cpuid()` 读取 `tp` 寄存器（在 `start()` 中保存的 `mhartid`）。只有 CPU 0 执行初始化代码，其他 CPU 进入 else 分支等待。

#### 基础 I/O 初始化

```c
    consoleinit();
```

- **控制台初始化**：初始化 UART（串口）硬件，设置波特率（如 115200），启用接收中断。这是为了让 `printf` 能工作。

```c
    printfinit();
```

- **格式化输出初始化**：设置 `printf` 使用的锁（防止多核同时打印混乱），以及指向控制台输出函数的指针。

```c
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
```

- **打印启动信息**：首次使用printf输出，验证控制台工作正常。


#### 内存管理初始化

```c
    kinit();         // physical page allocator
```

- **物理内存分配器**：初始化 `kalloc`。遍历从 `end`（内核结束地址，由链接脚本定义）到 `PHYSTOP`（物理内存上限，通常是 128MB）的所有页面，建立空闲页链表。


```c
    kvminit();       // create kernel page table
```

- **创建内核页表**：建立虚拟内存映射。将高地址（如 `0x0000003ffffff000` 以上）映射到物理地址，包括：

    - UART、PLIC、CLINT 等 MMIO 设备
    - 内核代码和数据
    - 物理内存区域

```c
    kvminithart();   // turn on paging
```

- **启用分页**：将 `kvminit()` 创建的页表基址写入 `satp`（Supervisor Address Translation and Protection）寄存器，并执行 `sfence.vma` 刷新 TLB。从此刻起，CPU 使用虚拟地址。


#### 进程与中断子系统

```c
    procinit();      // process table
```

- **进程表初始化**：为每个 CPU 初始化 `struct proc` 数组，分配内核栈，初始化进程状态锁。


```c
    trapinit();      // trap vectors
```

- **陷阱初始化**：设置全局陷阱处理相关数据结构（如 `kernelvec` 地址）。


```c
    trapinithart();  // install kernel trap vector
```

- **安装陷阱向量**：将 `kernelvec`（汇编编写的陷阱入口）地址写入 `stvec`（Supervisor Trap Vector Base Address）寄存器。当发生异常/中断时，CPU 会跳转到此地址。


#### 设备中断初始化

```c
    plicinit();      // set up interrupt controller
```

- **PLIC 初始化**：平台级中断控制器（Platform-Level Interrupt Controller）全局设置，设置中断优先级和阈值。

```c
    plicinithart();  // ask PLIC for device interrupts
```

- **每个 CPU 的 PLIC 设置**：启用特定设备中断（如磁盘、UART），设置该 CPU 的中断阈值。


#### 文件系统初始化

```c
    binit();         // buffer cache
```

- **缓冲区缓存初始化**：文件系统使用磁盘块缓存，建立 `buf` 结构体和双向链表，用于缓存磁盘数据块。

```c
    iinit();         // inode table
```

- **inode 表初始化**：初始化内存中的 inode 缓存（`icache`），管理已加载的 inode。

```c
    fileinit();      // file table
```

- **文件表初始化**：初始化系统级的打开文件表（`ftable`），管理进程打开的文件描述符。

```c
    virtio_disk_init(); // emulated hard disk
```

- **磁盘设备初始化**：初始化 VirtIO 磁盘设备，设置虚拟队列（virtqueue），启用磁盘中断。


#### 第一个用户进程

```c
    userinit();      // first user process
```

- **创建 init 进程**：从磁盘加载 `/init`（或硬编码的 initcode.S），创建第一个用户态进程。这个进程会 fork 出 shell。

!!! note "注意"

    此时进程尚未运行，只是创建 PCB（进程控制块），等待调度器调度。


#### 同步点


```c
    __sync_synchronize();
```

- **内存屏障（Memory Barrier）**：编译器和 CPU 可能重排序指令。此调用确保前面的**所有内存写入**（初始化完成）对其他 CPU **可见**，不会被重排序到 `started = 1` 之后。

```c
    started = 1;
```

- **通知从核**：将 `started` 置 1。由于 `volatile`，其他 CPU 的缓存会失效，强制从内存读取新值。


### 从核（其他 CPU）等待分支

```c
  } else {
    while(started == 0)
      ;
```

- **自旋等待**：非 0 号 CPU 在此空转，不断检查 `started` 变量。

- **忙等待（Busy-waiting/Spinlock）**：消耗 CPU 周期，但等待时间很短（主核初始化很快完成）。

```c
    __sync_synchronize();
```

- **内存屏障**：确保从核在看到 `started == 1` 后，**后续读取**（如页表访问）不会被重排序到屏障之前。防止读取到未初始化的数据。

```c
    printf("hart %d starting\n", cpuid());
```

- **报告启动**：从核打印启动信息，确认已进入初始化阶段。


#### 从核的必要初始化

```c
    kvminithart();    // turn on paging
```

- **启用分页**：从核使用主核创建的**同一套内核页表**（全局 `kernel_pagetable`），写入自己的 `satp` 寄存器。每个 CPU 独立的 MMU 需要单独启用。


```c
    trapinithart();   // install kernel trap vector
```

- **安装陷阱向量**：每个 CPU 有独立的 `stvec` 寄存器，必须单独设置。

```c
    plicinithart();   // ask PLIC for device interrupts
```

- **PLIC per-CPU 设置**：每个 CPU 需要独立配置中断使能寄存器（`claim/complete` 机制是每个 hart 独立的）。


### 进入调度器

```c
  }

  scheduler();
```

- **进入调度器**：**所有 CPU**（主核和从核）最终都调用 `scheduler()`。

- **永不返回**：`scheduler()` 是一个无限循环，负责选择可运行的进程并切换上下文。内核至此完成启动，进入正常运行状态。


### 执行流程图

```text
    所有 CPU 从 start() 进入 main()
            |
            ▼
        cpuid() == 0?
        /        \
      是          否 (从核)
      /            \
     ▼              ▼
 CPU 0 初始化      while(started==0)
  |-> 控制台           空转等待
  |-> 内存分配器          │
  |-> 页表创建            ▼
  |-> 启用分页      started == 1
  |-> 进程管理            │
  |-> 中断/陷阱           ▼
  |-> 文件系统       启用分页
  |-> 磁盘设备       设置陷阱向量
  |-> 创建 init      设置 PLIC
     │              |
  started = 1       │
     │              │
     |______________|
            |
            ▼
      都进入 scheduler()
      开始进程调度
```

### 关键设计要点

|机制|说明|
|:--|:--|
|**主从架构**|CPU 0 负责全局初始化，避免多核同时初始化导致的竞争条件|
|**自旋等待**|从核使用忙等待而非睡眠，因为调度器尚未运行，无法阻塞|
|**内存屏障**|`__sync_synchronize()` 确保多核间的内存可见性和顺序性|
|**Per-CPU 设置**|页表（`satp`）、陷阱向量（`stvec`）、PLIC 等寄存器是每个 CPU 独立的，必须每个核单独配置|
|**单一入口**|所有 CPU 执行相同代码，通过 `cpuid()` 区分职责，简化启动逻辑|

这段代码展示了**对称多处理（SMP）操作系统启动的经典模式**：一个 CPU 初始化全局资源，其他 CPU 等待后完成自身配置，最终全部进入调度循环。

针对 `main` 中涉及的功能块，后续再进行详细的说明