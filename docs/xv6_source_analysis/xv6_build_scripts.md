---
icon: lucide/layers
---

## 源码目录

整个 `xv6` 的源码目录结果如下

```bash linenums="1" hl_lines="4 12 20-21"
root@wuxiaoer:~/workdir/os_xv6/xv6-riscv# tree
.
├── LICENSE
├── Makefile
├── README
├── kernel
│   ├── bio.c
│   ├── buf.h
│   ├── console.c
│   ├── defs.h
│   ├── elf.h
│   ├── entry.S
│   ├── exec.c
│   ├── fcntl.h
│   ├── file.c
│   ├── file.h
│   ├── fs.c
│   ├── fs.h
│   ├── kalloc.c
│   ├── kernel.ld
│   ├── kernelvec.S
│   ├── log.c
│   ├── main.c
│   ├── memlayout.h
│   ├── param.h
│   ├── pipe.c
│   ├── plic.c
│   ├── printf.c
│   ├── proc.c
│   ├── proc.h
│   ├── riscv.h
│   ├── sleeplock.c
│   ├── sleeplock.h
│   ├── spinlock.c
│   ├── spinlock.h
│   ├── start.c
│   ├── stat.h
│   ├── string.c
│   ├── swtch.S
│   ├── syscall.c
│   ├── syscall.h
│   ├── sysfile.c
│   ├── sysproc.c
│   ├── trampoline.S
│   ├── trap.c
│   ├── types.h
│   ├── uart.c
│   ├── virtio.h
│   ├── virtio_disk.c
│   ├── vm.c
│   └── vm.h
├── mkfs
│   └── mkfs.c
├── test-xv6.py
└── user
    ├── cat.c
    ├── dorphan.c
    ├── echo.c
    ├── forktest.c
    ├── forphan.c
    ├── grep.c
    ├── grind.c
    ├── init.c
    ├── kill.c
    ├── ln.c
    ├── logstress.c
    ├── ls.c
    ├── mkdir.c
    ├── printf.c
    ├── rm.c
    ├── sh.c
    ├── stressfs.c
    ├── ulib.c
    ├── umalloc.c
    ├── user.h
    ├── user.ld
    ├── usertests.c
    ├── usys.pl
    ├── wc.c
    └── zombie.c

4 directories, 75 files
```

一共4个目录，75的文件, 以下对其中的高亮的关键文件进行学习

## 链接脚本kernel.ld

```c linenums="1" hl_lines="1-2 10 12 23 30 37"
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;

  .text : {
    kernel/entry.o(_entry)
    *(.text .text.*)
    . = ALIGN(0x1000);
    _trampoline = .;
    *(trampsec)
    . = ALIGN(0x1000);
    ASSERT(. - _trampoline == 0x1000, "error: trampoline larger than one page");
    PROVIDE(etext = .);
  }

  .rodata : {
    . = ALIGN(16);
    *(.srodata .srodata.*) /* do not need to distinguish this from .rodata */
    . = ALIGN(16);
    *(.rodata .rodata.*)
  }

  .data : {
    . = ALIGN(16);
    *(.sdata .sdata.*) /* do not need to distinguish this from .data */
    . = ALIGN(16);
    *(.data .data.*)
  }

  .bss : {
    . = ALIGN(16);
    *(.sbss .sbss.*) /* do not need to distinguish this from .bss */
    . = ALIGN(16);
    *(.bss .bss.*)
  }

  PROVIDE(end = .);
}
```

`kernel.ld` 是一个 RISC-V 架构的链接脚本（Linker Script），用于 xv6操作系统的内核链接。

### 头部声明

```c
OUTPUT_ARCH( "riscv" )
```

指定目标架构：告诉链接器生成的可执行文件是为 RISC-V 处理器架构设计的。这会影响链接器对指令格式、对齐要求等的处理 。

```c
ENTRY( _entry )
```

指定程序入口点：定义程序启动时执行的第一条指令的符号名。_entry 通常在 entry.S 汇编文件中定义，是内核启动的入口。链接器会将这个信息写入 ELF 文件的头部，供引导加载器使用 。

### SECTIONS 命令块

```c
SECTIONS
{
```

开始段定义块：所有关于输出文件段（section）的布局和组合规则都定义在这个块内。这是链接脚本的核心部分 。

