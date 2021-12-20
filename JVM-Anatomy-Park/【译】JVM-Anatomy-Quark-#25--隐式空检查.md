原文地址：[JVM Anatomy Quark #25: Implicit Null Checks](https://shipilev.net/jvm/anatomy-quarks/25-implicit-null-checks/)

## 问题

Java 规范上写着访问 `null` 对象字段时将会抛出 `NullPointerException`。这意味着 JVM 必须使用运行时检查对象是否为空？

## 理论

在理论上，(JIT)编译器可以确定某个对象不为 `null`，以此略去运行时空检查，例如对于[常量](https://shipilev.net/jvm/anatomy-quarks/15-just-in-time-constants/)来说：

```
static class Holder { int x; }
static final Holder H = new Holder();

int m() {
  return H.x; // H is known to be not null at JIT compilation time
}
```

如果这样还不行，例如无法自动推断是否为空，那么编译器也可以采用数据流分析来移除首次空检查之后的检查。例如：

```
int m(Holder h) {
  int x1 = h.x; // null-check here
  int x2 = h.x; // no need to null-check here again
  return x1 + x2;
}
```

这些优化非常有用，但是很无聊，并且不能解决其它情况下空检查的需求。

幸运的是，有一个更聪明的方法解决这个问题：**让用户代码在没有显式检查的情况下访问对象**！大部分情况下不会出现异常，因为大部分对象访问**不会是**空对象。但是我们仍然需要处理 `null` 访问的异常情况。当访问空对象时，JVM [可以拦截](http://hg.openjdk.java.net/jdk/jdk/file/b9d1ce20dd4b/src/hotspot/os_cpu/linux_x86/os_linux_x86.cpp#l486)生成的 SIGSEGV（信号：段错误），查看该信号返回的地址，识别出生成代码中的访问位置。一旦确定了访问位置，就可以知道在哪里调度控件来处理这种情况——在大部情况下就是抛出 `NullPointerException` 或者跳到另外的分支。

这种机制在 Hotspot 中称为 [*”隐式空检查“*](http://hg.openjdk.java.net/jdk/jdk/file/b9d1ce20dd4b/src/hotspot/share/runtime/globals.hpp#l1029)。该机制最近也以[类似的名称](https://llvm.org/docs/FaultMaps.html)添加到了 LLVM 中。

我们可以看一下它是如何工作的吗？

### 实践

请看这个巧妙而简单的 JMH 测试用例：

```
import org.openjdk.jmh.annotations.*;

import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3, jvmArgsAppend = {"-XX:LoopUnrollLimit=1"})
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class ImplicitNP {

    @Param({"false", "true"})
    boolean blowup;

    volatile Holder h;

    int itCnt;

    @Setup
    public void setup() {
        h = null;
        if (blowup && ++itCnt == 3) { // blow it up on 3-rd iteration
            for (int c = 0; c < 10000; c++) {
                try {
                    test();
                } catch (NullPointerException npe) {
                    // swallow
                }
            }
            System.out.print("Boom! ");
        }
        h = new Holder();
    }

    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    @Benchmark
    public int test() {
        int sum = 0;
        for (int c = 0; c < 100; c++) {
            sum += h.x;
        }
        return sum;
    }

    static class Holder {
        int x;
    }
}
```

从表面上看，这个测试用例很简单：执行 100 次整数加法。

具体来看，这个测试用例有几次很巧妙的地方：

1. 这个测试参数化了 `blowup`，当 `blowup = true` 时在第三次迭代会暴露 `null` 对象给 `test()` 方法。
2. 这个测试以不安全的方式使用循环。通过 `LoopUnrollLimit` 设置 Hotspot 不展开循环，这样可以消除这个问题。
3. 这个测试一次又一次地访问同一个对象。聪明的优化器可以将 `h` 字段的加载提升到循环外面，然后进行积极地优化。通过将 `h` 声明为 `volatile` 可以消除这个问题：除非我们面对的是一个像上帝一样聪明的优化器，否则这足以打破提升优化。
4. 这个测试使用编译器提示来打破 `test` 的内联。严格来说这不是该测试必需的，但是这是安全措施。原因如下：该测试依赖 `test` 的分析信息，更聪明的编译器可以使用 caller-callee profiles 来区分不同调用来源（`setup()` 或者测试用例自身的循环）的分析信息。

在最近的 8u232<sup>[1]</sup> 版本中测试结果如下：

```
Benchmark        (blowup)  Mode  Cnt   Score   Error  Units
ImplicitNP.test     false  avgt   15  40.417 ± 0.030  ns/op
ImplicitNP.test      true  avgt   15  63.187 ± 0.156  ns/op
```

这里具体的数据无关紧要，重要的是一种情况比另外一种快得多。`blowup = false` 的情况明显快。如果要深入探究原因，我们可以借助 `-prof perfnorm`，这个工具可以展示底层机器计数器：

```
Benchmark                       (blowup)  Mode  Cnt    Score    Error  Units

ImplicitNP.test                    false  avgt   15   40.484 ±  0.090  ns/op
ImplicitNP.test:L1-dcache-loads    false  avgt    3  206.606 ± 24.336   #/op
ImplicitNP.test:L1-dcache-stores   false  avgt    3    5.861 ±  0.426   #/op
ImplicitNP.test:branches           false  avgt    3  102.972 ± 13.679   #/op
ImplicitNP.test:cycles             false  avgt    3  141.252 ± 22.330   #/op
ImplicitNP.test:instructions       false  avgt    3  521.998 ± 87.292   #/op

ImplicitNP.test                     true  avgt   15   63.254 ±  0.047  ns/op
ImplicitNP.test:L1-dcache-loads     true  avgt    3  206.154 ± 15.231   #/op
ImplicitNP.test:L1-dcache-stores    true  avgt    3    4.971 ±  0.677   #/op
ImplicitNP.test:branches            true  avgt    3  199.993 ± 20.805   #/op ; +100 branches
ImplicitNP.test:cycles              true  avgt    3  221.388 ± 13.126   #/op ;  +80 cycles
ImplicitNP.test:instructions        true  avgt    3  714.439 ± 64.476   #/op ; +190 insns
```

所以我们需要寻找一些额外的 branches。注意测试的循环有100次迭代，所以每次迭代都有额外的分支？另外也多了 200 条额外的指令，感觉 "branch" 就是 x86_64 的 `test` 和 `jcc` 指令。

基于以上的假设，我们通过 `-prof perfasm` 的帮助看下一实际的热代码。以下是裁剪的片段。

首先，`blowup = false` 的情况：

```
           ...
  1.71%  ↗  0x...020: mov    0x10(%rsi),%r11d       ; get field "h"
  9.19%  │  0x...024: add    0xc(%r12,%r11,8),%eax  ; sum += h.x
         │                                          ; implicit exception:
         │                                          ; dispatches to 0x...03e
 59.60%  │  0x...029: inc    %r10d                  ; increment "c" and loop
  0.02%  │  0x...02c: cmp    $0x64,%r10d
         ╰  0x...030: jl     0x...d204020
  4.57%     0x...032: add    $0x10,%rsp
  3.16%     0x...036: pop    %rbp
  3.37%     0x...037: test   %eax,0x16a18fc3(%rip)
            0x...03d: retq
            0x...03e: mov    $0xfffffff6,%esi
            0x...043: callq  0x00007f8aed0453e0     ; <uncommon trap>
            ...
```

这里是一个非常紧密的循环，在 `0x…024` 行的指令组合了 `h` 的[压缩引用](https://shipilev.net/jvm/anatomy-quarks/23-compressed-references)解码，对 `h.x` 的访问，以及**隐式空检查**。我们没有发现对 `h` 进行空检查的额外指令。<sup>[2]</sup>

`implicit exception: dispatches to 0x…03e` 这行是 VM 输出的一部分，表示 VM 知道 SEGV 异常来自空检查失败的指令。然后 JVM 信号处理程序执行它的请求并将控制转移到 `0x…03e`，这里将会抛出异常。<sup>[3]</sup>

当然，如果在执行过程中经常遇到 `null`，那么每次都经过信号处理程序会很慢。对于当前的情况，我们可以说抛出异常也很慢，但是这里有两个逻辑问题。第一，即使异常[有时候很慢](https://shipilev.net/blog/2014/exceptional-performance/)，但是如果可以避免的话，那么没有理由让它更慢。第二，我们想要使用相同的机制处理用户编写的空检查，但是用户不会想要简单的 `if (h == null) { … } else { … }` 由于 `h` 的空检查而导致性能急剧下降。因此我们希望只有在 `null` 的频率比较低的情况下使用隐式空检查。

幸运的是，JVM 可以**基于运行时 profile**编译代码。也就是，当 JIT 编译器决定是否生成隐式空检查时，它可以查看分析信息，看看对象是否曾经为 `null`。此外，即使 JIT 编译器已经生成了隐式空检查，然后在关于 `null` 的优化假设违反后也可以**重新编译**代码。`blowup = true` 的情况通过在代码中赋值为 `null` 违反了优化假设。结果 JVM 重新编译代码为：<sup>[4]</sup>

```
            ...
 11.36%  ↗  0x...bd1: mov    0x10(%rsi),%r11d       ; get field "h"
 12.81%  │  0x...bd5: test   %r11d,%r11d            ; EXPLICIT NULL CHECK
  0.02% ╭│  0x...bd8: je     0x...bf4
 17.23% ││  0x...bda: add    0xc(%r12,%r11,8),%eax  ; sum += h.x
 25.07% ││  0x...bdf: inc    %r10d                  ; increment "c" and loop
  8.70% ││  0x...be2: cmp    $0x64,%r10d
  0.02% │╰  0x...be6: jl     0x...bd1
  3.31% │   0x...be8: add    $0x10,%rsp
  2.49% │   0x...bec: pop    %rbp
  2.72% │   0x...bed: test   %eax,0x160e640d(%rip)
        │   0x...bf3: retq
        ↘   0x...bf4: movabs $0x7821044f8,%rsi      ; <preallocated NullPointerException>
            0x...bfe: mov    %r12d,0x10(%rsi)       ; WTF
            0x...c02: add    $0x10,%rsp
            0x...c06: pop    %rbp
            0x...c07: jmpq   0x00007f887d1053a0     ; throw_exception
            ...
```

砰！现在生成的代码是显式空检查了！<sup>[5]</sup>没有用户的干预，隐式空检查转化为了显式。

你在完整的测试日志中可以实时看到相关信息：

```
# JMH version: 1.22
# VM version: JDK 1.8.0_232, OpenJDK 64-Bit Server VM, 25.232-b09
# VM options: -XX:LoopUnrollLimit=1
# Warmup: 5 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: org.openjdk.ImplicitNP.test
# Parameters: (blowup = true)

# Run progress: 50.00% complete, ETA 00:00:30
# Fork: 1 of 3
Warmup Iteration   1: 40.900 ns/op
Warmup Iteration   2: 40.698 ns/op
Warmup Iteration   3: Boom! 63.157 ns/op  // <--- recompilation happened here
Warmup Iteration   4: 63.158 ns/op
Warmup Iteration   5: 63.130 ns/op
Iteration   1: 63.188 ns/op
Iteration   2: 63.208 ns/op
Iteration   3: 63.128 ns/op
Iteration   4: 63.137 ns/op
Iteration   5: 63.143 ns/op
```

你可以看到前两个迭代都正常，然后在第三次迭代中赋值为 `null`，JVM 注意到变化进行重新编译。<sup>[6]</sup>这为空检查提供了基本平稳的性能模型。

## 其它琐事: Shenandoah GC

总的来说，这是一个非常有用的技术，除此之外还有其它使用场景。例如 [Shenandoah GC](https://wiki.openjdk.java.net/display/Shenandoah) 的 [load-reference-barrier](https://developers.redhat.com/blog/2019/06/27/shenandoah-gc-in-jdk-13-part-1-load-reference-barriers/) 需要检查对象是否在 collection set 中。如果不在，屏障可以跳过，因为当前对象不需要移动。

x86_64 平台的代码：

```
................. LRB fastpath............................
     0x...067: testb  $0x1,0x20(%r15)
  ╭  0x...06c: jne    0x...086
..│.............. actual heap access .....................
  │↗ 0x...06e: movl   $0x2a,0xc(%r9)
  ││  ...
..││............. LRB mid path ...........................
..││............. checking in-cset .......................
  ↘│ 0x...086: mov    %r9,%r10
   │ 0x...089: shr    $0x17,%r10           ; %r10 is biased region idx
   │ 0x...08d: movabs $0x7f60d00919f0,%r8  ; %r8 is biased cset bitmap
   │ 0x...097: cmpb   $0x0,(%r8,%r10,1)    ; <--- implicit check for null here!
   ╰ 0x...09c: je     0x...06e
      ...
```

"collection set" 比特是 region 的属性，所以存在一个全局的 "cset bitmap"，用于识别哪个 region 在 collection set 中。为了识别*对象*是否在 collection set 中，将对象的地址整除 region 的大小，然后检查对应的 region bitmap。需要注意的是堆不必以零地址开始。所以整除结果并不是实际的 region 索引。相反，它给你的是*带偏移的* region 索引：有一个偏移常量，这取决于实际的堆基址。实际实现中，我们可以使用*偏移后的索引*查看 cset bitmap！

这使我们在 region bitmap 中可以命中每个合法对象地址，**除了 `null`**，异常地址就访问到 bitmap 之外了。然而我们知道 `null` 将命中哪个地址，所以[可以在那里分配并提交零页](http://hg.openjdk.java.net/jdk/jdk/rev/24eb7720919c)，然后这个检查可以**假装** `null` 的答案是 `0` 或 "false"。这不需要使用单独的运行时检查来处理 `null`，也不是涉及任何信号处理机制。

## 结论

虚拟内存为处理内存访问提供了很多漂亮的技巧。*隐式空检查*利用了大部分空检查不会触发的事实，并在触发的时候让虚拟内存子系统通知我们。带有重新编译功能的托管运行时可以利用 profile 生成正确空检查代码，并且在空检查假设违反之后动态重新生成代码。最后，以上这些对用户来说或多或少是透明的，并且提供了显著的性能收益。

* * *

[1](). 我们使用 8u 版本 —— 而不是哪些新版本 JDK —— 的目的是展示这个优化不是很新 ;)

[2](). 在更复杂的情况中，简化的控制流和不使用显式空检查的空闲寄存器/标志可以提高代码质量。

[3](). 在这段代码中，它实际上进入了所谓的 ”uncommon trap“，之后我们会讨论这个主题。简单来说，这是向运行时发出通知，告诉它某个不会执行的分支被执行了，并要求 JVM 基于这些信息重新编译方法。

[4](). 虽然这个测试用例展示了动态重编译，但是如果我们在测试代码执行前赋值 `null`，更新初始的 profile，那么也会得到相同的效果。

[5](). `0x…bfe: mov %r12d,0x10(%rsi)` 是一个 [low-level WTF](https://twitter.com/shipilev/status/1213203153598001154).

[6](). `-prof perfasm` 过滤了预热阶段发生的事情，这就是我们没有看到反编译的原因。
