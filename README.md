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