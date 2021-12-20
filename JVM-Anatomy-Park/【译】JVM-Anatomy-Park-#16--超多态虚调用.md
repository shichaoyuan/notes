原文地址：[JVM Anatomy Park #16: Megamorphic Virtual Calls](https://shipilev.net/jvm-anatomy-park/16-megamorphic-virtual-calls/)

## 问题

我听说超多态虚调用（megamorphic virtual calls）非常糟糕，因为这种调用是由解释器执行的，而不是优化编译器。这是真的么？

## 理论

如果你读过[许多 Hotspot 中关于虚调用优化的文章](https://shipilev.net/blog/2015/black-magic-method-dispatch/)，你可能会有这样的印象：超多态调用邪恶到家了，因为它们执行慢路径处理，无法获得编译器优化的好处。如果你尝试理解调用去虚化失败之后 OpenJDK 的行为，那么你可能会惊讶这导致的性能问题。但是要考虑到，即使是基准编译器，JVM 也工作地相当好，在某些情况下，即使是解释器的性能也是可以接受的（并且这关系到 time-to-performance）。

所以，现在下结论说运行时系统只是放弃优化还为时过早？

## 实践

让我们尝试看看虚调用慢路径。因此我们在 JMH 测试用例中制造了人为的超多态调用点：使三个子类访问同一个调用点：

```
import org.openjdk.jmh.annotations.*;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class VirtualCall {

    static abstract class A {
        int c1, c2, c3;
        public abstract void m();
    }

    static class C1 extends A {
        public void m() { c1++; }
    }
    static class C2 extends A {
        public void m() { c2++; }
    }
    static class C3 extends A {
        public void m() { c3++; }
    }

    A[] as;

    @Param({"mono", "mega"})
    private String mode;

    @Setup
    public void setup() {
        as = new A[300];
        boolean mega = mode.equals("mega");
        for (int c = 0; c < 300; c += 3) {
            as[c]   = new C1();
            as[c+1] = mega ? new C2() : new C1();
            as[c+2] = mega ? new C3() : new C1();
        }
    }

    @Benchmark
    public void test() {
        for (A a : as) {
            a.m();
        }
    }
}
```

为了简化分析，我们设置参数 `-XX:LoopUnrollLimit=1 -XX:-TieredCompilation`：禁止循环展开，以免反汇编代码过于复杂，关闭分层编译，保证只用最终优化编译器。虽然我们不太关心性能数值，但是让我们用这些数据构建分析框架：

```
Benchmark         (mode)  Mode  Cnt     Score    Error  Units
VirtualCall.test    mono  avgt    5   325.478 ± 18.156  ns/op
VirtualCall.test    mega  avgt    5  1070.304 ± 53.910  ns/op
```

为了了解*不*使用优化编译器的情况，设置参数 `-XX:CompileCommand=exclude,org.openjdk.VirtualCall::test`

```
Benchmark         (mode)  Mode  Cnt      Score     Error  Units
VirtualCall.test    mono  avgt    5  11598.390 ± 535.593  ns/op
VirtualCall.test    mega  avgt    5  11787.686 ± 884.384  ns/op
```

所以，超多态调用确实很低效，但是这绝不是解释器的问题。在优化过的情况下 “mono” 与 “mega” 的差别基本上是调用开销：在 “mega” 情况下每个元素耗费 3ns，然而在 “mono” 情况下每个元素仅仅耗费 1ns。

通过 `perfasm` 输出 “mega” 情况下的执行情况。为了看得清晰，这里删除了一些内容：

```
....[Hottest Region 1].......................................................................
C2, org.openjdk.generated.VirtualCall_test_jmhTest::test_avgt_jmhStub, version 88 (143 bytes)

  6.93%    5.40%  ↗  0x...5c450: mov    0x40(%rsp),%r9
                  │  ...
  3.65%    4.31%  │  0x...5c47b: callq  0x...0bf60 ;*invokevirtual m
                  │                            ; - org.openjdk.VirtualCall::test@22 (line 76)
                  │                            ;   {virtual_call}
  3.12%    2.34%  │  0x...5c480: inc    %ebp
  3.33%    0.02%  │  0x...5c482: cmp    0x10(%rsp),%ebp
                  ╰  0x...5c486: jl     0x...5c450
                     ...
.............................................................................................
 31.26%   21.77%  <total for region 1>

....[Hottest Region 2].......................................................................
C2, org.openjdk.VirtualCall$C1::m, version 84 (14 bytes) <--- mis-attributed :(

                     ...
                   Decoding VtableStub vtbl[5]@12
  3.95%    1.57%     0x...59bf0: mov    0x8(%rsi),%eax
  3.73%    3.34%     0x...59bf3: shl    $0x3,%rax
  3.73%    5.04%     0x...59bf7: mov    0x1d0(%rax),%rbx
 16.45%   22.42%     0x...59bfe: jmpq   *0x40(%rbx)        ; jump to target
                     0x...59c01: add    %al,(%rax)
                     0x...59c03: add    %al,(%rax)
                     ...
.............................................................................................
 27.87%   32.37%  <total for region 2>

....[Hottest Region 3].......................................................................
C2, org.openjdk.VirtualCall$C3::m, version 86 (26 bytes)

# {method} {0x00007f75aaf4dd50} 'm' '()V' in 'org/openjdk/VirtualCall$C3'

                    ...
                  [Verified Entry Point]
 17.82%   26.04%    0x...595c0: sub    $0x18,%rsp
  0.06%    0.04%    0x...595c7: mov    %rbp,0x10(%rsp)
                    0x...595cc: incl   0x14(%rsi)       ; c3++
  3.53%    5.14%    0x...595cf: add    $0x10,%rsp
                    0x...595d3: pop    %rbp
  3.29%    5.10%    0x...595d4: test   %eax,0x9f01a26(%rip)
  0.02%    0.02%    0x...595da: retq
                    ...
.............................................................................................
 24.73%   36.35%  <total for region 3>
```

所以性能测试调用了一些东西，我们可以假设为虚调用处理程序，然后以 VirtualStub 结束，这应该是所有运行时对虚调用所做的事情：在[虚方法表(VMT)](https://en.wikipedia.org/wiki/Virtual_method_table)的帮助下跳转到实际的方法。<sup>[1]</sup>

但是等一下，这里不是这样！反汇编代码显示实际调用的是 `0x…0bf60`，而不是在 `0x…59bf0` 的 `VirtualStub`？！并且这个调用很频繁，所以调用的目标也应该很频繁，对么？这就是运行时系统戏弄我们的地方。即使编译器放弃优化虚调用，运行时也可以自行处理“悲观”的情况。为了更好的诊断问题，我们需要获取 [fastdebug OpenJDK 构建](https://builds.shipilev.net/openjdk-jdkX/)，这提供了[内联缓存(IC)](https://en.wikipedia.org/wiki/Inline_caching) 的追踪选项：`-XX:+TraceIC`。另外，我们通过 `-prof perfasm:saveLog=true` 保存 Hotspot 日志

你瞧！

```
$ grep IC org.openjdk.VirtualCall.test-AverageTime.log
    IC@0x00007fac4fcb428b: to megamorphic {method} {0x00007fabefa81880} 'm' ()V';
                                 in 'org/openjdk/VirtualCall$C2'; entry: 0x00007fac4fcb2ab0
```

好的，内联缓存代替了位于`0x00007fac4fcb428b` 的调用点。这是什么？这是我们的 Java 调用！

```
$ grep -A 4 0x00007fac4fcb428b: org.openjdk.VirtualCall.test-AverageTime.log
   0.02%    0x00007fac4fcb428b: callq  0x00007fac4fb7dda0
                                  ;*invokevirtual m {reexecute=0 rethrow=0 return_oop=0}
                                  ; - org.openjdk.VirtualCall::test@22 (line 76)
                                  ;   {virtual_call}
```

但是这个 Java 调用中的地址是什么？*解析*运行时存根：

```
$ grep -C 2  0x00007fac4fb7dda0 org.openjdk.VirtualCall.test-AverageTime.log
                    0x00007fac4fb7dcdf: hlt
                  Decoding RuntimeStub - resolve_virtual_call 0x00007fac4fb7dd10
                    0x00007fac4fb7dda0: push   %rbp
                    0x00007fac4fb7dda1: mov    %rsp,%rbp
                    0x00007fac4fb7dda4: pushfq
```

这基本上是由运行时调用的，找出我们想要调用的方法，然后让 IC **修补**指向新解析地址的调用！因为这是一次性操作，难怪不会出现在热代码中。IC 操作行提示修改入口为另一个地址，顺便说一下，也就是实际的 VtableStub：

```
$ grep -C 4 0x00007fac4fcb2ab0: org.openjdk.VirtualCall.test-AverageTime.log
                  Decoding VtableStub vtbl[5]@12
  8.94%    6.49%    0x00007fac4fcb2ab0: mov    0x8(%rsi),%eax
  0.16%    0.06%    0x00007fac4fcb2ab3: shl    $0x3,%rax
  0.20%    0.10%    0x00007fac4fcb2ab7: mov    0x1e0(%rax),%rbx
  2.34%    1.90%    0x00007fac4fcb2abe: jmpq   *0x40(%rbx)
                    0x00007fac4fcb2ac1: int3
```

最后就不需要通过运行时和编译器调用解析逻辑来分发了，解析逻辑就是调用做 VMT 分发的 `VtableStub` —— 从不离开生成的机器码。IC 机制将会以相同的方式处理虚单态和静态调用，指向不需要做 VMT 分发的存根和地址。

我们从第一次 JMH perfasm 输出中看到的像是编译**之后**，但是执行和运行时优化**之前**的代码。<sup>[2]</sup>

## 观察

仅仅因为编译器未能优化到最佳情况，并不意味着最坏情况会更糟糕。诚然，你会放弃一些优化，但是成本不足以大到完全避免虚调用。这个观点与[“Java 方法分发的黑魔法” 的结论](https://shipilev.net/blog/2015/black-magic-method-dispatch/#_conclusion)一致：除非你非常关心，否则没有必要担心调用的性能。

* * *

[1] 接口调用的处理方式与此类似，但是在存根中解析和调用的过程会有一些变化。

[2] [“分析器是说谎的霍比特人 （并且我们讨厌它们！）”](https://www.infoq.com/presentations/profilers-hotspots-bottlenecks) 的另一个例子