### 内存地址设置

```c
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;
```

设置当前位置计数器（Location Counter）：`.` 代表当前位置计数器，表示链接器当前在输出文件中的位置

将其设置为 0x80000000（2GB 处），这是 QEMU RISC-V 虚拟机 -kernel 参数加载内核的默认起始地址
注释说明：确保 _entry 位于这个地址，因为 QEMU 会跳转到此处开始执行

### .text 段（代码段）

```c
  .text : {
```

定义 `.text `输出段：开始定义代码段的布局，冒号 : 表示输出段的名称 。


```c
    kernel/entry.o(_entry)
```

放置特定文件的特定符号：将 kernel/entry.o 文件中的 _entry 段（或符号）放在这里。这确保了入口代码位于 .text 段的最开始位置，即地址 0x80000000。


```c
    *(.text .text.*)
```

通配符匹配所有输入文件的代码段：将所有输入文件（* 表示任意文件）的 .text 段和 .text.* 段（如 .text.startup、.text.hot 等）合并到输出文件的 .text 段中 。

```c
    . = ALIGN(0x1000);
```

4KB 对齐：将当前位置对齐到 4KB（0x1000）边界。这是为了页表对齐，因为 RISC-V 通常使用 4KB 页面 。

```c
    _trampoline = .;
```

定义符号：创建全局符号 _trampoline，其值为当前地址。这个符号标记了 trampoline 代码的起始位置，用于用户态和内核态之间的切换（系统调用/中断处理）。

```c
    *(trampsec)
```

放置 trampoline 段：将所有输入文件中的 trampsec 段（这是一个自定义段，专门用于 trampoline 代码）放在这里。trampoline 页需要同时映射到用户和内核空间 。

```c
    . = ALIGN(0x1000);
```

再次 4KB 对齐：确保 trampoline 段结束在页边界上。

```c
    ASSERT(. - _trampoline == 0x1000, "error: trampoline larger than one页");
```

编译时断言：检查 trampoline 段的大小是否正好是 4KB（一页）。如果不是，链接器会报错并停止链接。这确保了 trampoline 代码严格限制在一页内，便于内存管理 。

```c
    PROVIDE(etext = .);
```

条件定义符号：如果代码中没有其他定义 etext，则将其定义为当前地址。etext 通常标记代码段的结束位置，供内核在运行时确定代码段范围（如释放初始化代码）使用 。

```c
  }
```

结束 .text 段定义。

### .rodata 段（只读数据段）

```c
  .rodata : {
    . = ALIGN(16);
```

16 字节对齐：将当前位置对齐到 16 字节边界。RISC-V 架构通常要求数据对齐以获得最佳性能 。

```c
    *(.srodata .srodata.*) /* do not need to distinguish this from .rodata */
```
合并小只读数据段：.srodata 是短只读数据段（small read-only data），通常用于小尺寸常量。注释说明：在操作系统内核中，不需要区分 .srodata 和 .rodata，所以合并在一起 。

```c
    . = ALIGN(16);
    *(.rodata .rodata.*)
  }
```

再次对齐后放置普通只读数据：放置所有 .rodata 段（字符串常量、只读数组等）。


### .data 段（已初始化数据段）

```c
  .data : {
    . = ALIGN(16);
    *(.sdata .sdata.*) /* do not need to distinguish this from .data */
```

小数据段（small data）：.sdata 用于小尺寸全局变量，某些架构可以通过短偏移量高效访问。同样，这里选择合并到 .data 。

```c
    . = ALIGN(16);
    *(.data .data.*)
  }
```

放置普通已初始化数据：全局变量、静态变量等。

### .bss 段（未初始化数据段）

```c
  .bss : {
    . = ALIGN(16);
    *(.sbss .sbss.*) /* do not need to distinguish this from .bss */
```

小未初始化数据段：.sbss 是小尺寸未初始化数据段，同样选择合并 。

```c
    . = ALIGN(16);
    *(.bss .bss.*)
  }
```

放置普通未初始化数据：未初始化的全局变量和静态变量。BSS 段在 ELF 文件中不占用实际空间，只记录大小，由加载器或启动代码清零 。

### 结束符号

```c
  PROVIDE(end = .);
```

