# Zero-Copy

## 传统 IO

- 基于传统的 IO 方式，底层实际上通过调用 \_\_\_\_\_\_\_\_ 和 \_\_\_\_\_\_\_\_ 来实现。
- 通过 `read()` 把数据从 \_\_\_\_\_\_\_\_ 读取到\_\_\_\_\_\_\_\_，再复制到\_\_\_\_\_\_\_\_；然后再通过 `write()` 写入到\_\_\_\_\_\_\_\_，最后写入\_\_\_\_\_\_\_\_。

整个过程发生了 \_\_\_\_\_\_\_\_ 次用户态和内核态的 \_\_\_\_\_\_\_\_ 和 \_\_\_\_\_\_\_\_ 次 \_\_\_\_\_\_\_\_

## DMA 拷贝

- 相比 CPU 来说，IO 的速度太慢了，CPU 有大量的时间处于等待 IO 的状态。
- 因此就产生了 DMA（ \_\_\_\_\_\_\_\_ ）直接内存访问技术，本质上来说他就是一块主板上独立的芯片，通过它来进行内存和 IO 设备的数据传输，从而减少 CPU 的等待时间。

## 零拷贝

- 零拷贝技术是指计算机执行操作时，CPU 不需要先将数据从某处内存复制到另一个特定区域，这种技术通常用于通过网络传输文件时节省 CPU 周期和内存带宽。
- 那么对于零拷贝而言，并非真的是 \_\_\_\_\_\_\_\_ ，只不过是减少 \_\_\_\_\_\_\_\_ 以及 \_\_\_\_\_\_\_\_ 。

## \_\_\_\_\_\_\_\_ + \_\_\_\_\_\_\_\_

- 简单来说就是使用 \_\_\_\_\_\_\_\_ 替换了 \_\_\_\_\_\_\_\_ 中的 read 操作，减少了一次 CPU 的拷贝。
- Mmap 主要实现方式是将读缓冲区的地址和用户缓冲区的地址进行映射，内核缓冲区和应用缓冲区共享，从而减少了从读缓冲区到用户缓冲区的一次 CPU 拷贝。
- 节省了一次 CPU 拷贝，同时由于用户进程中的内存是虚拟的，只是映射到内核的读缓冲区，所以可以节省一半的内存空间，比较适合大文件的传输

整个过程发生了 \_\_\_\_\_\_\_\_ 次用户态和内核态的 \_\_\_\_\_\_\_\_ 和 \_\_\_\_\_\_\_\_ 次 \_\_\_\_\_\_\_\_

## \_\_\_\_\_\_\_\_

- 相比 mmap 来说，同样减少了一次 CPU 拷贝，而且还 \_\_\_\_\_\_\_\_ 。
- xxx 是 Linux 2.1 内核版本后引入的一个系统调用函数，通过使用 xxx 数据可以直接在内核空间进行传输，因此避免了用户空间和内核空间的拷贝，同时由于使用 xxx 替代了 read+write 从而节省了\_\_\_\_\_\_\_\_，也就是 \_\_\_\_\_\_\_\_ 。
- xxx 方法 IO 数据对\_\_\_\_\_\_\_\_完全不可见，所以只能适用于完全不需要用户空间处理的情况，比如静态文件服务器。

整个过程发生了 \_\_\_\_\_\_\_\_ 次用户态和内核态的 \_\_\_\_\_\_\_\_ 和 \_\_\_\_\_\_\_\_ 次 \_\_\_\_\_\_\_\_

## \_\_\_\_\_\_\_\_ + \_\_\_\_\_\_\_\_

- Linux 2.4 内核版本之后对 sendfile 做了进一步优化，通过引入新的硬件支持。
- 它将读缓冲区中的数据描述信息--内存地址和偏移量记录到 socket 缓冲区，由 DMA 根据这些将数据从读缓冲区拷贝到网卡，相比之前版本减少了一次 CPU 拷贝的过程
- \_\_\_\_\_\_\_\_ 和 sendfile 一样数据对用户空间不可见，而且需要硬件支持，同时输入文件描述符只能是 \_\_\_\_\_\_\_\_ ，但是过程中完全没有 \_\_\_\_\_\_\_\_ 过程，极大提升了性能。

整个过程发生了 \_\_\_\_\_\_\_\_ 次用户态和内核态的 \_\_\_\_\_\_\_\_ 和 \_\_\_\_\_\_\_\_ 次 \_\_\_\_\_\_\_\_