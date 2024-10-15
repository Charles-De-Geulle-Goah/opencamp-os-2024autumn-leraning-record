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