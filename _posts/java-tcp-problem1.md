---
title: 总结的一个关于java相关的文件传输讨论
date: 2016-07-29 17:21:43
tags:
 - nio
categories:
 - Java
---

论坛上看到的一场关于java在特定网络环境下大文件传输的讨论，自己之前对这方面的认识也很模糊，在这里特别记录一下。

现象：从机器A向机器B传输一个大文件，网络带宽足够，有百兆，但是网络延时较高，ping值大约为100ms。java bio单线程不能达到100k，采用java nio的话，则基本能打满。

原因分析：首先来看传输大文件这个流程，java socket算是应用层的，然后发送tcp包。先看bio传输文件的做法，一般是启一个socket，BufferedInputStream读文件到byte[] buffer里，socket.getOutputStream().write(buffer)，最终是调用natvie方法

```java
private native void socketWrite0(FileDescriptor fd, byte[] b, int off,
                                     int len) throws IOException;
```

由于BufferedInputStream默认buffer大小是8192，所以一次往socket里也是写8192，socket写并不是直接生成tcp包，而是发到tcp发送端的send buffer里，只要发送到send buffer里，socket的write就立刻返回，send buffer有一个默认初始大小，但是在较新的linux版本里，操作系统支持send buffer的动态调整(send buffer大小和socket write的buffer大小一致)，当发出一个tcp包之后，包数据仍然存在于send buffer里直到接收到相应的ack，这时send buffer就能清出空间。ack信息同时包含了接收方tcp receive buffer的可用空间大小，如果对方的receive buffer可用空间放不下一个tcp了，收到这样的ack信息，发送方就会停止发送tcp包，并定时发送试探对方的receive buffer是否腾出空间的包。如果send buffer满了，那么应用层的send就会block住。由于socket的写长期是8k，因此发送方的send buffer也是长期8k大小，8k*10=80k/s。

来看java nio的实现。一般是FileInputStream.getChannel().transferTo(long position, long count, socketChannel)，这个在fileChannelImpl(代码可以在openjdk中看)的最终实现是走

```java
// Maximum size to map when using a mapped buffer
465     private static final long MAPPED_TRANSFER_SIZE = 8L*1024L*1024L;
466 
467     private long transferToTrustedChannel(long position, long count,
468                                           WritableByteChannel target)
```

这个是在jvm之外的弄出一块native内存给所有线程共享，可以看出最大可有8m，相当于有了一个大小为8m的buffer，每次socket.write的buffer是8m，tcp send buffer的大小也动态调整为8m，最大可发送的速率为8m*10=80m/s 因此可以打满带宽。