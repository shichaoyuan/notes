原文地址：[JVM Anatomy Park #22: Safepoint Polls](https://shipilev.net/jvm-anatomy-park/22-safepoint-polls/)

## 问题

* JVM 如何停止 Java 线程以实现 stop-the-world？
* [在热循环中奇怪的 `test` 指令是什么？](https://stackoverflow.com/a/54055300/2613885)
* [为什么在 Java 11 中空方法执行变慢了？](https://stackoverflow.com/a/54010845/2613885)

所有这些问题的答案是同一个。

## 理论

像 JVM 这种托管运行时系统，有时需要停止 Java 线程，执行一些运行时代码。比如，执行 stop-the-world GC。可以等待所有线程最终调用 JVM，比如申请内存（经常执行 [TLAB](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/) 替换），或者类似的操作。但是这不一定会发生！如果线程正在执行某种频繁的循环逻辑而不做别的事情怎么办？

在大部分机器上停止运行的线程实际上是很简单的：向线程发送一个信号，强制处理器中断，等。停止线程正在执行的操作，将控制权转交给别处。然而，这还不足以让 Java 线程在任意位置停止，特别是如果你需要精确的垃圾回收。在这种情况下，你需要知道寄存器和栈中的内容，这些内容可能是你需要处理的对象引用。或者如果你想要取消偏向锁，你*需要*精确的知道线程的状态和获取的锁。或者如果你想要逆优化方法，你需要在不丢失执行状态和临时值的安全位置操作。

因此像 Hotspot 这种现代 JVMs 实现了协作机制：线程经常询问是否应该将控制权交给 VM，在线程生命周期中某些已知的位置，线程的状态是已知的。当所有线程都在已知的位置停止的时候，VM 被认为是到达了*安全点*。检查安全点请求的代码片段因此被称为*安全点检查（safepoint polls）*

这种实现需要满足以下有趣的权衡：安全点检查很少触发进程停止，所以在不触发时应该非常高效。我们可以通过实验观测么？

## 实践

考虑这个简单的 JMH 测试用例：

```
import org.openjdk.jmh.annotations.*;

import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class EmptyBench {
    @Benchmark
    public void emptyMethod() {
        // This method is intentionally left blank.
    }
}
```

你可能*认为*这个测试用例测量的是空方法，但是实际上这测量的是执行测试用例最小的基础设施代码：统计迭代，等待迭代执行时间。这段代码执行的非常快，所以可以使用 `-prof perfasm` 剖析执行过程。

这是原装 OpenJDK 8u191 的执行结果：

```
3.60%    ...a2: movzbl 0x94(%r8),%r10d       ; load "isDone" field
0.63%  │  ...aa: add    $0x1,%rbp             ; iterations++;
32.82% │  ...ae: test   %eax,0x1765654c(%rip) ; global safepoint poll
58.14% │  ...b4: test   %r10d,%r10d           ; if !isDone, do the cycle again
       ╰  ...b7: je     ...a2
```

这个空方法被内联了，所有的调用成本都消除了，仅仅剩下基础设施逻辑。

看到 “global safepoint poll” 了么？当需要检查点的时候，JVM 将会持有“检查页（polling page）”<sup>[1]</sup>，任何对该页的读操作将会触发 [段错误 (SEGV)](https://en.wikipedia.org/wiki/Segmentation_fault)。当安全点检查最终触发 SEGV 时，控制权将会传递给任一存在的 SEGV 处理器，JVM 已经准备好了一个！可以看一下 `JVM_handle_linux_signal` 如何[处理](http://hg.openjdk.java.net/jdk/jdk/file/af7afdababd3/src/hotspot/os_cpu/linux_x86/os_linux_x86.cpp#l430)[段错误](http://hg.openjdk.java.net/jdk/jdk/file/af7afdababd3/src/hotspot/os_cpu/linux_x86/os_linux_x86.cpp#l577)。

所有这些技巧的目的是使得安全点的成本尽可能低，因为很多位置都需要安全点，但是几乎**不会**触发。基于这个原因，使用 `test %eax, (addr)` 指令：当安全点检查没有触发的时候，这条指令没有作用<sup>[2]</sup>。这条指令的编码也很紧凑，在 x86_64 平台仅仅占用6字节。对于给定 JVM 进程，检查的页地址是固定的，所以 JIT 生成的代码可以使用[相对 RIP 寻址（RIP-relative addressing）](https://en.wikipedia.org/wiki/Addressing_mode#PC-relative_2)：从当前指令指针给定页地址的偏移量，而不需要耗费空间编码8字节绝对地址。

通常来说只有一个检查页来处理所有线程，所以生成的代码不需要辨别当前执行的线程。但是如果 VM 想要停止单个线程，如何操作？[JEP-312: "Thread-Local Handshakes"](https://openjdk.java.net/jeps/312) 给出了这个问题的答案。为 VM 提供了对单个线程触发*握手（handshake）*检查的能力，当前是通过为*每个线程*分配单独的检查页实现的，检查指令读取线程局部存储（thread-local storage）中的页地址。<sup>[3]</sup><sup>[4]</sup>

这是原装 OpenJDK 11.0.1 的执行结果：

```
0.31%    ...70: movzbl 0x94(%r9),%r10d   ; load "isDone" field
0.19%  │  ...78: mov    0x108(%r15),%r11  ; reading the thread-local poll page addr
25.62% │  ...7f: add    $0x1,%rbp         ; iterations++;
35.10% │  ...83: test   %eax,(%r11)       ; thread-local handshake poll
34.91% │  ...86: test   %r10d,%r10d       ; if !isDone, do the cycle again
       ╰  ...89: je     ...70
```

这纯粹是运行时的问题，所以可以通过 `-XX:-ThreadLocalHandshakes` 关闭这个特性，生成的代码将会与 8u191 中的一样。这解释了 8 与 11 测试结果不同的原因（让我们马上用 `-prof perfnorm` 执行测试用例）：


```
Benchmark                              Mode  Cnt  Score   Error  Units

# 8u191
EmptyBench.test                        avgt   15   0.383 ±  0.007  ns/op
EmptyBench.test:CPI                    avgt    3   0.203 ±  0.014   #/op
EmptyBench.test:L1-dcache-load-misses  avgt    3  ≈ 10⁻⁴            #/op
EmptyBench.test:L1-dcache-loads        avgt    3   2.009 ±  0.291   #/op
EmptyBench.test:cycles                 avgt    3   1.021 ±  0.193   #/op
EmptyBench.test:instructions           avgt    3   5.024 ±  0.229   #/op

# 11.0.1
EmptyBench.test                        avgt   15   0.590 ±  0.023  ns/op ; +0.2 ns
EmptyBench.test:CPI                    avgt    3   0.260 ±  0.173   #/op
EmptyBench.test:L1-dcache-loads        avgt    3   3.015 ±  0.120   #/op ; +1 load
EmptyBench.test:L1-dcache-load-misses  avgt    3  ≈ 10⁻⁴            #/op
EmptyBench.test:cycles                 avgt    3   1.570 ±  0.248   #/op ; +0.5 cycles
EmptyBench.test:instructions           avgt    3   6.032 ±  0.197   #/op ; +1 instruction

# 11.0.1, -XX:-ThreadLocalHandshakes
EmptyBench.test                        avgt   15   0.385 ±  0.007  ns/op
EmptyBench.test:CPI                    avgt    3   0.205 ±  0.027   #/op
EmptyBench.test:L1-dcache-loads        avgt    3   2.012 ±  0.122   #/op
EmptyBench.test:L1-dcache-load-misses  avgt    3  ≈ 10⁻⁴            #/op
EmptyBench.test:cycles                 avgt    3   1.030 ±  0.079   #/op
EmptyBench.test:instructions           avgt    3   5.031 ±  0.299   #/op
```

所以线程局部握手增加了一次额外的 L1 命中加载，这耗费了大约半个周期。这也为我们评估安全点检查的成本提供了一些基准：L1 命中加载，大概额外耗费半个周期。

## 观察

安全点和握手检查是托管运行时系统实现中的小细节。它们经常出现在生成代码的热路径中，有时候会影响性能，特别是在密集的循环中。然而，对于运行时系统实现诸如垃圾回收、锁优化、逆优化等重要的特性，这些操作是必要的。

有许多安全点相关的优化，我们将会单独讨论。


* * *
1.  在 Linux/POSIX 中，[调用](http://hg.openjdk.java.net/jdk/jdk/file/af7afdababd3/src/hotspot/os/linux/os_linux.cpp#l3336) `mprotect(PROT_NONE)` 就[足够](http://hg.openjdk.java.net/jdk/jdk/file/af7afdababd3/src/hotspot/os/linux/os_linux.cpp#l5074)了。
2. 差不多吧。在 x86 中，这条指令改变标志位，但是下一条指令将会重写，我们只需要注意别在 `test` 和相关的 `jCC` 之间再做安全点检查。
3. 线程局部存储是每个线程可以访问的一段本地数据。在许多平台中，寄存器的压力不是很高，生成的代码总是将线程局部存储放在寄存器中。在 x86_64 中，存储位置[通常是](http://hg.openjdk.java.net/jdk/jdk/file/af7afdababd3/src/hotspot/cpu/x86/x86_64.ad#l12878) `%r15`。
4. 从技术上讲，停止一部分线程不是一个“安全点”。但是当线程局部握手（thread-local handshakes）启动的时候，可以通过握手所有线程实现安全点。对于“安全点”和“握手”的场景都支持。