定义内核结束符号：标记整个内核镜像的结束位置。这个符号对内核内存管理至关重要：
用于确定内核占用的物理内存范围
作为 kalloc（内核内存分配器）的起始地址，从此处开始管理空闲内存

```c
}
```

结束 SECTIONS 块。

### 内存布局总结

| 地址范围    | 内容          | 说明                      |
|:----------:|---------------|---------------------------|
| 0x80000000 | _entry / .text 开始 | 内核入口，QEMU 跳转地址 |
| ...        | 内核代码       | entry.o + 其他代码        |
| ...        | Trampoline 页 | 4KB 固定大小，用于系统调用 |
| ...        | .rodata       | 只读常量数据              |
| ...        | .data         | 已初始化全局变量          |
| ...        | .bss          | 未初始化全局变量          |
| end        | 内核镜像结束   | 动态内存管理起始点        |

这个链接脚本精心设计，确保了 xv6 内核在 RISC-V/QEMU 环境下正确加载和运行，特别是入口点地址、trampoline 页对齐、以及内存段的合理布局。


!!! 链接脚本不同段用途

    .text → 代码

    .rodata → 只读数据

    .data → 已初始化变量

    .bss → 未初始化变量

## 编译脚本Makefile

Makefile 文件内容如下

```makefile
K=kernel
U=user

OBJS = \
  $K/entry.o \
  $K/start.o \
  $K/console.o \
  $K/printf.o \
  $K/uart.o \
  $K/kalloc.o \
  $K/spinlock.o \
  $K/string.o \
  $K/main.o \
  $K/vm.o \
  $K/proc.o \
  $K/swtch.o \
  $K/trampoline.o \
  $K/trap.o \
  $K/syscall.o \
  $K/sysproc.o \
  $K/bio.o \
  $K/fs.o \
  $K/log.o \
  $K/sleeplock.o \
  $K/file.o \
  $K/pipe.o \
  $K/exec.o \
  $K/sysfile.o \
  $K/kernelvec.o \
  $K/plic.o \
  $K/virtio_disk.o

# riscv64-unknown-elf- or riscv64-linux-gnu-
# perhaps in /opt/riscv/bin
#TOOLPREFIX =

# Try to infer the correct TOOLPREFIX if not set
ifndef TOOLPREFIX
TOOLPREFIX := $(shell if riscv64-unknown-elf-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-unknown-elf-'; \
	elif riscv64-elf-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-elf-'; \
	elif riscv64-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-linux-gnu-'; \
	elif riscv64-unknown-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-unknown-linux-gnu-'; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find a riscv64 version of GCC/binutils." 1>&2; \
	echo "*** To turn off this error, run 'gmake TOOLPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

QEMU = qemu-system-riscv64
MIN_QEMU_VERSION = 7.2

CC = $(TOOLPREFIX)gcc
AS = $(TOOLPREFIX)gas
LD = $(TOOLPREFIX)ld
OBJCOPY = $(TOOLPREFIX)objcopy
OBJDUMP = $(TOOLPREFIX)objdump

CFLAGS = -Wall -Werror -Wno-unknown-attributes -O -fno-omit-frame-pointer -ggdb -gdwarf-2
CFLAGS += -MD
CFLAGS += -mcmodel=medany
CFLAGS += -ffreestanding
CFLAGS += -fno-common -nostdlib
CFLAGS += -fno-builtin-strncpy -fno-builtin-strncmp -fno-builtin-strlen -fno-builtin-memset
CFLAGS += -fno-builtin-memmove -fno-builtin-memcmp -fno-builtin-log -fno-builtin-bzero
CFLAGS += -fno-builtin-strchr -fno-builtin-exit -fno-builtin-malloc -fno-builtin-putc
CFLAGS += -fno-builtin-free
CFLAGS += -fno-builtin-memcpy -Wno-main
CFLAGS += -fno-builtin-printf -fno-builtin-fprintf -fno-builtin-vprintf
CFLAGS += -I.
CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)

# Disable PIE when possible (for Ubuntu 16.10 toolchain)
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]no-pie'),)
CFLAGS += -fno-pie -no-pie
endif
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]nopie'),)
CFLAGS += -fno-pie -nopie
endif

LDFLAGS = -z max-page-size=4096

$K/kernel: $(OBJS) $K/kernel.ld
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS)
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym

$K/%.o: $K/%.S
	$(CC) -g -c -o $@ $<

tags: $(OBJS)
	etags kernel/*.S kernel/*.c

ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o

_%: %.o $(ULIB) $U/user.ld
	$(LD) $(LDFLAGS) -T $U/user.ld -o $@ $< $(ULIB)
	$(OBJDUMP) -S $@ > $*.asm
	$(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym

$U/usys.S : $U/usys.pl
	perl $U/usys.pl > $U/usys.S

$U/usys.o : $U/usys.S
	$(CC) $(CFLAGS) -c -o $U/usys.o $U/usys.S

$U/_forktest: $U/forktest.o $(ULIB)
	# forktest has less library code linked in - needs to be small
	# in order to be able to max out the proc table.
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_forktest $U/forktest.o $U/ulib.o $U/usys.o
	$(OBJDUMP) -S $U/_forktest > $U/forktest.asm

mkfs/mkfs: mkfs/mkfs.c $K/fs.h $K/param.h
	gcc -Wno-unknown-attributes -I. -o mkfs/mkfs mkfs/mkfs.c

# Prevent deletion of intermediate files, e.g. cat.o, after first build, so
# that disk image changes after first build are persistent until clean.  More
# details:
# http://www.gnu.org/software/make/manual/html_node/Chained-Rules.html
.PRECIOUS: %.o

UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_logstress\
	$U/_forphan\
	$U/_dorphan\

fs.img: mkfs/mkfs README $(UPROGS)
	mkfs/mkfs fs.img README $(UPROGS)

-include kernel/*.d user/*.d

clean:
	rm -f *.tex *.dvi *.idx *.aux *.log *.ind *.ilg \
	*/*.o */*.d */*.asm */*.sym \
	$K/kernel fs.img \
	mkfs/mkfs .gdbinit \
        $U/usys.S \
	$(UPROGS)

# try to generate a unique GDB port
GDBPORT = $(shell expr `id -u` % 5000 + 25000)
# QEMU's gdb stub command line changed in 0.11
QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
	then echo "-gdb tcp::$(GDBPORT)"; \
	else echo "-s -p $(GDBPORT)"; fi)
ifndef CPUS
CPUS := 3
endif

QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

qemu: check-qemu-version $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)

.gdbinit: .gdbinit.tmpl-riscv
	sed "s/:1234/:$(GDBPORT)/" < $^ > $@

qemu-gdb: $K/kernel .gdbinit fs.img
	@echo "*** Now run 'gdb' in another window." 1>&2
	$(QEMU) $(QEMUOPTS) -S $(QEMUGDB)

print-gdbport:
	@echo $(GDBPORT)

QEMU_VERSION := $(shell $(QEMU) --version | head -n 1 | sed -E 's/^QEMU emulator version ([0-9]+\.[0-9]+)\..*/\1/')
check-qemu-version:
	@if [ "$(shell echo "$(QEMU_VERSION) >= $(MIN_QEMU_VERSION)" | bc)" -eq 0 ]; then \
		echo "ERROR: Need qemu version >= $(MIN_QEMU_VERSION)"; \
		exit 1; \
	fi
```

