# 学习和问题记录

### 第二阶段

#### 资料

[rCore-Camp-Guide-2024A](https://learningos.cn/rCore-Camp-Guide-2024A/) [rCore-Tutorial-Book-v3](https://rcore-os.cn/rCore-Tutorial-Book-v3/)

#### 仓库

[rCore-Camp-Code-2024A]([LearningOS/2024a-rcore-Charles-De-Geulle-Goah: rcore-camp-2024a-classroom-2024a-rcore-rCore-Camp-Code-2024A created by GitHub Classroom](https://github.com/LearningOS/2024a-rcore-Charles-De-Geulle-Goah))

#### 2024/10/14

##### 事件

- 克隆实验代码，配置实验环境
- 配置了qemu的risc-v环境

##### 学习内容

- 熟悉了实验运行的流程

  ```shell
  $ git checkout ch1      #进入章节1
  $ cd os                 #进入os目录
  $ make run LOG=TRACE    #运行本章代码，并将日志级别设为 TRACE
  ```

- 配置qemu环境时，当我们编译完成后，可以运行`sudo make install` 将 Qemu 安装到 `/usr/local/bin` 目录下，但这样经常会引起 冲突。更好的办法是在`~/.bashrc`中添加`riscv`模拟的环境变量,例如：

```shell
export PATH=/home/guojia153/qemu-project/qemu/build_riscv/:$PATH
```

- `riscv64gc-unknown-none-elf` 的 CPU 架构是 riscv64gc，厂商是 unknown，操作系统是 none， elf 表示没有标准的运行时库，没有任何系统调用的封装支持，但可以生成 ELF 格式的执行程序。

##### 问题

- 在配置qemu的时候，我之前曾经配过aarch64的环境，结果在qemu目录下运行.`/configure`命令，会出现`ERROR: ./build dir already exists and was not previously created by configure`的错误

##### 原因

- 之前已经有`build`目录了，要想别的办法

##### 解决方法

```shell
mkdir build-new   #创建一个新目录用来存放riscv的东西
cd build-new      #进入目录
../configure      #重新运行configure脚本，注意相对路径的变化
make
```

#### 2024/10/15

##### 事件

- 了解RISC-V的特权等级知识
- 熟悉了实验1的流程和涉及到的知识点

##### 学习内容

- 软件的执行环境栈中，下层是上层的**执行环境**，下层为上层提供**接口**。
- 平台的信息通过**目标三元组**表示，包括**CPU指令集，OS类型和标准库类型**（这正好是软件栈中app层的下面三层）
- 查看平台信息：

```
$ rustc --version --verbose
   rustc 1.61.0-nightly (68369a041 2022-02-22)
   binary: rustc
   commit-hash: 68369a041cea809a87e5bd80701da90e0e0a4799
   commit-date: 2022-02-22
   host: x86_64-unknown-linux-gnu
   release: 1.61.0-nightly
   LLVM version: 14.0.0
```

host一项就是平台信息，其值为`x86_64-unknown-linux-gnu`，CPU 架构是 x86_64，CPU 厂商是 unknown，操作系统是 linux，运行时库是 gnu libc。

而平台信息的通用命名法则是：

**arch [-vendor] [-os] [-(gnu)eabi]**

arch：平台架构，如x86_64，aarch64，MIPS等

vendor：厂商

os：支持的操作系统

-(gnu)eabi：嵌入式应用二进制接口。支持的库。这里有三种比较常见的值：

- libc：GNU标准C库
- eabi：嵌入式ABI
- elf：没有标准的运行时库，没有任何系统调用的封装支持，但可以生成 ELF 格式的执行程序

==============================================================================

- rust用到的一些注解：

  - #![no_std]:不使用标准库
  - #![no_main]:不使用真正意义的main函数
  - #[panic_handler]：标记panic处理函数，告诉编译器被[panic_handler]标记的函数就是我们的panic处理函数

- 程序分析工具：

  - file：输出文件格式
  - rust-readobj -h:输出文件头信息
  - rust-objdump -S：反汇编导出汇编程序
  - rust -objcopy --binary：ELF转换为binary文件

- qemu的使用：

  - 用户态模拟，如 `qemu-riscv64` 程序， 能够模拟不同处理器的用户态指令的执行，并可以直接解析ELF可执行文件， 加载运行那些为不同处理器编译的用户级Linux应用程序。使用方是为`qemu-riscv64 [程序] `
  - 内核态模拟，如 `qemu-system-riscv64` 程序， 能够模拟一个完整的基于不同CPU的硬件系统，包括处理器、内存及其他外部设备，支持运行完整的操作系统。在实验里，裸机启动就使用这种模式：

  ```
  qemu-system-riscv64 \
              -machine virt \
              -nographic \
              -bios $(BOOTLOADER) \
              -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
              
  # -bios $(BOOTLOADER) 意味着硬件加载了一个 BootLoader 程序，即 RustSBI
  # -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA) 表示os在硬件内存中的特定位置  $(KERNEL_BIN) 放置了操作系统的二进制代码 。 $(KERNEL_ENTRY_PA) 的值是 0x80200000 。
  ```

- 用户态执行环境的搭建

  - 禁用标准库而改用core，所做的工作包括：移除`println!`宏，提供`panic_handler`，移除`main`函数
  - 编写`_start()`函数作为程序的入口
  - 编写`sys_exit()`函数作为退出函数，这个函数封装了操作系统提供的系统调用

- 裸机执行环境的搭建

  - 操作系统启动流程：上电后，寄存器初始化，PC跳转至0x1000，运行一段引导程序后跳转至0x80000000执行RUSTSBI执行硬件初始化，完成硬件初始化后，会跳转到0x80200000，执行OS的第一条指令
  - 修改配置文件config.toml：

  ```
  // os/.cargo/config
  [build]
  target = "riscv64gc-unknown-none-elf"   #设置目标平台
  
  [target.riscv64gc-unknown-none-elf]  #指定链接脚本，链接脚本能让我们把操作系统放在正确位置
  rustflags = [
      "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
  ]
  ```

  - `asm()!`宏可以让我们在rust源代码中嵌入汇编语句。这里我们用`ecall`指令来调用SBI
  - 正确配置操作系统栈空间的布局，包括设置操作系统运行时栈和设置应用调用入口两个工作：

  ```
      .section .text.entry
      .globl _start
  _start:   #操作系统的入口点
      la sp, boot_stack_top   #将栈指针设为boot_stack_top 
      call rust_main          #设置应用调用入口，rust_main函数
  
      .section .bss.stack  #栈名
      .globl boot_stack_lower_bound
  boot_stack_lower_bound:  #栈底
      .space 4096 * 16     #栈大小
      .globl boot_stack_top  #栈顶
  boot_stack_top:
  ```

  - 操作系统运行流程：从entry.asm的`_start`开始运行，运行到`call rust_main`时跳转到main.rs的`rust_main()`函数
  - 在main.rs中，`core::arch::global_asm!(include_str!("entry.asm"));`的作用是将同目录下的entry.asm文件嵌入到代码中

##### 问题

- asm()!宏的用法是什么
- ecall指令的用法是什么
- 如何设置栈名，栈大小和栈顶，栈底

#### 2024/10/16

##### 事件

- 学习了实验2中实现应用程序和批处理操作系统的内容

##### 学习内容

- 批处理程序：多个程序打包到一起加载到计算机，一个一个地依次执行
- 特权级：特权级的目的是保护OS不受出错程序的干扰，实现用户态和内核态的分离，实现用户态和内核态的隔离
- rust的一些注解：
  - #![feature(linkage)] ：启用弱链接特性
  - #[link_section = ".text.entry"]：将函数编译后的代码放在代码段`.text.entry`中
  - #[linkage = "weak"]：将符号定义为弱标志
- 弱标志的理解：比如说lib.rs和bin目录下都有main，而lib.rs下的main被标记为弱标志，那么bin目录下的main具有更高的优先级。也就是说链接器会使用 `bin` 目录下的函数作为 `main` 。
- 对于用户态程序，我们可以使用ecall指令调用批处理系统接口，主要流程是：应用程序调用ecall，由于应用程序运行在用户态（即 U 模式）， `ecall` 指令会触发名为 `Environment call from U-mode` 的异常， 并 Trap 进入 S 模式执行批处理系统针对这个异常特别提供的服务程序。 
- RISC-V 寄存器编号从 `0~31` ，表示为 `x0~x31` 。 其中： - `x10~x17` : 对应 `a0~a7` - `x1` ：对应 `ra`
- ecall约定：寄存器 `a0~a6` 保存系统调用的参数， `a0` 保存系统调用的返回值， 寄存器 `a7` 用来传递 syscall ID
- 生成应用程序二进制文件的流程：
  - 对于 `src/bin` 下的每个应用程序， 在 `target/riscv64gc-unknown-none-elf/release` 目录下生成一个同名的 ELF 可执行文件；
  - 使用 objcopy 二进制工具删除所有 ELF header 和符号，得到 `.bin` 后缀的纯二进制镜像文件。 它们将被链接进内核，并由内核在合适的时机加载到内存。
- link_app.S将应用程序链接到内核，从这段代码可以学会一些汇编语法

```asm
.align 3
 4    .section .data  #表明这是一个数据段
 5    .global _num_app  #定义一个全局变量_num_app
 6_num_app:  #是一个数组
 7    .quad 3  #第一个元素为数组大小
 8    .quad app_0_start   #后面则按照顺序放置每个应用程序的起始地址
 9    .quad app_1_start
10    .quad app_2_start
11    .quad app_2_end   #最后一个应用程序的结束位置
12
13    .section .data
14    .global app_0_start  #指定开始位置
15    .global app_0_end    #指定结束位置
16app_0_start:
17    .incbin "../user/target/riscv64gc-unknown-none-elf/release/hello_world.bin"  #插入应用二进制程序
18app_0_end:
19
20    .section .data
21    .global app_1_start
22    .global app_1_end
23app_1_start:
24    .incbin "../user/target/riscv64gc-unknown-none-elf/release/bad_address.bin"
25app_1_end:
```

- 在操作系统中我们需要实现一个应用管理器对象AppManager，其他操作系统也会有这玩意，有的叫elfloader
- 用容器 `UPSafeCell` 包裹 `AppManager` 是为了防止全局对象 `APP_MANAGER` 被重复获取。
-  `lazy_static!` 可以让静态变量第一次被使用到的时候才会进行实际的初始化工作。

##### 问题

- 暂无

#### 2024/10/17

##### 事件

- 学习了实验2有关特权级切换的知识

##### 学习内容

- 控制状态寄存器CSR：

| CSR 名  | 该 CSR 与 Trap 相关的功能                                    |
| ------- | ------------------------------------------------------------ |
| sstatus | `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
| sepc    | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址 |
| scause  | 描述 Trap 的原因                                             |
| stval   | 给出 Trap 附加信息                                           |
| stvec   | 控制 Trap 处理代码的入口地址                                 |

- 特权级切换需要由硬件和OS共同完成，其中硬件完成：
  - `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级（U/S）。
  - `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。
  - `scause/stval` 分别会被修改成这次 Trap 的原因以及相关的附加信息。
  - CPU 会跳转到 `stvec` 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。
- 特权级切换时，OS会进入Trap处理入口并报错之前的上下文（寄存器）
-  `stvec` 的MODE位于[1:0].当 MODE 字段为 0 的时候， `stvec` 被设置为 Direct 模式，此时进入 S 模式的 Trap 无论原因如何，处理 Trap 的入口地址都是 `BASE<<2` ， CPU 会跳转到这个地方进行异常处理。而 `stvec` 还可以被设置为 Vectored 模式
- risc-v的栈是向下生长的
- 在批处理系统初始化时，我们需要修改 `stvec` 寄存器来指向正确的 Trap 处理入口点。
- Trap处理流程：首先通过 `__alltraps` 将 Trap 上下文保存在内核栈上，然后跳转到使用 Rust 编写的 `trap_handler` 函数 完成 Trap 分发及处理。当 `trap_handler` 返回之后，使用 `__restore` 从保存在内核栈上的 Trap 上下文恢复寄存器。最后在`__restore` 中通过一条 `sret` 指令回到应用程序执行。
- `csrrw` 指令原型是 csrrw rd, csr, rs 可以将 CSR 当前的值读到通用寄存器 rd 中，然后将 通用寄存器 rs 的值写入该 CSR 。
- `.rept`类似于循环
- Trap的原因是`Trap::Exception(Exception::IllegalInstruction)`：系统调用
- `syscall`函数也是根据sycall ID分发到具体handler的
- 当批处理操作系统初始化完成，或者是某个应用程序运行结束或出错的时候，我们要调用 `run_next_app` 函数切换到下一个应用程序。此时 CPU 运行在 S 特权级，我们需要让它切换到U特权级

##### 问题

- 感觉汇编相关的程序好难看懂啊，我觉得需要系统学习RISC-V相关内容但是感觉时间不够

#### 2024/10/18

##### 事件

- 学习了risc-v汇编相关基础知识

##### 学习内容

- 本次学习是通过《RISC-V-Reader-Chinese-v2p1》学习的，后续的学习和编程，很多东西可以通过此书查阅
- riscv的指令图示：26页图2.1
- riscv有6中指令格式：26页图2.2
- riscv的寄存器主要分为指针、功能、保存、临时四类寄存器
- 函数调用过程（示例在45页左右）：
  - 开头：
    - 调整栈指针（sp 寄存器）分配栈帧
    - 保存返回地址（ra 寄存器）
    - 按需保存其它寄存器
  - 函数体
  - 结束：
    - 按需恢复其它寄存器
    - 恢复返回地址
    - 释放栈帧空间
    - 返回调用点
- 汇编指示符：见图3.9
- risc-v的各种指令以及用法：在“附录A-RISC-V指令表“里面去查

##### 问题

- 特权转换，中断处理和虚拟内存相关的内容没有看懂。后续可能去看官方那45页的PPT