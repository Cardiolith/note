## Linux



### 安装软件

CentOS:	rpm, yum

Ubuntu:	apt

### 运行程序

- 命令行运行
- 后台运行 nohup
- 服务运行 systemctl service



### 系统调用

fork 创建进程；

brk，mmap 在堆中分配内存。

brk分配的内存会和原来的堆连接在一起。

mmap重新划分一块区域分配。

**glibc**: Linux 下使用的开源的标准 C 库，它是 GNU 发布的 libc 库，封装了系统调用。

 ![img](https://static001.geekbang.org/resource/image/ff/f0/ffb6847b94cb0fd086095ac263ac4ff0.jpg?wh=2491*1221)

![img](https://static001.geekbang.org/resource/image/3a/23/3afda18fc38e7e53604e9ebf9cb42023.jpeg?wh=2749*1882)

![img](https://static001.geekbang.org/resource/image/2d/1c/2dc8237e996e699a0361a6b5ffd4871c.jpeg?wh=2770*1849)

![img](https://static001.geekbang.org/resource/image/e2/76/e2e92f2239fe9b4c024d300046536d76.jpeg?wh=4042*1705)



![img](https://static001.geekbang.org/resource/image/5f/fc/5f364ef5c9d1a3b1d9bb7153bd166bfc.jpeg?wh=1796*1148)

BIOS -> bootloader

操作系统在哪儿呢？一般都会在安装在硬盘上，在 BIOS 的界面上。你会看到一个启动盘的选项。启动盘有什么特点呢？它一般在第一个扇区，占 512 字节，而且以 0xAA55 结束。这是一个约定，当满足这个条件的时候，就说明这是一个启动盘，在 512 字节以内会启动相关的代码。

**Grub2**，全称 Grand Unified Bootloader Version 2.

grub2 第一个要安装的就是 boot.img。它由 boot.S 编译而成，一共 512 字节，正式安装到启动盘的第一个扇区。这个扇区通常称为 MBR（Master Boot Record，主引导记录 / 扇区）。

由于 512 个字节实在有限，**boot.img** 做不了太多的事情。它能做的最重要的一个事情就是加载 grub2 的另一个镜像 core.img。

引导扇区就是你找到的门卫，虽然他看着档案库的大门，但是知道的事情很少。他不知道你的宝典在哪里，但是，他知道应该问谁。门卫说，档案库入口处有个管理处，然后把你领到门口。core.img 就是管理处，它们知道的和能做的事情就多了一些。**core.img** 由 lzma_decompress.img、diskboot.img、kernel.img 和一系列的模块组成，功能比较丰富，能做很多事情。

![img](https://static001.geekbang.org/resource/image/2b/6a/2b8573bbbf31fc0cb0420e32d07b196a.jpeg?wh=2489*1520)

#### 从实模式切换到保护模式

第一项是**启用分段**，就是在内存里面建立段描述符表，将寄存器里面的段寄存器变成段选择子，指向某个段描述符，这样就能实现不同进程的切换了。第二项是**启动分页**。能够管理的内存变大了，就需要将内存分成相等大小的块，这些我们放到内存那一节详细再讲。

![img](https://static001.geekbang.org/resource/image/0a/6b/0a29c1d3e1a53b2523d2dcab3a59886b.jpeg?wh=1819*4309)