以下对其中每一行的含义进行详细说明

### 路径变量定义

```makefile
K=kernel
U=user
```

- **定义路径前缀变量**：

`K` 代表内核源码目录 `kernel/`，`U` 代表用户程序目录 `user/`。使用变量便于后续引用和修改路径结构。

### 内核对象文件列表

```makefile
OBJS = \
  $K/entry.o \
  $K/start.o \
  ...（所有内核 .o 文件）
```

- **定义内核对象文件列表**：

`OBJS` 变量包含构建内核所需的全部目标文件。使用 `\` 进行续行，保持可读性。这些是内核的各个模块，包括：

- `entry.o`：内核入口（汇编）
- `start.o`：早期启动代码
- `console.o`, `uart.o`：控制台和串口驱动
- `kalloc.o`：内核内存分配器
- `vm.o`：虚拟内存管理
- `proc.o`：进程管理
- `swtch.o`：上下文切换（汇编）
- `trampoline.o`：用户/内核态切换跳板
- `trap.o`：中断和异常处理
- `syscall.o`, `sysproc.o`, `sysfile.o`：系统调用处理
- `fs.o`, `bio.o`, `log.o`：文件系统
- `virtio_disk.o`：磁盘驱动（VirtIO）

### 工具链检测与配置

```makefile
# riscv64-unknown-elf- or riscv64-linux-gnu-
#TOOLPREFIX =
```

- **注释说明支持的交叉编译工具链前缀**：

可以是 `riscv64-unknown-elf-`（裸机 ELF）或 `riscv64-linux-gnu-`（Linux 工具链）。当前被注释掉，表示由自动检测逻辑决定。

```makefile
ifndef TOOLPREFIX
TOOLPREFIX := $(shell if riscv64-unknown-elf-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-unknown-elf-'; \
	elif riscv64-elf-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-elf-'; \
	elif riscv64-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-linux-gnu-'; \
	elif riscv64-unknown-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
	then echo 'riscv64-unknown-linux-gnu-'; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find a riscv64 version of GCC/binutils." 1>&2; \
	echo "*** To turn off this error, run 'gmake TOOLPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif
```

- **自动检测 RISC-V 工具链**：

如果 `TOOLPREFIX` 未定义，执行 shell 脚本检测：

1. 依次尝试 `riscv64-unknown-elf-`、`riscv64-elf-`、`riscv64-linux-gnu-`、`riscv64-unknown-linux-gnu-`

2. 使用 `objdump -i` 检查是否支持 `elf64-big` 格式（RISC-V 64 位大端，虽然实际可能是小端，但这是检测 ELF64 支持的方法）

3. 如果都找不到，输出错误信息并退出

4. 注释提到可以使用 `gmake TOOLPREFIX=` 来禁用此错误（使用系统默认工具）


### 工具定义

```makefile
QEMU = qemu-system-riscv64
MIN_QEMU_VERSION = 7.2
```

- **QEMU 模拟器路径**：指定使用 `qemu-system-riscv64`（RISC-V 64 位系统模拟器）。
- **最低版本要求**：要求 QEMU 版本 >= 7.2，以确保支持所需的 RISC-V 特性。

```makefile
CC = $(TOOLPREFIX)gcc
AS = $(TOOLPREFIX)gas
LD = $(TOOLPREFIX)ld
OBJCOPY = $(TOOLPREFIX)objcopy
OBJDUMP = $(TOOLPREFIX)objdump
```

- **定义交叉编译工具**：根据检测到的前缀，定义 GCC 编译器、GNU 汇编器、链接器、对象复制工具（用于格式转换）和反汇编器。


### 编译器标志 (CFLAGS)

```makefile
CFLAGS = -Wall -Werror -Wno-unknown-attributes -O -fno-omit-frame-pointer -ggdb -gdwarf-2
```

- **基础编译选项**：
    - `-Wall`：开启所有警告
    - `-Werror`：将警告视为错误（强制代码干净）
    - `-Wno-unknown-attributes`：忽略未知属性警告（GCC 版本兼容性）
    - `-O`：优化级别 1（简单优化，保持调试友好）
    - `-fno-omit-frame-pointer`：保留帧指针（便于调试栈回溯）
    - `-ggdb -gdwarf-2`：生成 GDB 调试信息，使用 DWARF 2 格式

```makefile
CFLAGS += -MD
```

- **自动生成依赖**：`-MD` 让编译器生成 `.d` 依赖文件（记录头文件依赖关系，用于增量构建）。

```makefile
CFLAGS += -mcmodel=medany
```

- **代码模型**：RISC-V 的 medium-any 代码模型，允许代码和数据分布在任意位置（适合内核，因为内核加载地址较高，如 0x80000000）。

```makefile
CFLAGS += -ffreestanding
```

- **独立环境**：表示编译的目标不是在操作系统上运行的程序，而是裸机/内核代码（不使用标准库启动代码）。

```makefile
CFLAGS += -fno-common -nostdlib
```

- **禁用 COMMON 段**：未初始化的全局变量放入 BSS 而非 COMMON，确保链接行为明确。
- **不链接标准库**：内核不使用 libc。

```makefile
CFLAGS += -fno-builtin-strncpy -fno-builtin-strncmp -fno-builtin-strlen -fno-builtin-memset
CFLAGS += -fno-builtin-memmove -fno-builtin-memcmp -fno-builtin-log -fno-builtin-bzero
CFLAGS += -fno-builtin-strchr -fno-builtin-exit -fno-builtin-malloc -fno-builtin-putc
CFLAGS += -fno-builtin-free
CFLAGS += -fno-builtin-memcpy -Wno-main
CFLAGS += -fno-builtin-printf -fno-builtin-fprintf -fno-builtin-vprintf
```

- **禁用内置函数**：GCC 通常会将标准 C 函数（如 `memcpy`、`printf`）优化为内置指令或库调用。这里全部禁用，因为内核要自己实现这些函数，避免与 GCC 内置实现冲突。

- `-Wno-main`：不警告 `main` 函数签名问题（内核入口不是标准 main）。

```makefile
CFLAGS += -I.
```

- **头文件搜索路径**：添加当前目录到包含路径（用于找 `kernel/`、`user/` 下的头文件）。

```makefile
CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
```

- **检测并禁用栈保护器**：测试编译器是否支持 `-fno-stack-protector`（禁用栈溢出保护），如果支持则添加。栈保护器需要运行时支持（如 `__stack_chk_fail`），裸机环境无法提供。


### 位置无关代码 (PIE) 处理

```makefile
# Disable PIE when possible (for Ubuntu 16.10 toolchain)
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]no-pie'),)
CFLAGS += -fno-pie -no-pie
endif
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]nopie'),)
CFLAGS += -fno-pie -nopie
endif
```

- **禁用位置无关可执行文件 (PIE)**：Ubuntu 16.10+ 等系统默认启用 PIE，但内核需要固定在特定地址运行（如 0x80000000）。

- 检测编译器 specs 中是否包含 `no-pie` 选项，如果包含则添加 `-fno-pie`（编译时不生成位置无关代码）和 `-no-pie`/`-nopie`（链接时不生成 PIE）。


### 链接器标志

```makefile
LDFLAGS = -z max-page-size=4096
```

- **最大页大小**：告诉链接器内存页大小为 4KB。这影响段对齐，确保段在页边界对齐（与 RISC-V Sv39 页表兼容）。


### 内核构建规则

```makefile
$K/kernel: $(OBJS) $K/kernel.ld
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS)
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym
```

- **内核链接规则**：
    - 依赖：所有对象文件 `$(OBJS)` 和链接脚本 `$K/kernel.ld`
    - 使用 `ld` 链接，指定链接脚本 `-T` 控制内存布局
    - 生成反汇编文件 `kernel.asm`（`-S` 混合源代码和汇编）
    - 生成符号表 `kernel.sym`：过滤掉表头，只保留符号名和地址（用于调试）

### 汇编文件编译规则

```makefile
$K/%.o: $K/%.S
	$(CC) -g -c -o $@ $<
