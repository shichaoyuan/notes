
原文地址：[JVM Anatomy Park #1: Lock Coarsening and Loops](https://shipilev.net/jvm-anatomy-park/1-lock-coarsening-for-loops/)

## 问题

众所周知 Hotspot 的[锁粗化优化（lock coarsening optimizations）](https://en.wikipedia.org/wiki/Java_performance#Escape_analysis_and_lock_coarsening)可以有效地合并多个相邻的加锁代码块，因此可以减少加锁的成本。该特性可以将下述代码：

```
synchronized (obj) {
  // statements 1
}
synchronized (obj) {
  // statements 2
}
```

转化为：

```
synchronized (obj) {
  // statements 1
  // statements 2
}
```

这里有一个有趣的问题，Hotspot 对*循环*也会做这种优化么？比如：

```
for (...) {
  synchronized (obj) {
    // something
  }
}
```

...将会优化成这样？

```
synchronized (this) {
  for (...) {
     // something
  }
}
```

理论上来说是可以这样优化的，这有点儿像针对锁的[循环判断外提（loop unswitching）](https://en.wikipedia.org/wiki/Loop_unswitching)。然而如此优化的缺点是将加锁的粒度增加太多，线程在执行循环时将会长时间独占锁。

## 实验

回答这个问题最简单的方法是通过执行代码找到相关的证据。很幸运我们可以使用[JMH](http://openjdk.java.net/projects/code-tools/jmh/)，JMH不仅可以用于性能测试，也可以用于*分析*执行情况，而分析可能是工程领域最重要的部分。让我们从一个简单的测试用例开始：

```
@Fork(..., jvmArgsPrepend = {"-XX:-UseBiasedLocking"})
@State(Scope.Benchmark)
public class LockRoach {
    int x;

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void test() {
        for (int c = 0; c < 1000; c++) {
            synchronized (this) {
                x += 0x42;
            }
        }
    }
}
```

（完整的代码在[这里](https://shipilev.net/jvm-anatomy-park/1-lock-coarsening-for-loops/LockRoach.java)）

这里有几个重要的技巧：
1. 通过`-XX:-UseBiasedLocking`关闭偏向锁，以避免长时间的预热，因为偏向锁并不是直接启动，而是通过初始化阶段等待5秒钟才开始。（参看`BiasedLockingStartupDelay`选项）
2. 关闭`@Benchmark`方法的内联可以帮忙我们从反汇编中分离出相关代码。
3. 每次累加一个魔术数字`0x42`可以帮助我们快速地在反汇编中找到增量。

执行环境 i7 4790K, Linux x86_64, JDK EA 9b156：

```
Benchmark            Mode  Cnt      Score    Error  Units
LockRoach.test       avgt    5   5331.617 ± 19.051  ns/op
```

你能从这些数字里发现什么？说明不了任何问题，对么？我们需要深入看底层做了什么操作。`-prof perfasm`这个参数非常有用，它可以展示生成代码中最热的部分。基于默认设置执行，发现代码中最热的指令是执行锁操作的`lock cmpxchg`(compare-and-sets)，另外只打印了该指令附近的操作。通过设置`-prof perfasm:mergeMargin=1000`来放宽打印的热区代码范围，你可能会得到这样一个咋一看挺可怕的[输出片段](https://shipilev.net/jvm-anatomy-park/1-lock-coarsening-for-loops/default.perfasm)。

进一步剥离这些代码——指令跳跃的关联是加锁和解锁——同时注意累积最多循环的代码（第一列），我们可以看到最热的循环是：

```
 ↗  0x00007f455cc708c1: lea    0x20(%rsp),%rbx
 │          < blah-blah-blah, monitor enter >     ; <--- coarsened!
 │  0x00007f455cc70918: mov    (%rsp),%r10        ; load $this
 │  0x00007f455cc7091c: mov    0xc(%r10),%r11d    ; load $this.x
 │  0x00007f455cc70920: mov    %r11d,%r10d        ; ...hm...
 │  0x00007f455cc70923: add    $0x42,%r10d        ; ...hmmm...
 │  0x00007f455cc70927: mov    (%rsp),%r8         ; ...hmmmmm!...
 │  0x00007f455cc7092b: mov    %r10d,0xc(%r8)     ; LOL Hotspot, redundant store, killed two lines below
 │  0x00007f455cc7092f: add    $0x108,%r11d       ; add 0x108 = 0x42 * 4 <-- unrolled by 4
 │  0x00007f455cc70936: mov    %r11d,0xc(%r8)     ; store $this.x back
 │          < blah-blah-blah, monitor exit >      ; <--- coarsened!
 │  0x00007f455cc709c6: add    $0x4,%ebp          ; c += 4   <--- unrolled by 4
 │  0x00007f455cc709c9: cmp    $0x3e5,%ebp        ; c < 1000?
 ╰  0x00007f455cc709cf: jl     0x00007f455cc708c1
```

哈！这个循环以4的粒度[展开（unrolled）](https://en.wikipedia.org/wiki/Loop_unrolling)了，*然后*锁通过这4次迭代粗化了！好吧，如果这是由于循环展开造成的，那么我们可以这样量化性能影响，通过`-XX:LoopUnrollLimit=1`调低展开的粒度：

```
Benchmark            Mode  Cnt      Score    Error  Units

# Default
LockRoach.test       avgt    5   5331.617 ± 19.051  ns/op

# -XX:LoopUnrollLimit=1
LockRoach.test       avgt    5  20679.043 ±  3.133  ns/op
```

咳！4倍的性能影响！这是显而易见的，因为我们已经观察到最热的指令是锁操作`lock cmpxchg`。自然而然，4倍的锁粗化意味着4倍的吞吐量。很酷！我们可以下结论了么？还不行，我们还得验证关闭循环展开后的反汇编代码。通过 perfasm [可以看出](https://shipilev.net/jvm-anatomy-park/1-lock-coarsening-for-loops/noUnroll.perfasm) 代码在做类似的循环，只是每次只走一步。

```
 ↗  0x00007f964d0893d2: lea    0x20(%rsp),%rbx
 │          < blah-blah-blah, monitor enter >
 │  0x00007f964d089429: mov    (%rsp),%r10        ; load $this
 │  0x00007f964d08942d: addl   $0x42,0xc(%r10)    ; $this.x += 0x42
 │          < blah-blah-blah, monitor exit >
 │  0x00007f964d0894be: inc    %ebp               ; c++
 │  0x00007f964d0894c0: cmp    $0x3e8,%ebp        ; c < 1000?
 ╰  0x00007f964d0894c6: jl     0x00007f964d0893d2 ;
```

哈！好的，一起都清楚了。

## 观察

虽然 Hotspot 不会对整个循环进行锁粗化，但是*另一个*循环优化——循环展开——为通常的锁粗化做了准备，因为循环展开后的代码就是 N 个连续的加锁解锁代码块。如此这样获得了性能收益，而且限制了粗化的粒度，避免了对循环的过度粗化。
