---
title: 应用迁移至k8s后文件句柄暴增定位
date: 2018-11-14 16:34:34
tags: 
- 故障排查
categories:
- 故障排查
comments: true
---


## 背景

运维发现部署在k8s环境上的应用A比在stable环境中的句柄数要多十倍. 

## 排查过程

### 初步分析

通过lsof 命令看了下进程打开的文件句柄，发现其中**pipe**出现了很多次，大概1300次, 而相对应的stable环境中的应用A打开的pipe为130次

问题:

为什么会有这么多pipe,是谁创建的，为什么要创建?

我并不是很熟悉java中涉及pipe的场景，所以根本就不知道谁会创建pipe，只能先找到所有会使用pipe的地方，然后在进一步分析.

**如何找到所有使用pipe的地方？**

想想貌似只能拦截下pipe的调用，一旦发现有进程调pipe，就记录下调用栈，通过这样的方式应该就可以拿到一些pipe的使用场景了.

由于不知道是哪些java的api会调用pipe操作，在java层面进行拦截是不可能的了，只能在系统层进行尝试了.

鉴于pipe是一个system call, 可以考虑通过strace -e trace=pipe 进行跟踪。

不过由于strace仅仅是负责跟踪，在我们拿到相应进程号之后再去查看调用栈的时候可能已经太迟了。 所以该方法没啥用.

### 可能的思路

能够拦截特定调用，在拦截到后又能够执行特定action的工具，听说过的有以下几种:

1. dtrace 

    不熟悉,看了下文档用起来很麻烦的样子

2. 使用linux LD_PRELOAD机制增强pipe调用. 

    只是个思路，或许可行

3. gdb 

    相对熟悉一些

### gdb的排查步骤

1. 使用gdb启动应用

    ```

    gdb --args /usr/enniu/java/bin/java /var/www/xxx/target/XXXX.jar

    ```

2. 拦截pipe的断点

    ```
    (gdb) catch syscall pipe

    ```

3. 忽略SIGSEGV信号

    ```
    (gdb) handle SIGSEGV nostop noprint pass
    ```
    为什么要做这个?看这个 : <https://neugens.wordpress.com/2015/02/26/debugging-the-jdk-with-gdb/>

4. 执行程序
    
    ```
    (gdb) run
    当java程序调用pipe时，程序就会暂停下来，等待用户指令.

    ```

5. 等待捕获pipe调用事件

6. 捕获pipe调用事件

    ```
    Catchpoint 1 (returned from syscall 'pipe'), 0x00007fd22237ff07 in pipe () at ../sysdeps/unix/syscall-template.S:82
    82	T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)

    ```

7. 执行info thread 查看下当前的线程如下:

    ```

    3 Thread 0x7fd2203b7700 (LWP 1630)  pthread_cond_wait@@GLIBC_2.3.2 () at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:183
    * 2 Thread 0x7fd222e81700 (LWP 1605)  0x00007fd22237ff07 in pipe () at ../sysdeps/unix/syscall-template.S:82
    1 Thread 0x7fd222e83700 (LWP 1522)  0x00007fd222a5a2fd in pthread_join (threadid=140540505495296, thread_return=0x7ffeaef13e00) at pthread_join.c:89

    ```

    可以看到线程2是活的。记住1605这个进程号.

8. 执行gcore dump下当前的内存镜像

    ```
    (gdb) gcore
    Saved corefile core.1522
    ```
    为啥要gcore ,因为在默认的gdb环境下，看不到java的调用栈.

9. 使用 jstack java core.1522 打印线程栈.

    ```
    Thread 1605: (state = IN_NATIVE)
    - sun.nio.ch.IOUtil.makePipe(boolean) @bci=0 (Interpreted frame)
    - sun.nio.ch.EPollSelectorImpl.<init>(java.nio.channels.spi.SelectorProvider) @bci=27, line=65 (Interpreted frame)
    - sun.nio.ch.EPollSelectorProvider.openSelector() @bci=5, line=36 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoop.openSelector() @bci=4, line=174 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoop.<init>(io.netty.channel.nio.NioEventLoopGroup, java.util.concurrent.ThreadFactory, java.nio.channels.spi.SelectorProvider, io.netty.channel.SelectStrategy, io.netty.util.concurrent.RejectedExecutionHandler) @bci=88, line=150 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoopGroup.newChild(java.util.concurrent.ThreadFactory, java.lang.Object[]) @bci=29, line=103 (Interpreted frame)
    - io.netty.util.concurrent.MultithreadEventExecutorGroup.<init>(int, java.util.concurrent.ThreadFactory, java.lang.Object[]) @bci=146, line=64 (Interpreted frame)
    - io.netty.channel.MultithreadEventLoopGroup.<init>(int, java.util.concurrent.ThreadFactory, java.lang.Object[]) @bci=14, line=50 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoopGroup.<init>(int, java.util.concurrent.ThreadFactory, java.nio.channels.spi.SelectorProvider, io.netty.channel.SelectStrategyFactory) @bci=22, line=70 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoopGroup.<init>(int, java.util.concurrent.ThreadFactory, java.nio.channels.spi.SelectorProvider) @bci=7, line=65 (Interpreted frame)
    - io.netty.channel.nio.NioEventLoopGroup.<init>(int, java.util.concurrent.ThreadFactory) @bci=6, line=56 (Interpreted frame)
    - org.asynchttpclient.netty.channel.ChannelManager.<init>(org.asynchttpclient.AsyncHttpClientConfig, io.netty.util.Timer) @bci=394, line=173 (Interpreted frame)
    - org.asynchttpclient.DefaultAsyncHttpClient.<init>(org.asynchttpclient.AsyncHttpClientConfig) @bci=73, line=85 (Interpreted frame)

    ```

    可以看到1605线程的调用栈如上. （此时看到了java层面调用pipe的地方，也许只是java中唯一与pipe发生交互的地方，也许不是,在采集几个样本看看...).

10. 执行cont,不一会又会捕捉到pipe调用事件, 重复8,9 步几次. 

    可以观察到几乎所有的pipe操作都是sun.nio.ch.IOUtil.makePipe 触发的，而上游都由Netty的NioEventLoopGroup触发.

### NioEventLoopGroup代码分析

翻了下NioEventLoopGroup的代码,有如下一段:

```

MultithreadEventLoopGroup.java

protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
        ...
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(threadFactory, args);// 这里每次调用创建一个pipe。 如果循环很大的话，是可能创建很多pipe的.
                success = true;
            }
        }
        ...
} 

```

查看下nThreads的赋值逻辑如下:

如果调用方有指定，则使用指定值，否则为cpu数*2. 代码如下

```
MultithreadEventLoopGroup.java

static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}

```

## 结论

从代码的分析结果来看，怀疑是k8s环境中cpu核数过多导致，查看k8s环境中应用所在容器的cpu数，为20是stable环境的10倍。

至此, stable环境中为130，k8s为1300的原因就找到啦。


## 遗留问题 （先占坑）

1. 为什么用epoll的时候会出现触发pipe的创建?