```

- **汇编文件编译**：将 `.S`（大写，表示需要预处理的汇编）编译为 `.o`。

- 使用 `$(CC)` 而非 `$(AS)`，因为 `.S` 文件需要 C 预处理器处理（处理 `#include`、`#define` 等）。

- `-g` 生成调试信息，`$@` 是目标，`$<` 是第一个依赖（源文件）。


### 标签生成

```makefile
tags: $(OBJS)
	etags kernel/*.S kernel/*.c
```

- **生成 Emacs TAGS 文件**：用于在 Emacs 中快速跳转函数定义。`etags` 扫描所有汇编和 C 源文件。

### 用户库定义

```makefile
ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o
```

- **用户态库文件**：所有用户程序链接时需要的库：
    - `ulib.o`：用户态工具函数（如 `strcpy`、`memmove`）
    - `usys.o`：系统调用桩（由 `usys.S` 生成）
    - `printf.o`：用户态 printf 实现
    - `umalloc.o`：用户态内存分配（`malloc`/`free`）


### 用户程序构建规则

```makefile
_%: %.o $(ULIB) $U/user.ld
	$(LD) $(LDFLAGS) -T $U/user.ld -o $@ $< $(ULIB)
	$(OBJDUMP) -S $@ > $*.asm
	$(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

- **通用用户程序构建**：`_%` 模式规则，构建如 `_cat`、`_echo` 等用户程序。
- 依赖：对应的 `.o` 文件（如 `cat.o`）、用户库、用户链接脚本。
- 使用用户链接脚本 `$U/user.ld`（通常将程序加载到用户态地址空间，如 0x1000）。
- 同样生成反汇编和符号表。

### 系统调用桩生成

```makefile
$U/usys.S : $U/usys.pl
	perl $U/usys.pl > $U/usys.S
