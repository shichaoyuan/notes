原文地址：[JVM Anatomy Park #7: Initialization Costs](https://shipilev.net/jvm-anatomy-park/7-initialization-costs/)

## 问题

为什么新建对象的成本如此之高？影响对象实例化性能的主要因素是什么？

## 理论

如果你仔细观察大对象的实例化过程，你必然想要知道不同阶段的耗时情况，整个过程中真正的瓶颈是什么。我们已经知道[TLAB 分配很高效](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/)，[系统初始化与用户初始化可以合并](https://shipilev.net/jvm-anatomy-park/6-new-object-stages/)。但是最终还得写入内存呀 —— 我们可以测量写入内存的成本么？

## 实验


Java 数组很适合用来测试初始化。因为数组需要初始化，而且长度是可变的，这样就可以测试不同长度的情况。认识到这一点，让我们构建测试用例吧：

```
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class UA {
    @Param({"1", "10", "100", "1000", "10000", "100000"})
    int size;

    @Benchmark
    public byte[] java() {
        return new byte[size];
    }
}
```

我们用最新的 JDK 9 EA 执行，设置参数 `-XX:+UseParallelOldGC` 最小化 GC 的开销，设置参数 `-Xmx20g -Xms20g -Xmn18g` 保证大部分堆内存用于新的分配。闲话少说，让我们在一台性能良好的开发台式机（比如 i7-4790K, 4.0 GHz, Linux x86_64）上执行测试用例，当所有 8 个硬件线程跑满的时候，结果如下：

```
Benchmark                  (size)  Mode  Cnt       Score       Error   Units

# Time to allocate
UA.java                         1  avgt   15      20.307 ±     4.532   ns/op
UA.java                        10  avgt   15      26.657 ±     6.072   ns/op
UA.java                       100  avgt   15     106.632 ±    34.742   ns/op
UA.java                      1000  avgt   15     681.176 ±   124.980   ns/op
UA.java                     10000  avgt   15    4576.433 ±   909.956   ns/op
UA.java                    100000  avgt   15   44881.095 ± 13765.440   ns/op

# Allocation rate
UA.java:·gc.alloc.rate          1  avgt   15    6228.153 ±  1059.385  MB/sec
UA.java:·gc.alloc.rate         10  avgt   15    6335.809 ±   986.395  MB/sec
UA.java:·gc.alloc.rate        100  avgt   15    6126.333 ±  1354.964  MB/sec
UA.java:·gc.alloc.rate       1000  avgt   15    7772.263 ±  1263.453  MB/sec
UA.java:·gc.alloc.rate      10000  avgt   15   11518.422 ±  2155.516  MB/sec
UA.java:·gc.alloc.rate     100000  avgt   15   12039.594 ±  2724.242  MB/sec
```

长度为1时，分配过程大约耗时 20 ns（单线程的情况会比较低，但是这是超线程下的情况），随着数组长度逐步增加到 100K，耗时也逐步增加到 40 us。再看一下分配速率，最终稳定在大约 12 GB/sec。顺便说一下，这个测试为其它性能测试提供了基准：理解特定机器的内存带宽和分配速率量级是非常重要。

我们可以找到影响执行耗时的代码么？当然可以，再次使用 JMH 的 `-prof perfasm`。对于 `-p size=100000`，将会打印出下述最热的指令：

```
              0x00007f1f094f650b: movq   $0x1,(%rdx)              ; store mark word
  0.00%       0x00007f1f094f6512: prefetchnta 0xc0(%r9)
  0.64%       0x00007f1f094f651a: movl   $0xf80000f5,0x8(%rdx)    ; store klass word
  0.02%       0x00007f1f094f6521: mov    %r11d,0xc(%rdx)          ; store array length
              0x00007f1f094f6525: prefetchnta 0x100(%r9)
  0.05%       0x00007f1f094f652d: prefetchnta 0x140(%r9)
  0.07%       0x00007f1f094f6535: prefetchnta 0x180(%r9)
  0.09%       0x00007f1f094f653d: shr    $0x3,%rcx
  0.00%       0x00007f1f094f6541: add    $0xfffffffffffffffe,%rcx
              0x00007f1f094f6545: xor    %rax,%rax
              0x00007f1f094f6548: cmp    $0x8,%rcx
        ╭     0x00007f1f094f654c: jg     0x00007f1f094f655e       ; large enough? jump
        │     0x00007f1f094f654e: dec    %rcx
        │╭    0x00007f1f094f6551: js     0x00007f1f094f6565       ; zero length? jump
        ││↗   0x00007f1f094f6553: mov    %rax,(%rdi,%rcx,8)       ; small loop init
        │││   0x00007f1f094f6557: dec    %rcx
        ││╰   0x00007f1f094f655a: jge    0x00007f1f094f6553
        ││ ╭  0x00007f1f094f655c: jmp    0x00007f1f094f6565
        ↘│ │  0x00007f1f094f655e: shl    $0x3,%rcx
 89.12%  │ │  0x00007f1f094f6562: rep rex.W stos %al,%es:(%rdi)   ; large loop init
  0.20%  ↘ ↘  0x00007f1f094f6565: mov    %r8,(%rsp)
```

指令的结构与之前文章[《TLAB 分配》](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/) 和 [《新建对象的阶段》](https://shipilev.net/jvm-anatomy-park/6-new-object-stages/)中的类似。唯一的不同是这里需要初始化更大段的数组数据。因此我们可以看到内联的 x86 指令 `rep stos` —— 这条指令负责依次赋值  —— 在最近的 x86 平台上非常高效。对于比较小（小于等于 8 个元原始）的数组，这里是通过循环赋值 —— 因为 `rep stos` 指令有一些启动成本，所以对于小数组使用循环赋值更合适。

这个测试可以说明，对于大对象或数组，初始化赋值是影响性能的最主要的因素。对于小对象或数组，写入元数据（头部信息、数组长度）是影响性能的最主要的因素。小对象的场景与小数组的场景类似。

我们可以估算没有初始化过程的性能么？编译器可以合并系统和用户初始化，但是我们可以*关闭初始化*么？当然，获取未初始化的对象是没有实际意义的，因为稍后我们还得写入用户数据 —— 但是人为的测试是可以的，我说的对吧？

有一个 `Unsafe` 的方法可以分配未初始化的数组，所以我们可以用这个方法测试。`Unsafe` 不遵守 Java 的规则，甚至有时候会违反 JVM 的规则。这不是公开使用的 API，它仅供 JDK 内部使用，以及 JDK-VM 互操作性。通常来说你不应该使用，它不保证有效，它可能会毫无征兆的退出、崩溃。

然而我们可以在测试中使用，就像这样：

```
import jdk.internal.misc.Unsafe;
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class UA {
    static Unsafe U;

    static {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            U = (Unsafe) field.get(null);
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    @Param({"1", "10", "100", "1000", "10000", "100000"})
    int size;

    @Benchmark
    public byte[] unsafe() {
        return (byte[]) U.allocateUninitializedArray(byte.class, size);
    }
}
```

执行一下：

```
Benchmark                  (size)  Mode  Cnt        Score       Error   Units
UA.unsafe                       1  avgt   15       19.766 ±     4.002   ns/op
UA.unsafe                      10  avgt   15       27.486 ±     7.005   ns/op
UA.unsafe                     100  avgt   15       80.040 ±    15.754   ns/op
UA.unsafe                    1000  avgt   15      156.041 ±     0.552   ns/op
UA.unsafe                   10000  avgt   15      162.384 ±     1.448   ns/op
UA.unsafe                  100000  avgt   15      309.769 ±     2.819   ns/op

UA.unsafe:·gc.alloc.rate        1  avgt   15     6359.987 ±   928.472  MB/sec
UA.unsafe:·gc.alloc.rate       10  avgt   15     6193.103 ±  1160.353  MB/sec
UA.unsafe:·gc.alloc.rate      100  avgt   15     7855.147 ±  1313.314  MB/sec
UA.unsafe:·gc.alloc.rate     1000  avgt   15    33171.384 ±   153.645  MB/sec
UA.unsafe:·gc.alloc.rate    10000  avgt   15   315740.299 ±  3678.459  MB/sec
UA.unsafe:·gc.alloc.rate   100000  avgt   15  1650860.763 ± 14498.920  MB/sec
```

天哪！长度为 100K 的数组的分配速率达到了 1.6 **万亿字节每秒**。*现在*主要的耗时阶段是什么呢？

```
          0x00007f65fd722c74: prefetchnta 0xc0(%r11)
 66.06%   0x00007f65fd722c7c: movq   $0x1,(%rax)           ; store mark word
  0.40%   0x00007f65fd722c83: prefetchnta 0x100(%r11)
  4.43%   0x00007f65fd722c8b: movl   $0xf80000f5,0x8(%rax) ; store class word
  0.01%   0x00007f65fd722c92: mov    %edx,0xc(%rax)        ; store array length
          0x00007f65fd722c95: prefetchnta 0x140(%r11)
  5.18%   0x00007f65fd722c9d: prefetchnta 0x180(%r11)
  4.99%   0x00007f65fd722ca5: mov    %r8,0x40(%rsp)
          0x00007f65fd722caa: mov    %rax,%rdx
```


是的，写入元数据到内存。执行周期倾向于预取指令，因为它们为写操作之前的预访问内存承担了大部分成本。

你可能想知道 GC 有多大影响，答案是：不多！因为大部分对象都已经死了，所以任意标记-（复制|清除|整理）GC 都能轻轻松松完成工作。但是当对象以 TB/sec 的速率*存活*时，事情就有意思了。一些搞 GC 的家伙告诉我这是“不可能的” —— 因为现实中是不可能的，而且也不可能处理。让我们尝试从实践中学习。

不管怎样，像测试用例中纯分配的场景，GC 的表现还可以。使用相同的工作负载，我们通过 JMH 的 `-prof pauses` 分析器观察应用程序停顿。分析器运行一个高优先级的线程，记录感知到的停顿：

```
Benchmark                  (size)  Mode  Cnt    Score   Error  Units
UA.unsafe                  100000  avgt    5  315.732 ± 5.133  ns/op
UA.unsafe:·pauses          100000  avgt   84  537.018             ms
UA.unsafe:·pauses.avg      100000  avgt         6.393             ms
UA.unsafe:·pauses.count    100000  avgt        84.000              #
UA.unsafe:·pauses.p0.00    100000  avgt         2.560             ms
UA.unsafe:·pauses.p0.50    100000  avgt         6.148             ms
UA.unsafe:·pauses.p0.90    100000  avgt         9.642             ms
UA.unsafe:·pauses.p0.95    100000  avgt         9.802             ms
UA.unsafe:·pauses.p0.99    100000  avgt        14.418             ms
UA.unsafe:·pauses.p0.999   100000  avgt        14.418             ms
UA.unsafe:·pauses.p0.9999  100000  avgt        14.418             ms
UA.unsafe:·pauses.p1.00    100000  avgt        14.418             ms
```

分析器检测到了 84 次停顿，最长的一次停顿了 14 ms，而平均停顿大约 6 ms。这种分析器本来就是不精确的，因为它们与工作线程争用 CPU，分析结果依赖 OS 调度器等。

在很多情况下，最好让 JVM 记录停止应用线程的情况。这可以通过 JMH 的 `-prof safepoints` 分析器实现，它将会记录应用线程停止时的“安全点”、“万物静止”事件。GC 停顿是安全点事件的子集。

```
Benchmark                            (size)  Mode  Cnt     Score    Error  Units
UA.unsafe                            100000  avgt    5   328.247 ± 34.450  ns/op
UA.unsafe:·safepoints.interval       100000  avgt       5043.000              ms
UA.unsafe:·safepoints.pause          100000  avgt  639   617.796              ms
UA.unsafe:·safepoints.pause.avg      100000  avgt          0.967              ms
UA.unsafe:·safepoints.pause.count    100000  avgt        639.000               #
UA.unsafe:·safepoints.pause.p0.00    100000  avgt          0.433              ms
UA.unsafe:·safepoints.pause.p0.50    100000  avgt          0.627              ms
UA.unsafe:·safepoints.pause.p0.90    100000  avgt          2.150              ms
UA.unsafe:·safepoints.pause.p0.95    100000  avgt          2.241              ms
UA.unsafe:·safepoints.pause.p0.99    100000  avgt          2.979              ms
UA.unsafe:·safepoints.pause.p0.999   100000  avgt         12.599              ms
UA.unsafe:·safepoints.pause.p0.9999  100000  avgt         12.599              ms
UA.unsafe:·safepoints.pause.p1.00    100000  avgt         12.599              ms
```

该分析器记录了 639 个安全点，平均耗时小于 1ms，最长的一次是 12ms。还不错，考虑到 1.6 TB/sec 的分配速度！

## 观察

对象和数组实例化过程中最主要的成本是初始化。通过 TLAB 分配，对象和数组的创建速度或者受制于元数据写入（对于小对象），或者受制于数据初始化（对于大对象）。分配速率并不*总是*一个好的性能指标，因为通过一些小技巧可以实现非常高的分配速率。
