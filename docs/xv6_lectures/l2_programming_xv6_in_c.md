---
icon: lucide/book-text
---

# 用C语言编写xv6

## 为什么选择使用 C 语言？

- 适合底层编程

    - 便于将C代码映射到RISC-V指令

    - 便于将C类型映射到硬件结构

        例如，设置设备硬件寄存器中的位标志

- 运行时开销小
    - 易于移植到其他硬件平台

    - 可直接访问硬件

- 显式内存管理

    - 无需垃圾回收机制

    - 内核完全控制内存管理

- 高效：编译（无解释器）

  - 编译器将C代码编译为汇编代码

- 广泛用于构建内核、系统软件等

    - 几乎在任何平台上都对C有良好支持

## 为什么不使用C？

- 容易写出错误的代码

- 容易写出存在安全漏洞的代码

## xv6 中 C 语言的应用

- 内存布局
- 指针
- 数组
- 字符串
- 列表
- 位运算符

[非C语言的通用介绍]

## xv6中C程序的内存布局
[参见课本第3.4图]

- text：代码、只读数据
- data：全局 C 变量
- stack：函数的局部变量
- heap：使用sbrk、malloc/free动态分配内存

### 示例：编译 `cat.c`

Makefile定义了编译方式

gcc将源码编译为.o文件

ld将.o文件链接成可执行文件

ulib.o是xv6最小的C库

```bash
riscv64-linux-gnu-objdump -htr user/cat.o
```
    从cat.c编译出的代码，带有重定位记录以插入ulib库

可执行文件采用.a.out格式，包含以下部分：
text（代码）、初始化data、符号表、调试信息等

```bash
riscv64-linux-gnu-objdump -fp user/_cat
```

各节由user.ld定义

### 探索_a.out文件中的_user/_cat

```bash
riscv64-linux-gnu-objdump -S user/_cat
```

与user/cat.asm相同

- 0x0: cat

    如果同时运行两个cat程序会怎样？

    参见 `pgtbl` 课程

- 0xf6: start

    user.ld的默认入口点

- start是什么？

    在ulib.c中定义，调用main()并exit(0)

- data内存在哪里？（例如buf）

    位于data/bss段

    必须由内核设置

但我们知道buf应被分配的地址

```bash
riscv64-linux-gnu-nm -n user/_cat
```

## C指针

指针是一个内存地址

- 每个变量都有一个内存地址（即 p = &i），

- 因此可以通过其指针访问每个变量（即 *i）。

- 指针本身也可以是变量（例如 int *p），

    - 从而拥有自己的内存地址，等等。

指针算术

```bash
char *c;
int *i;
```

`c+1` 和 `i+1` 的值是多少？

结构体元素的引用

```bash
struct {
    int a;
    int b;
} *p;
p->a = 10
```

[示例 ptr.c]

## C 数组

连续内存，存储相同的数据类型（char、int 等）

    无边界检查，不支持动态增长

访问数组的两种方式：

- 通过索引：buf[0]，buf[1]

- 通过指针：*buf，*(buf+1)

[演示 array.c]

## C 字符串

字符数组，以 0 结尾

[演示 str.c]

ulib.c 提供了多个字符串函数

  - strlen() —— 使用数组访问

  - strcmp() —— 使用指针访问

ls.c

    argv：字符串数组

        每个条目都包含一个字符串的地址

        xv6 的 exec 系统将它们压入栈中作为 main 函数的参数
        RISC-V 调用约定：a0=argc, a1=argv [参见 riscv-calling.pdf](https://pdos.csail.mit.edu/6.828/2025/readings/riscv-calling.pdf)

    打印 argv

        [绘制示意图；参见书中的图 3.4]

    T_DIR 代码片段

        mkdir d

        echo hi > d/f

        ls d

        [参见 fs.h 中的 struct dirent]


## C 列表（更多指针）

- 单链表

    - kernel/kalloc.c 实现了一个内存分配器

    - 维护一个自由“页”的列表

        - 一页为 4096 字节

        - free 操作在链表头部插入

        - kalloc 从列表前端获取内存

- 双链表

    - kernel/bio.c 实现了一个 LRU 缓存缓冲区

    - brelse() 需要将某个缓冲区移到链表头部

    - 参见 buf.h

        - 两个指针：prev 和 next

作业题

xv6 是 64 位系统，与 32 位硬件问题略有不同
[demo hw.c]

哪些打印出的值可以保证是该值？

为什么所有打印都能成功？

## 位运算符
char、int、long 和指针在 RISC-V 架构上都有对应的位（分别为 8、32、64 位）。

你可以使用 `|`、`&`、`~`、`^` 来操作这些位。

```c
10001 & 10000 = 10000
10001 | 10000 = 10001
10001 ^ 10000 = 00001
~1000 = 0111
```

示例：

- user/usertests.c

- kernel/fcntl.h

- kernel/sysfile.c

稍后会介绍更有趣的例子

关键词：

`static`：使变量的可见性仅限于其声明所在的文件，但在该文件内是全局的

`void`：无类型、无值、无参数

常见的 C 语言 bug

    - 内存释放后使用

    - 双重释放

    - 未初始化的内存

        - 栈上的内存或 malloc 返回的内存不为零

    - 缓冲区溢出

        - 写入超出数组边界

    - 内存泄漏

    - 类型混淆

        - 错误的类型转换

## 参考资料：
https://blog.regehr.org/archives/1393
https://pdos.csail.mit.edu/6.828/2025/readings/riscv-calling.pdf