```

- **生成系统调用汇编**：`usys.pl` 是 Perl 脚本，读取系统调用列表，生成 `usys.S`（包含每个系统调用的汇编包装函数，如 `fork`、`exit`、`write` 等，通过 `ecall` 指令触发特权态切换）。


```makefile
$U/usys.o : $U/usys.S
	$(CC) $(CFLAGS) -c -o $U/usys.o $U/usys.S
```

- **编译系统调用桩**：使用 CFLAGS（包含 `-mcmodel=medany` 等）编译汇编文件。

### 特殊程序 forktest

```makefile
$U/_forktest: $U/forktest.o $(ULIB)
	# forktest has less library code linked in - needs to be small
	# in order to be able to max out the proc table.
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_forktest $U/forktest.o $U/ulib.o $U/usys.o
	$(OBJDUMP) -S $U/_forktest > $U/forktest.asm
```

- **特殊的 forktest 构建**：`forktest` 是一个压力测试程序，需要创建尽可能多的进程。

- 为了节省内存（最大化进程数量），**不链接完整的 ULIB**，只链接 `ulib.o` 和 `usys.o`（不使用 `printf.o` 和 `umalloc.o`）。

- `-N`：将 `.data` 段标记为可读可写（不设置为纯只读）。

- `-e main`：显式指定入口点为 `main`（而不是默认的 `_start`）。

- `-Ttext 0`：将代码段加载地址设为 0（用户态虚拟地址起始）。


### 文件系统工具

```makefile
mkfs/mkfs: mkfs/mkfs.c $K/fs.h $K/param.h
	gcc -Wno-unknown-attributes -I. -o mkfs/mkfs mkfs/mkfs.c
