原文地址：[JVM Anatomy Park #18: Scalar Replacement](https://shipilev.net/jvm-anatomy-park/18-scalar-replacement/)

## 问题

我听说 Hotspot 可以做栈上分配。被称为逃逸分析（Escape Analysis），这很神奇。对么？

## 理论

这种表述有点儿混乱。在“栈上分配”这种表述中，“分配”的意思好像是将整个对象分配在栈上，而不是堆里。但是真正发生的是，*编译器*执行所谓的[逃逸分析（EA）](https://en.wikipedia.org/wiki/Escape_analysis)，逃逸分析可以识别哪些新创建的对象不会逃逸到堆里，然后就可以做一些有趣的优化。注意 EA 本身*并不是*优化，它是为优化器提供重要数据的分析阶段。<sup>[1]</sup>

优化器可以为没有逃逸的对象做的其中一个事情是，将对对象字段的访问映射到对合成本地操作数的访问：<sup>[2]</sup>执行*标量替换*。因为这些操作数由寄存器分配器处理，所以某些操作数可以申请当前方法占用的栈槽（从寄存器“溢出”了），可能*看起来*像是对象字段块直接分配在栈上。但是这是一个不合适的比喻：操作数可能根本没有具象化，或者可能存在于寄存器中，对象头根本没有创建。从对象字段访问映射而来的操作数在栈上可能都不是连续的！这与栈分配是不同的。

如果真的执行了栈分配，那么整个对象都会分配在栈上，包括头部和字段，并且会在生成的代码中引用对象。这个方案的风险是，一旦对象*逃逸*了，我们需要复制栈上的整个对象块到堆里，因为我们无法保证当前线程一直待在这个方法里面，使得这部分栈空间一直持有这个对象。这意味着我们需要拦截对堆内存的*写操作*，以防我们曾经写栈分配的对象 —— 也就说，执行 GC 写屏障。

Hotspot 自身不做栈分配，但是他通过标量替换实现了类似的功能。

我们可以在实践中观察么？

## 实践

考虑这个 JMH 测试用例。创建一个包含一个字段的对象，字段通过构造参数初始化，然后读取字段，丢弃这个对象：

```
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class ScalarReplacement {

    int x;

    @Benchmark
    public int single() {
        MyObject o = new MyObject(x);
        return o.x;
    }

    static class MyObject {
        final int x;
        public MyObject(int x) {
            this.x = x;
        }
    }

}
```

如果你设置 `-prof gc` 执行这个测试用例，那么你将会发现测试没有分配任何对象：

```
Benchmark                                      Mode  Cnt     Score    Error   Units

ScalarReplacement.single                       avgt   15     1.919 ±  0.002   ns/op
ScalarReplacement.single:·gc.alloc.rate        avgt   15    ≈ 10⁻⁴           MB/sec
ScalarReplacement.single:·gc.alloc.rate.norm   avgt   15    ≈ 10⁻⁶             B/op
ScalarReplacement.single:·gc.count             avgt   15       ≈ 0           counts
```

`-prof perfasm` 显示仅仅访问了字段 `x` 一次。

```
....[Hottest Region 1].............................................................
C2, level 4, org.openjdk.ScalarReplacement::single, version 459 (26 bytes)

                  [Verified Entry Point]
  6.05%    2.82%    0x00007f79e1202900: sub    $0x18,%rsp          ; prolog
  0.95%    0.78%    0x00007f79e1202907: mov    %rbp,0x10(%rsp)
  0.04%    0.21%    0x00007f79e120290c: mov    0xc(%rsi),%eax      ; get field $x
  5.80%    7.43%    0x00007f79e120290f: add    $0x10,%rsp          ; epilog
                    0x00007f79e1202913: pop    %rbp
 23.91%   33.34%    0x00007f79e1202914: test   %eax,0x17f0b6e6(%rip)
  0.21%    0.02%    0x00007f79e120291a: retq
...................................................................................
```

注意其中的魔法：编译器发现 `MyObject` 实例没有逃逸，所以重新映射它的字段到局部操作数，然后（请击鼓）识别到操作数的写操作之后跟着的是加载操作，所以可以完全消除 store-load 操作 —— 就像处理局部变量变量一样！然后，修剪掉分配流程，因为再也不需要分配了，任何对象的回忆都消失了。

当然，这需要复杂的 EA 实现来识别没有逃逸的候选者。当 EA 出问题时，标量替换也会出问题。当前 Hotspot EA 中最常见的问题发生在访问之前的控制流合并。例如，如果我们有两个不同的对象（但是内容是一样的），在不同的选择分支中，EA 就会出问题，即使两个对象明显（对我们人类来说）没有逃逸：

```
public class ScalarReplacement {

    int x;
    boolean flag;

    @Setup(Level.Iteration)
    public void shake() {
        flag = ThreadLocalRandom.current().nextBoolean();
    }

    @Benchmark
    public int split() {
        MyObject o;
        if (flag) {
            o = new MyObject(x);
        } else {
            o = new MyObject(x);
        }
        return o.x;
    }

   // ...
}
```

这是分配情况：

```
Benchmark                                      Mode  Cnt     Score    Error   Units

ScalarReplacement.single                       avgt   15     1.919 ±  0.002   ns/op
ScalarReplacement.single:·gc.alloc.rate        avgt   15    ≈ 10⁻⁴           MB/sec
ScalarReplacement.single:·gc.alloc.rate.norm   avgt   15    ≈ 10⁻⁶             B/op
ScalarReplacement.single:·gc.count             avgt   15       ≈ 0           counts

ScalarReplacement.split                        avgt   15     3.781 ±  0.116   ns/op
ScalarReplacement.split:·gc.alloc.rate         avgt   15  2691.543 ± 81.183  MB/sec
ScalarReplacement.split:·gc.alloc.rate.norm    avgt   15    16.000 ±  0.001    B/op
ScalarReplacement.split:·gc.count              avgt   15  1460.000           counts
ScalarReplacement.split:·gc.time               avgt   15   929.000               ms
```

如果那是“真正的”栈分配，那么很容易处理这种情况：这将在运行时扩展栈空间，分配，访问，然后再离开方法前清除栈内容，栈分配将会收回。但是保护对象逃逸的写屏障的复杂性仍然存在。

## 观察

逃逸分析是支撑有趣的优化的一项有趣的编译器技术。标量替换是其中一种优化技术，它并不是将对象存储在栈上。相反它拆解对象，重写局部访问的代码，并且进一步优化，当寄存器压力比较高时，这些访问有时候会溢出到栈上。在很多情况下，它可以成功并且高效优化关键热点路径。

但是 EA 并不理想：如果我们不能静态地判定对象没有逃逸，那么必须假设对象会逃逸。复杂的控制流可能会提前退出。调用非内联 —— 因此对当前的分析不透明 —— 实例方法将会退出。依赖对象身份的操作也会退出，虽然与没有逃逸的对象进行引用对比这样简单的事情就可以有效的解决问题。

这不是一个理想的优化，但是当它起作用时，确实非常有效。编译器技术的进一步改善将会扩大 EA 有效的场景。<sup>[3]</sup>

* * *

[1] 当人们声称 EA 做了这些事情时，我有点儿恼火：它没有做，是优化器做的！

[2] 就像局部变量的中间表示拥有的，以及其它临时操作数编译器想要拥有的

[3] 例如，Graal 中著名的部分逃逸分析（Partial Escape Analysis），在复杂数据流场景下更有弹性。
