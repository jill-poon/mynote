# Zero-Copy

## 传统 IO

- 基于传统的 IO 方式，底层实际上通过调用 `read()` 和 `write()` 来实现。
- 通过 `read()` 把数据从**硬盘**读取到**内核缓冲区**，再**复制到用户缓冲区**；然后再通过 `write()` 写入到**socket 缓冲区**，最后**写入网卡设备**。

![mmap_01.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630722045000/b461f4a1866643b4b95624e3d2b85715.png)

- 用户空间指的就是用户进程的运行空间，内核空间就是内核的运行空间。
- 如果进程运行在内核空间就是内核态，运行在用户空间就是用户态。
- 为了安全起见，他们之间是互相隔离的，而在用户态和内核态之间的上下文切换也是比较耗时的。
- 从上面我们可以看到，一次简单的 IO 过程产生了 4 次上下文切换，这个无疑在高并发场景下会对性能产生较大的影响。

## DMA 拷贝

> 那么什么又是 DMA 拷贝呢？

- 因为对于一个 IO 操作而言，都是通过 CPU 发出对应的指令来完成，但是相比 CPU 来说，IO 的速度太慢了，CPU 有大量的时间处于等待 IO 的状态。
- 因此就产生了 DMA（Direct Memory Access）直接内存访问技术，本质上来说他就是一块主板上独立的芯片，通过它来进行内存和 IO 设备的数据传输，从而减少 CPU 的等待时间。
- 但是无论谁来拷贝，频繁的拷贝耗时也是对性能的影响。

## 零拷贝

- 零拷贝技术是指计算机执行操作时，CPU 不需要先将数据从某处内存复制到另一个特定区域，这种技术通常用于通过网络传输文件时节省 CPU 周期和内存带宽。
- 那么对于零拷贝而言，**并非真的是完全没有数据拷贝**的过程，只不过是减少**用户态和内核态的切换次数**以及 **CPU 拷贝的次数**。

## mmap+write

- Mmap+write 简单来说就是使用 mmap 替换了 read+write 中的 read 操作，**减少了一次 CPU 的拷贝**。
- Mmap 主要实现方式是将 **读缓冲区的地址** 和 **用户缓冲区的地址** ***进行映射*** ，内核缓冲区和应用缓冲区 **共享**，从而减少了从读缓冲区到用户缓冲区的一次 CPU 拷贝。
- mmap 的方式节省了一次 CPU 拷贝，同时由于用户进程中的内存是虚拟的，只是映射到内核的读缓冲区，所以可以节省一半的内存空间，比较适合大文件的传输

 ![mmap_02.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630722045000/d303ba82fa4443b6b66989566820bf74.png)

## sendfile

- 相比 mmap 来说，sendfile 同样减少了一次 CPU 拷贝，而且还减少了 2 次上下文切换。
- sendfile 是 Linux 2.1 内核版本后引入的一个系统调用函数，通过使用 sendfile 数据可以直接在**内核空间**进行传输，因此避免了用户空间和内核空间的拷贝，同时由于使用 sendfile 替代了 read+write 从而节省了**一次系统调用**，也就是**2次上下文切换**。
- sendfile 方法 IO 数据对用户空间完全不可见，所以只能适用于完全不需要用户空间处理的情况，比如静态文件服务器。

![mmap_03.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630722045000/0b1825a42e9b42958f628ed172fb3bb4.png)

## Sendfile+DMA Scatter/Gather

- Linux 2.4 内核版本之后对 sendfile 做了进一步优化，通过引入新的**硬件支持**，这个方式叫做 DMA Scatter/Gather 分散/收集功能。
- 它将读缓冲区中的数据描述信息--内存地址和偏移量记录到 socket 缓冲区，由 DMA 根据这些将数据从读缓冲区拷贝到网卡，相比之前版本减少了一次 CPU 拷贝的过程
- DMA gather 和 sendfile 一样数据对用户空间不可见，而且需要硬件支持，同时输入文件描述符只能是**文件**，但是过程中完全没有 **CPU 拷贝**过程，极大提升了性能。

![mmap_04.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630722045000/685cb43d658746f993c0c03ace042773.png)

## 总结

- 由于 CPU 和 IO 速度的差异问题，产生了 DMA 技术，通过 DMA 搬运来减少 CPU 的等待时间。
- 传统的 IO `read` + `write` 方式会产生 2 次 DMA 拷贝+2 次 CPU 拷贝，同时有 4 次上下文切换。
- 而通过 mmap+write 方式则产生 2 次 DMA 拷贝+1 次 CPU 拷贝，4 次上下文切换，通过内存映射减少了一次 CPU 拷贝，可以减少内存使用，适合大文件的传输。
- Sendfile 方式是新增的一个系统调用函数，产生 2 次 DMA 拷贝+1 次 CPU 拷贝，但是只有 2 次上下文切换。因为只有一次调用，减少了上下文的切换，但是用户空间对 IO 数据不可见，适用于静态文件服务器。
- Sendfile+DMA gather 方式产生 2 次 DMA 拷贝，没有 CPU 拷贝，而且也只有 2 次上下文切换。虽然极大地提升了性能，但是需要依赖新的硬件设备支持。