```

- **构建 mkfs 工具**：在**宿主机**（非交叉编译）上运行的工具，用于创建文件系统镜像 `fs.img`。
- 使用宿主机的 `gcc`（不是 `$(CC)`），因为要在构建机器上运行。
- 包含 `fs.h`（文件系统结构定义）和 `param.h`（文件系统参数）。


### 保留中间文件


```makefile
# Prevent deletion of intermediate files, e.g. cat.o, after first build, so
# that disk image changes after first build are persistent until clean.  More
# details:
# http://www.gnu.org/software/make/manual/html_node/Chained-Rules.html
.PRECIOUS: %.o
```

- **保留中间对象文件**：通常 Make 会自动删除作为"中间文件"的 `.o` 文件（如果只被间接需要）。`.PRECIOUS` 阻止这种删除，确保 `cat.o` 等文件保留，因为 `mkfs` 需要它们来构建文件系统（将用户程序放入磁盘镜像）。


### 用户程序列表

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_logstress\
	$U/_forphan\
	$U/_dorphan\
```

- **用户程序清单**：定义要构建的所有用户态程序，包括基础工具（`cat`、`echo`、`ls`）、测试程序（`forktest`、`usertests`）和特殊测试（`logstress`、`forphan` 等）。


### 文件系统镜像构建

```makefile
fs.img: mkfs/mkfs README $(UPROGS)
	mkfs/mkfs fs.img README $(UPROGS)
```

- **构建磁盘镜像**：`fs.img` 是虚拟磁盘文件。

- 依赖：`mkfs` 工具、`README` 文件、所有用户程序。

- 运行 `mkfs/mkfs`，将 `README` 和所有用户程序（`_$prog`）写入 `fs.img`，作为 xv6 启动后的根文件系统内容。


### 自动依赖包含


```makefile
-include kernel/*.d user/*.d
```

- **包含自动生成的依赖文件**：`.d` 文件（由 `gcc -MD` 生成）记录每个 `.o` 文件依赖哪些头文件（`.h`）。这样修改头文件时，Make 知道需要重新编译哪些源文件。`-` 前缀表示如果文件不存在不报错（首次构建时）。


### 清理规则

```makefile
clean:
	rm -f *.tex *.dvi *.idx *.aux *.log *.ind *.ilg \
	*/*.o */*.d */*.asm */*.sym \
	$K/kernel fs.img \
	mkfs/mkfs .gdbinit \
        $U/usys.S \
	$(UPROGS)
```

