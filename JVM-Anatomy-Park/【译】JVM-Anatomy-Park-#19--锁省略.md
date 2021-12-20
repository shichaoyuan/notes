原文地址：[JVM Anatomy Park #19: Lock Elision](https://shipilev.net/jvm-anatomy-park/19-lock-elision/)

## Question

我听说 JVM 放弃了所有关于锁的编译器优化，所以如果我写了 `synchronized`，那么 JVM 必须做同步操作！是这样么？

## 理论

在当前的 Java 内存模型（Java Memory Model）中，没有用的锁[不保证有任何内存影响](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/#wishful-unobserved-sync)。另外，这意味着对非共享对象的同步是无效的，因此运行时系统也不会做额外的操作。这仍然是可能的，但是并非必须的，并且这为优化提供了机会。

因此，如果逃逸分析识别出对象没有逃逸，那么编译器可以自由决定消除同步。这在实践中可以观察到么？

## 实践

考虑这个简单的 JMH 测试用例。我们在新对象上使用同步或者不使用同步：

```
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class LockElision {

    int x;

    @Benchmark
    public void baseline() {
        x++;
    }

    @Benchmark
    public void locked() {
        synchronized (new Object()) {
            x++;
        }
    }
}
```

我们启动 `-prof perfnorm` 分析器执行测试，结果如下：

```
Benchmark                                   Mode  Cnt   Score    Error  Units

LockElision.baseline                        avgt   15   0.268 ±  0.001  ns/op
LockElision.baseline:CPI                    avgt    3   0.200 ±  0.009   #/op
LockElision.baseline:L1-dcache-loads        avgt    3   2.035 ±  0.101   #/op
LockElision.baseline:L1-dcache-stores       avgt    3  ≈ 10⁻³            #/op
LockElision.baseline:branches               avgt    3   1.016 ±  0.046   #/op
LockElision.baseline:cycles                 avgt    3   1.017 ±  0.024   #/op
LockElision.baseline:instructions           avgt    3   5.076 ±  0.346   #/op

LockElision.locked                          avgt   15   0.268 ±  0.001  ns/op
LockElision.locked:CPI                      avgt    3   0.200 ±  0.005   #/op
LockElision.locked:L1-dcache-loads          avgt    3   2.024 ±  0.237   #/op
LockElision.locked:L1-dcache-stores         avgt    3  ≈ 10⁻³            #/op
LockElision.locked:branches                 avgt    3   1.014 ±  0.047   #/op
LockElision.locked:cycles                   avgt    3   1.015 ±  0.012   #/op
LockElision.locked:instructions             avgt    3   5.062 ±  0.154   #/op
```

哇，测试结果**完全**相同：耗时相同，加载、存储、周期、指令的数量都相同。有很大的概率，这意味着两者生成的指令是一样的。确实是这样：

```
14.50%   16.97%    incl   0xc(%r8)              ; increment field
76.82%   76.05%  │  movzbl 0x94(%r9),%r10d       ; JMH infra: do another @Benchmark
 0.83%    0.10%  │  add    $0x1,%rbp
 0.47%    0.78%  │  test   %eax,0x15ec6bba(%rip)
 0.47%    0.36%  │  test   %r10d,%r10d
                 ╰  je     BACK
```

这个锁被完全*省略*了，没有分配，没有同步，什么都没有。如果我们设置 JVM 参数 `-XX:-EliminateLocks`，或者我们通过 `-XX:-DoEscapeAnalysis` 关闭 EA（这将会影响所有依赖 EA 的优化，包括锁省略），那么 `locked` 计数器的耗时将会激增。

```
Benchmark                                   Mode  Cnt   Score    Error  Units

LockElision.baseline                        avgt   15   0.268 ±  0.001  ns/op
LockElision.baseline:CPI                    avgt    3   0.200 ±  0.001   #/op
LockElision.baseline:L1-dcache-loads        avgt    3   2.029 ±  0.082   #/op
LockElision.baseline:L1-dcache-stores       avgt    3   0.001 ±  0.001   #/op
LockElision.baseline:branches               avgt    3   1.016 ±  0.028   #/op
LockElision.baseline:cycles                 avgt    3   1.015 ±  0.014   #/op
LockElision.baseline:instructions           avgt    3   5.078 ±  0.097   #/op

LockElision.locked                          avgt   15  11.590 ±  0.009  ns/op
LockElision.locked:CPI                      avgt    3   0.998 ±  0.208   #/op
LockElision.locked:L1-dcache-loads          avgt    3  11.872 ±  0.686   #/op
LockElision.locked:L1-dcache-stores         avgt    3   5.024 ±  1.019   #/op
LockElision.locked:branches                 avgt    3   9.027 ±  1.840   #/op
LockElision.locked:cycles                   avgt    3  44.236 ±  3.364   #/op
LockElision.locked:instructions             avgt    3  44.307 ±  9.954   #/op
```

结果展示了分配的成本，以及没有什么价值的同步。

## 观察

锁省略是逃逸分析支撑的另一个优化，它移除了一些多余的同步。当内部同步实现没有逃逸的时候，性能改善尤其明显：然后，我们可以完全免除同步！这是编译器优化的禅 —— 如果看不到同步锁，那么它存在么？
