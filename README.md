# 嵌入式面试知识点笔记

按主题整理面试经验，涵盖程序内存布局、I2C、C/C++、FreeRTOS、Linux 进程与线程，以及网络、调试和编程题。建议先掌握**六大内存分区**，再用 `static`、`malloc`、任务栈等章节把概念串起来；同一知识点的新问题继续补充到对应小节。

## 目录

- [六大内存分区](#memory-layout)
  - [一段代码看变量位置](#memory-example) · [各区域的作用](#memory-regions) · [MCU 启动时发生什么](#memory-startup) · [常见追问](#memory-questions)
- [Cache 与 DMA](#cache-dma)
  - [Cache 基础](#cache-basics) · [DMA 与缓存一致性](#dma-coherency)
- [I2C 总线与显示屏通信](#i2c)
  - [基础时序](#i2c-signals) · [7 位地址与设备数量](#i2c-address-count) · [显示屏写入与寄存器读取](#i2c-examples)
- [C/C++ 基础](#c-basics)
  - [`volatile`](#volatile) · [`static`](#static) · [结构体内存对齐](#struct-alignment) · [`malloc` 与 `free`](#malloc-free)
- [FreeRTOS](#freertos)
  - [任务调度](#freertos-scheduling) · [高优先级任务与饥饿](#freertos-starvation) · [任务间通信](#freertos-communication) · [创建任务](#freertos-task-creation) · [检查任务栈](#freertos-stack-check) · [FreeRTOS 与 Linux 的栈](#freertos-linux-stack)
- [Linux 进程与线程](#linux)
  - [新线程的默认栈大小](#linux-thread-stack-size) · [创建进程](#linux-process-creation) · [创建线程](#linux-thread-creation) · [多线程与多进程](#threads-vs-processes)
- [TCP 服务端建立连接](#tcp-server-connection)
- [嵌入式调试接口排障](#debug-interface)
- [编程题：只用 switch case 判断分数](#switch-score)

<a id="memory-layout"></a>
## 六大内存分区
<!-- TOPIC:memory-layout:START -->

**面试问题：程序的 code、rodata、data、bss、heap、stack 分别存放什么？**先记住一条主线：**代码和只读常量通常随程序映像保存；有固定生命周期的可写数据在启动时准备好；运行中临时需要的空间由动态分配器或函数调用管理。**这里说的“六大分区”是常见教学模型，不表示物理内存一定被等分或严格按这个顺序排列；实际布局取决于编译器、链接脚本和平台。

<a id="memory-example"></a>
### 一段代码看变量位置

```c
#include <stdio.h>
#include <stdlib.h>

int global_level = 3;              // 通常在 .data
int global_error;                  // 通常在 .bss
const char screen_name[] = "OLED"; // 通常在 .rodata

void draw_page(void) {             // 函数指令通常在 .text
    static int frames;             // 通常在 .bss，不因函数返回而消失
    int page = 1;                  // 自动局部变量：可能用栈或寄存器
    unsigned char *pixels = malloc(128);
    // pixels 是局部指针；它指向的 128 字节来自动态分配器

    if (pixels == NULL) return;
    pixels[0] = (unsigned char)(global_level + page + frames);
    printf("%s: %u, error=%d\n", screen_name,
           (unsigned)pixels[0], global_error);
    ++frames;
    free(pixels);
}
```

“通常”很重要：C 语言规定对象的行为和生命周期，具体放进哪个节是实现选择；没有被实际使用的对象甚至可能被优化掉。

<a id="memory-regions"></a>
### 六个区域各管什么？

| 区域 | 典型内容 | 生命周期与例子 | 记忆重点 |
| --- | --- | --- | --- |
| **代码区 `.text`** | 函数编译后的机器指令 | 程序运行时可执行；例：`draw_page()` 的指令 | 函数的代码在这里，函数里的变量不因此属于 `.text`。 |
| **只读数据区 `.rodata`** | 常量数据、字符串字面量 | 通常随程序映像存在；例：`screen_name` 的内容 | `const` 不保证对象一定落在 `.rodata`；局部常量也可能被优化为立即数。 |
| **已初始化数据区 `.data`** | 有非零初值、运行时可写的静态存储期对象 | 整个程序运行期间；例：`global_level = 3` | MCU 常在 Flash 保存初值，启动时复制到 RAM。 |
| **零初始化区 `.bss`** | 未显式初始化或初始化为零的静态存储期对象 | 整个程序运行期间；例：`global_error`、`frames` | 程序启动时清零，通常不必在固件映像里逐字节存零。 |
| **堆／动态分配区** | 运行时申请的内存 | 成功申请后至释放前；例：`malloc(128)` 返回的缓冲区 | 申请失败要检查；`free` 后不能继续使用原内存。 |
| **栈** | 调用现场以及通常的局部自动变量 | 随函数调用变化；例：`page`、局部指针 `pixels` | 局部变量可能被放在寄存器；每个 RTOS 任务／Linux 线程有自己的栈。 |

**别把“变量写在函数里”和“变量在栈上”画等号。**`frames` 是函数内的 `static` 变量，却具有静态存储期，函数返回后仍保留值；未初始化时通常在 `.bss`。`pixels` 是局部指针，指针本身通常在栈或寄存器里，所指向的 128 字节才来自动态分配器。Linux 的 `malloc` 实现也可能通过其他内存映射取得空间，不能把“堆”理解为所有动态分配必定落在一块连续区域。[Linux `malloc(3)`](https://man7.org/linux/man-pages/man3/malloc.3.html)

<a id="memory-startup"></a>
### MCU 启动时发生什么？

```text
Flash / ROM                           RAM
┌──────────────────────┐             ┌─────────────────────────┐
│ .text：机器指令       │              │ .data：可写的已初始化数据  │
│ .rodata：只读数据     │              │ .bss：清零后的数据        │
│ .data 的初始值 ───────┼──复制──────→ │                         │
└──────────────────────┘             │ 动态分配区、任务栈等       │
                                     └─────────────────────────┘
```

在常见的 Flash 运行型 MCU 程序中，启动代码先把 `.data` 的初值复制到运行时的 RAM 地址，再把 `.bss` 清零，然后才进入 `main()`。因此 `global_level` 一开始是 `3`，`global_error` 一开始是 `0`。`.data` 的初值通常同时占用固件映像空间和运行时 RAM；`.bss` 主要占用运行时 RAM。具体地址要看链接脚本或 map 文件。[GNU 链接器的 ROM/RAM 示例](https://sourceware.org/binutils/docs/ld.html)

<a id="memory-questions"></a>
### 面试常见追问

1. **`int x = 0` 一定在 `.data` 吗？**若它是全局变量或静态变量，通常可放在 `.bss`；若它是普通局部变量，则具有自动存储期，不能只凭初值判断为 `.data`。
2. **`static int n` 写在函数里，为什么不是栈变量？**`static` 让它在整个程序运行期间存在；它只初始化一次、跨调用保留值。参见下方 [`static` 小节](#static)。
3. **`malloc` 得到的内存和指针变量各在哪里？**分配得到的缓冲区属于动态分配；局部指针本身通常在栈或寄存器。参见 [`malloc` 与 `free`](#malloc-free)。
4. **栈满或堆不够会怎样？**动态分配失败通常以 `NULL` 表示；栈溢出的表现依平台及保护机制而异，在 MCU 上可能破坏其他内存。FreeRTOS 可通过[任务栈高水位](#freertos-stack-check)估算历史最小余量。
5. **这六块在 Linux 中也按图摆放吗？**不一定。Linux 进程使用虚拟地址空间，还有共享库、内存映射等区域；可查看 `/proc/<pid>/maps` 观察映射，不能把上面的 MCU 示意图当成通用地址表。[Linux `proc_pid_maps(5)`](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html)

**一句话复述：**`.text` 放指令，`.rodata` 放通常只读的数据，`.data` 放有初值的可写静态数据，`.bss` 放启动时清零的静态数据，动态分配区按需申请，栈跟随函数调用和任务／线程运行。
<!-- TOPIC:memory-layout:END -->

<a id="cache-dma"></a>
## Cache 与 DMA
<!-- TOPIC:cache-dma:START -->

**面试问题：Cache 是什么？它与 RAM、DMA 和 `volatile` 有什么关系？**

<a id="cache-basics"></a>
### Cache 基础

**Cache（高速缓存）是 CPU 附近保存数据或指令副本的小容量高速存储**，目的是减少访问较慢内存的等待。CPU 要读某地址时，若其内容已经在 Cache 中，就是**命中**；否则是**未命中**，需要从更远的内存取入。讨论 DMA 时，主要关心数据 Cache（D-Cache）。Cache 不是 `.text`、`.data`、堆、栈之外的“第七个程序分区”：这些名称描述程序内容及其生命周期，Cache 则是硬件对其中部分内容保存的临时副本。[Arm 对 Cache 与一致性的介绍](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/exploring-how-cache-coherency-accelerates-heterogeneous-compute)

```text
CPU 寄存器：当前参与计算的少量值
      ↕
CPU 数据 Cache：近期访问的数据副本
      ↕
RAM：程序运行时存放的大量数据
```

**Cache 按“缓存行”管理数据，而非只处理一个字节。**例如某平台一行是 32 字节，读取一个字节时可能把它所在的整行取进 Cache；32 字节只是示例，实际大小看芯片手册。后续对 DMA 缓冲区执行缓存维护时，要考虑行对齐，以及缓冲区是否与其他变量共享同一行，否则可能影响邻近数据。[Linux DMA 指南](https://docs.kernel.org/core-api/dma-api-howto.html)

<a id="dma-coherency"></a>
### DMA 与缓存一致性

DMA 让外设与内存交换数据时无需 CPU 逐字节搬运；CPU 通常负责配置传输、处理完成事件等工作。在使用**回写式（write-back）**数据 Cache 的平台上，CPU 修改缓冲区后，新值可能暂时只在 Cache，RAM 仍是旧值。这一行称为**脏行**。`clean` 将脏数据写回到 DMA 等设备可见的位置；`invalidate` 则使旧副本失效，让 CPU 下次重新取数据。直接丢弃仍含有未写回修改的脏行可能丢数据，因此操作顺序必须遵循芯片或驱动文档。下表只讨论 **CPU Cache 与 DMA 不自动保持一致** 的平台。[Arm 缓存维护说明](https://documentation-service.arm.com/static/684be32a3f793d5d7b223563)

| 传输方向 | 可能发生的旧数据问题 | 交接缓冲区时的典型处理 |
| --- | --- | --- |
| **CPU 写，DMA 读**，如 CPU 准备显示数据后由 I2C DMA 发送 | 新数据还在 CPU Cache，DMA 从 RAM 读到旧数据 | DMA 开始前，按平台要求对发送缓冲区执行 **clean**。 |
| **DMA 写，CPU 读**，如 UART DMA 接收数据 | RAM 已更新，CPU 却命中 Cache 中的旧副本 | DMA 完成、CPU 读取前，按平台要求对接收缓冲区执行 **invalidate**；有些平台还要求在 DMA 开始前处理该缓冲区。 |

```text
发送例子：CPU 把 buffer[0] 改为 0xFF → 新值暂留 Cache
          DMA 若直接从 RAM 读到旧值 0x00 → 屏幕收到错误数据

接收例子：DMA 把 RAM 中 buffer[0] 改为 0x5A
          CPU 若从旧 Cache 读到 0x00 → 误以为没有新数据
```

**不是所有项目都要手工清理 Cache。**有的 MCU 没有启用 D-Cache；有的平台由硬件保证 CPU 与 DMA 一致；若 [I2C 显示屏](#i2c)由 CPU 直接写外设寄存器、没有使用 DMA 缓冲区，也不会按上述方式出现“DMA 读到旧缓冲区”的问题。在 Linux 驱动中应使用 DMA 映射与同步 API，让平台实现处理缓存一致性；裸机或 RTOS 下则遵循芯片手册和驱动要求。[Arm Cortex-M7 缓存维护操作](https://documentation-service.arm.com/static/61efd6602dd99944d051417b?token=)；[Linux DMA API 指南](https://docs.kernel.org/core-api/dma-api-howto.html)

**与 [`volatile`](#volatile) 区分：**`volatile` 约束编译器对对象访问的优化，不会自动把 Cache 中的脏数据写回，也不会让旧缓存行失效。因此遇到 DMA 旧数据问题，单纯给缓冲区加 `volatile` 不能代替正确的缓存同步。
<!-- TOPIC:cache-dma:END -->

<a id="i2c"></a>
## I2C 总线与显示屏通信
<!-- TOPIC:i2c:START -->

下面用 **SSD1306 I2C OLED** 做写入示例。假设屏幕的 **7 位地址是 `0x3C`**，那么“地址 + 写位 `0`”组成的总线地址字节就是 **`0x78`**（`0x3C << 1`）。屏幕的实际地址可能不同；调用 I2C 驱动时，也要确认 API 要求传入 7 位地址还是已左移的地址字节。

<a id="i2c-signals"></a>
### 基础时序：先记住四个信号

- **START（起始）**：SCL 为高时，主控让 SDA 从高变低。
- **数据位**：每个字节按最高位到最低位发送；SCL 为高时，SDA 保持稳定，通常在 SCL 为低时改变。
- **ACK/NACK（应答）**：8 位数据之后还有第 **9 个 SCL 脉冲**。接收方拉低 SDA 是 ACK，保持高电平是 NACK。
- **STOP（停止）**：SCL 为高时，主控让 SDA 从低变高。

<a id="i2c-address-count"></a>
### 7 位地址为什么不是 8 位？一条总线能接多少设备？

**面试问题：为什么是 `2^7` 而不是 `2^8`？“127 个设备”怎么算？**

常见的 7 位寻址格式中，START 后发送的第一个字节由 **7 位设备地址 + 1 位 R/W 方向位**组成。方向位不能算作设备地址，所以 7 位地址共有 `2^7 = 128` 种组合；`0x3C` 这个屏幕地址加写位 `0` 后形成总线字节 `0x78`。按地址格式计算，读位 `1` 会形成 `0x79`，并不代表另一台设备；这只是在解释地址格式，实际设备能否读要看手册。

“127 个设备”不是 I2C 的通用上限：它通常只扣除了 `0x00`，却忽略其他保留地址。按标准普通 7 位设备地址范围 `0x08`～`0x77` 计算，常规可分配地址有 **112 个**（128 减去首尾各 8 个保留地址）。这仍只是地址数量；实际可挂设备数还受地址冲突、总线电容、上拉电阻与速率等条件限制。某些保留地址有特定用途，不能简单当作普通设备地址使用。

参考：[NXP I2C 总线规范 UM10204，保留地址表](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)。

<a id="i2c-examples"></a>
### 显示屏写入与寄存器读取示例

下面的图按**从上到下**的顺序画出同一次事务。每一行字节波形都包含 8 位数据和第 9 拍应答；蓝色是 SDA，灰色是 SCL。

#### 例 1：向 SSD1306 发送“打开显示”命令

```text
START → 0x78 → ACK → 0x00 → ACK → 0xAF → ACK → STOP
         地址+写位       控制字节       显示开启命令
```

`0x00` 是**控制字节**，告诉 SSD1306 后面的内容按“命令”解释；`0xAF` 才是“打开显示”的命令。三个 ACK 都由**屏幕**在各自的第 9 个时钟拉低 SDA 发出。

![SSD1306 打开显示命令的完整 I2C 时序：START、地址 0x78、控制字节 0x00、命令 0xAF、三个 ACK 和 STOP](images/ssd1306-command-write.svg)

#### 例 2：向 SSD1306 写入一个显示数据字节

```text
START → 0x78 → ACK → 0x40 → ACK → 0xFF → ACK → STOP
         地址+写位       控制字节       显示数据
```

`0x40` 也是**控制字节**，这次表示后续内容是写入显存的数据。`0xFF` 的 8 位都是 `1`，对应当前显存列中一页的 8 个垂直像素位。它落在屏幕上的具体位置，取决于此前设置的寻址模式、页地址和列地址；发送多个数据字节时，可以在同一次事务中继续写。

![SSD1306 写显示数据的完整 I2C 时序：START、地址 0x78、控制字节 0x40、数据 0xFF、三个 ACK 和 STOP](images/ssd1306-data-write.svg)

**容易混淆：**例 1 的 `0x00` 与例 2 的 `0x40` 都是控制字节；同一个十六进制值在不同位置可能有不同含义，必须结合前面的控制字节解释。

#### 例 3：设备支持读取时，怎样读寄存器？

这是**通用读流程示意，不是 SSD1306 显存回读**。为方便看每一位，假设另一台可读设备的 7 位地址为 `0x2A`、寄存器地址为 `0x10`，且读出的值为 `0x5A`：

```text
START → 0x54(地址+写) → ACK → 0x10(寄存器地址) → ACK
      → 重复 START → 0x55(地址+读) → ACK → 0x5A(设备发数据) → NACK → STOP
```

前半段的“写”只是告诉设备**想读哪个寄存器**，没有发 STOP，而是用重复 START 切换到读。读出最后一个字节后，**主控**在第 9 拍保持 SDA 为高，回 NACK 表示“读完了”，然后发 STOP。

![通用寄存器读事务时序：先写寄存器地址，再用重复 START 切换为读，最后由主控回 NACK](images/generic-register-read.svg)

**SSD1306 的 I2C 串行接口不提供显存数据回读。**如果你的显示屏项目确实会从屏幕读取数据，请先确认控制器型号及手册中可读的寄存器；读命令、地址和应答方式要以实际控制器为准。

#### 读图时抓住这三点

1. **SCL 高电平期间 SDA 发生下降或上升**，分别表示 START 或 STOP；普通数据位此时应保持不变。
2. **每发完 8 位就看第 9 拍**：写事务中通常由屏幕回 ACK；读事务中数据由设备发，最后一个字节通常由主控回 NACK。
3. **地址和负载要分开看**：`0x3C` 是 7 位设备地址，`0x78` 是加上写位后的总线字节；`0x00`/`0x40` 决定 SSD1306 怎样解释后面的字节。

参考：[SSD1306 数据手册，第 8.1.5 节（I2C 接口）及命令表](https://files.waveshare.com/upload/a/af/SSD1306-Revision_1.1.pdf)。
<!-- TOPIC:i2c:END -->

<a id="c-basics"></a>
## C/C++ 基础

<a id="volatile"></a>
### `volatile`
<!-- TOPIC:volatile:START -->

**面试问题：`volatile` 有什么作用？什么时候用？**

`volatile` 告诉编译器：这个对象的值可能在当前代码看不到的地方发生变化，因此对它的访问不能像普通对象那样被省略或长期保存在寄存器中。典型场景是内存映射的硬件寄存器，以及在满足平台约束时由中断例程和主程序共同访问的标志。

```c
volatile int event_ready = 0;

void interrupt_handler(void) {
    event_ready = 1;
}

int main(void) {
    while (!event_ready) {
        /* 等待中断设置标志 */
    }
    return 0;
}
```

**追问：不加 `volatile` 会怎样？加锁后变量为什么仍可能在寄存器里？** 对硬件寄存器轮询而言，不加 `volatile` 可能使编译器省略重复读取，看不到硬件更新。锁并非“禁止使用寄存器”：CPU 仍会把值读入寄存器运算；互斥锁负责控制并发访问和建立线程间的内存同步。所有线程都正确使用同一把锁保护普通共享变量时，通常无须再加 `volatile`。

**边界：**`volatile` 不保证原子性或线程同步，不能让 `count++` 自动安全。多线程共享数据应使用原子类型或同步机制；中断与主程序之间还要核对目标平台的访问宽度、原子性与必要的临界区。[GCC 对 `volatile` 的说明](https://gcc.gnu.org/onlinedocs/gcc/Volatiles.html)；[POSIX 对互斥锁内存同步的规定](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/V1_chap04.html)。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)。
<!-- TOPIC:volatile:END -->

<a id="static"></a>
### `static`
<!-- TOPIC:static:START -->

**面试问题：`static` 放在不同位置分别是什么意思？**

| 用法 | 效果 |
| --- | --- |
| 函数内的局部变量 | 仍只在所在作用域可见，但具有静态存储期；只初始化一次，跨调用保留值。 |
| 文件作用域的变量或函数 | 具有内部链接，只能由当前翻译单元按名称访问。 |
| C++ 类的静态数据成员 | 属于类，所有实例共享同一成员；一般需要按相应 C++ 规则提供定义或使用 `inline`。 |
| C++ 类的静态成员函数 | 不依赖某个对象调用，没有 `this` 指针。 |
| C99 数组形参中的 `static N` | 声明调用者传入的数组至少有 `N` 个元素；这是接口约定，不能代替运行时检查。 |

```c
int next_count(void) {
    static int count = 0;
    return ++count;  /* 连续调用得到 1、2、3…… */
}

/* 此处的 helper 只在当前 .c 文件内具有链接可见性。 */
static void helper(void) {}

/* C99：要求调用者传入至少 10 个 int。 */
void process_data(int data[static 10]) {
    /* 使用 data[0] 到 data[9] */
}
```

**区分两个概念**：局部 `static` 主要改变存储期；文件作用域 `static` 主要改变链接属性。C++ 静态类成员属于 C++ 用法，和 C 语言本身无关。

**追问：与不加 `static` 的局部变量有什么区别？**例如 `int a = 0; static int b = 0;` 写在同一函数里，每次调用都会重新执行 `a` 的初始化，而 `b` 只初始化一次并保留上次的值。`b` 的生命周期和典型内存位置见[六大内存分区](#memory-layout)；未初始化的普通自动局部变量不能默认当作 0 使用。

**追问：`static` 函数能跨文件使用吗？**另一个 `.c` 文件不能直接按名字调用本文件的 `static` 函数，因为它只有内部链接。如果确实要作为跨文件接口，应去掉函数定义上的 `static`，在头文件中放**声明**，并只在一个 `.c` 文件里放**定义**。不同 `.c` 文件可以各自定义同名 `static` 辅助函数而不冲突；把普通外部函数定义写进被多个 `.c` 文件包含的头文件，通常会造成重复定义。参见 [C 标准草案中的存储期与链接属性](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)。
<!-- TOPIC:static:END -->

<a id="struct-alignment"></a>
### 结构体内存对齐
<!-- TOPIC:struct-alignment:START -->

**面试问题：下列两个结构体各占多少字节？**

```c
struct student {
    int no;
    char name;
    short sex;
};

struct teach {
    char no;
    int name;
    short sex;
};
```

在常见的 `int` 为 4 字节且按 4 字节对齐、`short` 为 2 字节且按 2 字节对齐的 ABI 下：

| 结构体 | 成员布局 | 尾部填充 | `sizeof` |
| --- | --- | --- | --- |
| `student` | `no`：0–3；`name`：4；填充：5；`sex`：6–7 | 0 字节 | **8 字节** |
| `teach` | `no`：0；填充：1–3；`name`：4–7；`sex`：8–9 | 2 字节 | **12 字节** |

成员要放在符合各自对齐要求的位置；结构体整体大小还要满足自身对齐要求，以便结构体数组中的每个元素正确对齐。**实际结果取决于平台 ABI、编译器选项和打包设置**，可用 `sizeof`、`_Alignof`（或 C++ 的 `alignof`）和 `offsetof` 验证。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)。
<!-- TOPIC:struct-alignment:END -->

<a id="malloc-free"></a>
### `malloc` 与 `free`
<!-- TOPIC:malloc-free:START -->

**面试问题：动态分配一个整数、赋值、打印并释放，会输出什么？**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof *p);
    if (p == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        return 1;
    }

    *p = 12345;
    printf("result = %d\n", *p);
    free(p);
    return 0;
}
```

分配成功时输出 `result = 12345`；分配失败时输出错误信息并返回非零值。`malloc` 得到的内存未初始化，必须先赋值再读取；`free(p)` 之后不能继续解引用 `p`。在 C 语言中通常不需要强制转换 `malloc` 的返回值。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)。
<!-- TOPIC:malloc-free:END -->

<a id="freertos"></a>
## FreeRTOS

<a id="freertos-scheduling"></a>
### 任务调度
<!-- TOPIC:freertos-scheduling:START -->

**面试问题：FreeRTOS 如何选择下一个运行的任务？**

调度器从**就绪态**任务中选择优先级最高的任务运行。任务创建时会指定优先级，也可以在运行时通过 `vTaskPrioritySet()` 调整。阻塞或挂起的任务不参与就绪任务的选择。

- **抢占式调度**：启用抢占时，更高优先级任务变为就绪态可触发任务切换。
- **协作式调度**：任务不会因普通抢占规则自动让出 CPU；任务主动让出、阻塞等情况会引发切换。具体行为取决于 FreeRTOS 配置。
- **同优先级任务**：是否按时间片轮转受 `configUSE_TIME_SLICING` 等配置影响。时间片不会让较低优先级任务越过持续就绪的高优先级任务。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)；调度规则参考 [FreeRTOS 官方文档](https://www.freertos.org/Documentation/02-Kernel/02-Kernel-features/01-Tasks-and-co-routines/04-Task-scheduling)。
<!-- TOPIC:freertos-scheduling:END -->

<a id="freertos-starvation"></a>
### 高优先级任务与饥饿
<!-- TOPIC:freertos-starvation:START -->

**面试问题：高优先级任务一直运行，会不会占满 CPU？**

会。在可抢占的优先级调度中，如果高优先级任务始终处于就绪态、持续运行且不阻塞，低优先级任务可能长期得不到 CPU，出现**任务饥饿**。这可能影响低优先级的通信、采样或维护任务。

常见处理方式：

- 让周期性任务通过 `vTaskDelay()` 或 `vTaskDelayUntil()` 进入阻塞态，而不是忙等。例如 `vTaskDelay(pdMS_TO_TICKS(10));`。
- 按实时要求设置优先级，缩短高优先级任务的连续执行时间。
- 共享资源使用合适的互斥与同步机制，并缩短持锁时间。

**注意**：`taskYIELD()` 只是让出一次调度机会；若当前任务仍是最高优先级的就绪任务，它并不能保证低优先级任务运行。若希望低优先级任务获得 CPU，高优先级任务通常需要阻塞或挂起。

来源：[海康 BSP 嵌入式开发实习面试经验](https://chrisy0618.github.io/2025/04/15/hello-world/)；任务饥饿说明参考 [FreeRTOS 官方文档](https://www.freertos.org/Documentation/02-Kernel/02-Kernel-features/01-Tasks-and-co-routines/04-Task-scheduling)。
<!-- TOPIC:freertos-starvation:END -->

<a id="freertos-communication"></a>
### 任务间通信
<!-- TOPIC:freertos-communication:START -->

**面试问题：FreeRTOS 任务间通信有哪些方式？**

可以先按用途回答：**传递数据用队列，通知事件用任务通知或信号量，保护共享资源用互斥锁**。

| 机制 | 适合的场景 | 关键点 |
| --- | --- | --- |
| 队列 `Queue` | 任务间传递固定大小的消息 | 队列会复制消息内容；若传指针，只复制指针，需管理所指数据的生命周期。 |
| 直接任务通知 `Task Notification` | 向确定的单个任务发事件或较小的值 | 不必单独创建队列或信号量，开销较小；接收方只能是指定任务。 |
| 二值／计数信号量 | 完成通知、事件计数、可用资源计数 | 主要用于同步，不用来承载一段完整业务数据。 |
| 互斥锁 `Mutex` | 多任务访问共享设备或共享变量 | 用于互斥，支持优先级继承；与普通二值信号量的用途不同。 |
| 事件组 `Event Group` | 等待一个或多个条件同时满足 | 每一位可代表一个事件或状态。 |
| 流／消息缓冲区 | 传连续字节流或变长消息 | 默认按单写入者、单读取者设计；多写或多读要另外同步。 |

**显示屏项目例子：**业务任务把“绘制请求”放入队列，由显示任务统一更新屏幕；若多个任务必须直接访问同一条 I2C 总线，则用互斥锁保护总线事务。中断中要使用对应的 `...FromISR` API，不能直接套用会阻塞的任务 API。

参考：[FreeRTOS 任务间协调文档](https://docs.aws.amazon.com/freertos/latest/userguide/inter-task-coordination.html)。
<!-- TOPIC:freertos-communication:END -->

<a id="freertos-task-creation"></a>
### 创建任务
<!-- TOPIC:freertos-task-creation:START -->

**面试问题：怎么创建一个 RTOS 任务？**

1. 写一个任务入口函数，形如 `void DisplayTask(void *arg)`；任务通常在循环中处理工作，没事时阻塞等待事件，而不是空转。
2. 调用 `xTaskCreate(DisplayTask, "display", stackDepth, arg, priority, &handle)`，传入任务函数、名称、栈深度、参数、优先级和句柄地址，并检查是否返回 `pdPASS`。
3. 创建好需要的任务和通信对象后，调用 `vTaskStartScheduler()` 启动调度器。

`xTaskCreate()` 为任务控制块和栈动态分配内存；`xTaskCreateStatic()` 则由调用者提供这两块内存。**标准 FreeRTOS 的栈深度按 `StackType_t` 元素数计，通常称为“字”，不是字节数**；使用厂商改造版时应核对该平台的 API 文档。

参考：[FreeRTOS 任务创建说明](https://github.com/FreeRTOS/FreeRTOS-Kernel-Book/blob/main/ch04.md)。
<!-- TOPIC:freertos-task-creation:END -->

<a id="freertos-stack-check"></a>
### 检查任务栈
<!-- TOPIC:freertos-stack-check:START -->

**面试问题：怎么知道任务堆栈使用情况？**

保存任务句柄，调用 `uxTaskGetStackHighWaterMark(handle)`；传 `NULL` 可查询当前任务。返回值是任务运行以来**最少剩余过的栈空间**，并非当前瞬间的剩余量；越接近 0，距离栈溢出越近。

例如创建任务时分配 256 个栈元素，测得高水位余量为 40 个元素，则曾经至少用到约 216 个元素；若每个 `StackType_t` 占 4 字节，最低余量约为 160 字节。这个值需在最深函数调用、异常处理等路径都跑过后才有参考意义，调试时还可开启 `configCHECK_FOR_STACK_OVERFLOW` 并实现 `vApplicationStackOverflowHook()`。

参考：[FreeRTOS 栈检查说明](https://github.com/FreeRTOS/FreeRTOS-Kernel-Book/blob/main/ch13.md)。
<!-- TOPIC:freertos-stack-check:END -->

<a id="freertos-linux-stack"></a>
### FreeRTOS 与 Linux 的栈
<!-- TOPIC:freertos-linux-stack:START -->

**面试问题：FreeRTOS 和 Linux 的栈区有什么区别？**

两者的每个任务／线程都有自己的执行栈，主要区别在**地址空间和分配方式**：

| 方面 | 常见 MCU 上的 FreeRTOS | Linux 用户线程 |
| --- | --- | --- |
| 所在位置 | 通常是创建任务时分配或提供的一块固定 RAM。 | 位于所属进程的虚拟地址空间；新线程有自己的用户栈。 |
| 与其他任务／线程的关系 | 普通移植通常没有进程级地址空间隔离；部分 MPU 移植可限制内存访问。 | 同一进程的线程共享堆和全局数据，但不共用各自的用户栈。 |
| 大小与保护 | 创建时确定大小，需结合高水位和溢出检测评估。 | 可通过线程属性指定大小，通常有保护页；Linux 线程运行内核代码时还使用独立的内核栈。 |

**不要把“向上增长还是向下增长”当作二者的固定区别**：栈增长方向取决于处理器架构和 ABI。

参考：[FreeRTOS 任务创建说明](https://github.com/FreeRTOS/FreeRTOS-Kernel-Book/blob/main/ch04.md)、[FreeRTOS MPU 支持](https://www.freertos.org/Security/04-FreeRTOS-MPU-memory-protection-unit)、[Linux pthreads 手册](https://man7.org/linux/man-pages/man7/pthreads.7.html)。
<!-- TOPIC:freertos-linux-stack:END -->

<a id="linux"></a>
## Linux 进程与线程

<a id="linux-thread-stack-size"></a>
### 新线程的默认栈大小
<!-- TOPIC:linux-thread-stack-size:START -->

**面试问题：Linux 创建一个线程，默认栈空间有多大？**

不能只回答“固定 8 MB”。在常见的 Linux glibc/NPTL 实现中，**程序启动时**的 `RLIMIT_STACK` 软限制若为有限值，就决定新线程的默认栈大小；不少环境恰好配置成 **8 MiB**。若该限制为 `unlimited`，多数架构使用 **2 MiB**，POWER 和 Sparc-64 使用 **4 MiB**。这里主要是虚拟地址空间的栈映射，不代表创建线程时就占满相同大小的物理内存。

可用 `ulimit -s` 查看当前 shell 的栈限制；用 `pthread_getattr_np()` 配合 `pthread_attr_getstacksize()` 查询已创建线程的实际栈大小；创建线程时可通过 `pthread_attr_setstacksize()` 指定大小。主线程的栈不要与新建 pthread 的默认栈简单混为一谈。

参考：[Linux `pthread_create(3)`](https://man7.org/linux/man-pages/man3/pthread_create.3.html)、[`pthread_getattr_np(3)`](https://man7.org/linux/man-pages/man3/pthread_getattr_np.3.html)。
<!-- TOPIC:linux-thread-stack-size:END -->

<a id="linux-process-creation"></a>
### 创建进程
<!-- TOPIC:linux-process-creation:START -->

**面试问题：怎么创建一个进程？**

常见流程是 **`fork()` 创建子进程 → 子进程按需调用 `execve()` 运行新程序 → 父进程用 `waitpid()` 回收子进程**。`fork()` 成功时在子进程返回 `0`，在父进程返回子进程 PID；失败返回 `-1`。`execve()` 是用新程序替换当前进程中的程序映像，**它本身不创建新进程**。

`fork()` 后父子进程各有自己的虚拟地址空间；Linux 通常通过写时复制避免在创建时立即复制所有物理内存。父进程要适时回收结束的子进程，避免僵尸进程。

参考：[Linux `fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html)、[`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html)、[`waitpid(2)`](https://man7.org/linux/man-pages/man2/waitpid.2.html)。
<!-- TOPIC:linux-process-creation:END -->

<a id="linux-thread-creation"></a>
### 创建线程
<!-- TOPIC:linux-thread-creation:START -->

**面试问题：怎么创建一个线程？**

Linux C 程序常用 `pthread_create(&tid, NULL, worker, arg)`：第一个参数接收线程 ID，第二个是线程属性（`NULL` 表示默认属性），第三个是线程入口函数，第四个是传给入口函数的参数。成功返回 `0`，失败直接返回错误号。

线程结束后，用 `pthread_join()` 等待并取得结果，或者把它设置为 detached，使资源在结束后自动回收。新线程与同进程其他线程共享堆、全局变量和文件描述符，但有自己的栈；访问共享数据时要考虑同步。

参考：[Linux `pthread_create(3)`](https://man7.org/linux/man-pages/man3/pthread_create.3.html)、[`pthreads(7)`](https://man7.org/linux/man-pages/man7/pthreads.7.html)。
<!-- TOPIC:linux-thread-creation:END -->

<a id="threads-vs-processes"></a>
### 多线程与多进程
<!-- TOPIC:threads-vs-processes:START -->

**面试问题：实现同一个业务，多线程和多进程有什么区别？**

| 方面 | 多线程 | 多进程 |
| --- | --- | --- |
| 数据共享 | 同一进程内直接共享堆和全局数据，交换信息方便，但要处理竞争和锁。 | 地址空间相互隔离，交换数据通常需要管道、套接字或共享内存等 IPC。 |
| 故障影响 | 一个线程的严重内存错误通常会影响整个进程。 | 进程间隔离较强，单个子进程退出通常不会直接终止其他进程。 |
| 开销与选择 | 创建和共享数据通常较轻便，适合同一服务内密切协作的工作。 | 隔离、独立部署或独立生命周期更方便，但 IPC 与管理通常更复杂。 |

**两者都能利用多核。**不要笼统断言“多线程一定更快”；要根据数据共享需求、故障隔离、IPC 成本和具体负载选择。

参考：[Linux `pthreads(7)`](https://man7.org/linux/man-pages/man7/pthreads.7.html)、[`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html)。
<!-- TOPIC:threads-vs-processes:END -->

<a id="tcp-server-connection"></a>
## TCP 服务端建立连接
<!-- TOPIC:tcp-server-connection:START -->

**面试问题：网络编程创建连接时，服务端需要调用哪些 API？**

典型 TCP 服务端流程：

```text
socket() → [setsockopt()] → bind() → listen() → accept() → recv()/send() → close()
```

`socket()` 创建套接字，`bind()` 绑定本地地址和端口，`listen()` 进入监听状态，`accept()` 接收一个客户端连接并返回**新的已连接 socket**。原监听 socket 仍可继续接受新连接；读写使用新 socket。`setsockopt()` 按需设置选项，例如地址复用。处理大量并发连接时，还可结合 `poll`、`epoll`、线程或进程。

**区分客户端：**主动发起连接通常由客户端调用 `connect()`，它不属于服务端上述基本流程。

参考：[Linux `socket(2)`](https://man7.org/linux/man-pages/man2/socket.2.html)、[`bind(2)`](https://man7.org/linux/man-pages/man2/bind.2.html)、[`accept(2)`](https://man7.org/linux/man-pages/man2/accept.2.html)。
<!-- TOPIC:tcp-server-connection:END -->

<a id="debug-interface"></a>
## 嵌入式调试接口排障
<!-- TOPIC:debug-interface:START -->

**面试问题：调试接口遇到过哪些问题？请具体举例。**

应按“**现象 → 排查 → 原因 → 修复与验证**”讲自己的真实经历，不要只回答“线接错了”。如果遇到过 SWD 无法连接 MCU，可以这样组织思路：

1. **现象**：下载程序后，调试器无法识别或连接目标芯片。
2. **基础排查**：确认目标板供电、共地、SWDIO、SWCLK、NRST 连线；核对调试器使用 SWD 而非 JTAG 模式，并尝试降低调试时钟。
3. **缩小范围**：尝试在复位状态下连接。如果能连上，再检查固件是否重配了调试引脚、很快进入低功耗模式，或触发了复位循环。
4. **修复验证**：针对找到的原因修改固件或连接配置，重新烧录，并验证冷启动后仍能稳定连接。

这是**排查示例，不代表已经发生在你的项目中**。面试时应替换成自己的设备、报错、测量结果和最终原因；如果没有遇到过，就如实说“我会按这个顺序排查”。参考：[Arm 调试器连接目标设备指南](https://documentation-service.arm.com/static/6763f2ad3f2a9a07789de3ff?token=)。
<!-- TOPIC:debug-interface:END -->

<a id="switch-score"></a>
## 编程题：只用 `switch case` 判断分数
<!-- TOPIC:switch-score:START -->

**面试问题：输入整数，80～90（含边界）输出“良好”，其余输出“未识别”；只允许使用 `switch case`。**

```c
#include <stdio.h>

int main(void) {
    int score = 0;
    (void)scanf("%d", &score); /* 非整数输入保留初始值 0，落入 default */

    switch (score) {
        case 80: case 81: case 82: case 83: case 84:
        case 85: case 86: case 87: case 88: case 89:
        case 90:
            puts("良好");
            break;
        default:
            puts("未识别");
            break;
    }
    return 0;
}
```

`case` 逐个匹配整数常量，所以这里列出 80～90 的 11 个值；`break` 防止继续执行下一分支。若题目没有“只用 `switch case`”的限制，直接判断 `score >= 80 && score <= 90` 更清晰。`case 80 ... 90` 是 GCC 的范围扩展，不是标准 C 写法。
<!-- TOPIC:switch-score:END -->