- **清理构建产物**：删除所有生成的文件，包括：

    - LaTeX 相关（xv6 文档编译残留）

    - 所有 `.o`、`.d`、`.asm`、`.sym` 文件

    - 内核、文件系统镜像、mkfs 工具

    - GDB 配置文件、自动生成的 `usys.S`、所有用户程序


### QEMU 调试配置

```makefile
# try to generate a unique GDB port
GDBPORT = $(shell expr `id -u` % 5000 + 25000)
```

- **生成唯一 GDB 端口**：基于用户 ID (`id -u`) 计算端口号（25000-30000 范围内），避免多用户同时调试时端口冲突。


```makefile
# QEMU's gdb stub command line changed in 0.11
QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
	then echo "-gdb tcp::$(GDBPORT)"; \
	else echo "-s -p $(GDBPORT)"; fi)
```

- **检测 QEMU GDB 选项格式**：旧版 QEMU（<0.11）使用 `-s -p`，新版使用 `-gdb tcp::port`。通过检测 `-help` 输出确定使用哪种格式。


```makefile
ifndef CPUS
CPUS := 3
endif
```

- **默认 CPU 数量**：如果未指定，使用 3 个 CPU 核心（支持 SMP 多核调度测试）。


### QEMU 运行选项

```makefile
QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
```

- **基础 QEMU 选项**：

    - `-machine virt`：使用 RISC-V VirtIO 机器类型（QEMU 的标准 RISC-V 虚拟机）

    - `-bios none`：不使用 BIOS（xv6 内核直接启动）

    - `-kernel $K/kernel`：指定内核文件

    - `-m 128M`：分配 128MB 内存

    - `-smp $(CPUS)`：多处理器支持（3 核）

    - `-nographic`：无图形界面，使用串口控制台

```makefile
QEMUOPTS += -global virtio-mmio.force-legacy=false
```

- **VirtIO 现代模式**：禁用 legacy 模式，使用现代 VirtIO MMIO 设备接口。

```makefile
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

- **磁盘设备配置**：

    1. 定义驱动器：使用 `fs.img` 作为原始格式磁盘，命名为 `x0`

    2. 添加 VirtIO 块设备：将驱动器挂载到 VirtIO MMIO 总线，xv6 通过 MMIO 访问磁盘


### QEMU 运行目标

```makefile
qemu: check-qemu-version $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)
```

- **运行 QEMU**：先检查版本，确保内核和文件系统镜像已构建，然后启动 QEMU。


### GDB 配置生成

```makefile
.gdbinit: .gdbinit.tmpl-riscv
	sed "s/:1234/:$(GDBPORT)/" < $^ > $@
```

- **生成 GDB 配置文件**：从模板 `.gdbinit.tmpl-riscv` 复制，将默认端口 `1234` 替换为计算出的 `$(GDBPORT)`。


### GDB 调试运行

```makefile
qemu-gdb: $K/kernel .gdbinit fs.img
	@echo "*** Now run 'gdb' in another window." 1>&2
	$(QEMU) $(QEMUOPTS) -S $(QEMUGDB)
```

- **启动带调试的 QEMU**：

    - `-S`：启动时暂停 CPU（等待 GDB 连接）

    - `$(QEMUGDB)`：启用 GDB stub，监听指定端口

    - 提示用户在另一个终端运行 `gdb` 连接


```makefile
print-gdbport:
	@echo $(GDBPORT)
```

- **打印 GDB 端口**：方便用户知道当前使用的端口号。


### QEMU 版本检查

```makefile
QEMU_VERSION := $(shell $(QEMU) --version | head -n 1 | sed -E 's/^QEMU emulator version ([0-9]+\.[0-9]+)\..*/\1/')
check-qemu-version:
	@if [ "$(shell echo "$(QEMU_VERSION) >= $(MIN_QEMU_VERSION)" | bc)" -eq 0 ]; then \
		echo "ERROR: Need qemu version >= $(MIN_QEMU_VERSION)"; \
		exit 1; \
	fi
```

- **检查 QEMU 版本**：

    - 提取 QEMU 版本号（如 `7.2`）
    - 使用 `bc` 计算器比较版本
    - 如果低于 `7.2`，报错退出（旧版本可能缺少必要的 RISC-V 支持）