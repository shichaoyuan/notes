原文地址：[JVM Anatomy Park #6: New Object Stages](https://shipilev.net/jvm-anatomy-park/6-new-object-stages/)


## 问题

你可能听说过分配并不是初始化。但是 Java  有构造方法！构造方法是分配？还是初始化？

## 理论

[《GC 手册》](http://gchandbook.org/)将创建新对象划分为三个阶段：

1. **分配**。也就是从进程地址空间为对象数据分配内存。
2. **系统初始化**。也就是编程语言需要的初始化。在 C 语言中，`新`分配的对象不需要初始化。在 Java 语言中，所有对象都需要初始化：设置默认值，设置头部信息，等。
3. **次级（用户）初始化**。也就是执行对象关联的初始化方法和构造方法。

我们在之前的文章已经介绍过[”TLAB 分配“](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/)。本文将会介绍初始化的过程。如果你熟悉 Java 字节码，那你应该知道 Java 语言中的 new 对应*许多*字节码指令。例如：

```
public Object t() {
  return new Object();
}
```

编译为：

```
  public java.lang.Object t();
    descriptor: ()Ljava/lang/Object;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: new           #4                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: areturn
```

*看起来* `new` 指令对应分配和系统初始化两个阶段，调用构造方法（`<init>`）对应用户初始化。但是聪明的运行时系统知道在没有外部操作的时候是可以合并初始化阶段的——例如通过观察构造方法结束之前的对象。我们可以测试 Hotspot 中的初始化优化么？

## 实验

当然我们可以。在测试用例中我们初始化两个类的对象，两个类都是包含一个`int`字段的类：

```
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class UserInit {

    @Benchmark
    public Object init() {
        return new Init(42);
    }

    @Benchmark
    public Object initLeaky() {
        return new InitLeaky(42);
    }

    static class Init {
        private int x;
        public Init(int x) {
            this.x = x;
        }
    }

    static class InitLeaky {
        private int x;
        public InitLeaky(int x) {
            doSomething();
            this.x = x;
        }

        @CompilerControl(CompilerControl.Mode.DONT_INLINE)
        void doSomething() {
            // intentionally left blank
        }
    }
}
```

在测试中禁止空`doSomething()`方法的内联，强制优化器假设在方法中可能会访问`x`。换句话说，对象可能会被*泄漏*给某些外部代码——因为我们不能确保`doSomething()`方法不会泄漏 x。

最好设置启动参数为 `-XX:+UseParallelGC -XX:-TieredCompilation -XX:-UseBiasedLocking`，这样将会使生成的代码更容易理解——总之这是一个教学练习题。JMH 的 `-prof perfasm` 参数可以打印测试用例生成的代码。

这是 `Init` 生成的代码：

```
                                                  ; ------- allocation ----------
0x00007efdc466d4cc: mov    0x60(%r15),%rax          ; TLAB allocation below
0x00007efdc466d4d0: mov    %rax,%r10
0x00007efdc466d4d3: add    $0x10,%r10
0x00007efdc466d4d7: cmp    0x70(%r15),%r10
0x00007efdc466d4db: jae    0x00007efdc466d50a
0x00007efdc466d4dd: mov    %r10,0x60(%r15)
0x00007efdc466d4e1: prefetchnta 0xc0(%r10)
                                                  ; ------- /allocation ---------
                                                  ; ------- system init ---------
0x00007efdc466d4e9: movq   $0x1,(%rax)              ; put mark word header
0x00007efdc466d4f0: movl   $0xf8021bc4,0x8(%rax)    ; put class word header
                                                  ; ...... system/user init .....
0x00007efdc466d4f7: movl   $0x2a,0xc(%rax)          ; x = 42.
                                                  ; -------- /user init ---------
```

你可以看到 TLAB 分配，对象元数据初始化，然后是合并的系统+用户初始化。在 `InitLeaky` 中有些许不同：

```
                                                  ; ------- allocation ----------
0x00007fc69571bf4c: mov    0x60(%r15),%rax
0x00007fc69571bf50: mov    %rax,%r10
0x00007fc69571bf53: add    $0x10,%r10
0x00007fc69571bf57: cmp    0x70(%r15),%r10
0x00007fc69571bf5b: jae    0x00007fc69571bf9e
0x00007fc69571bf5d: mov    %r10,0x60(%r15)
0x00007fc69571bf61: prefetchnta 0xc0(%r10)
                                                  ; ------- /allocation ---------
                                                  ; ------- system init ---------
0x00007fc69571bf69: movq   $0x1,(%rax)              ; put mark word header
0x00007fc69571bf70: movl   $0xf8021bc4,0x8(%rax)    ; put class word header
0x00007fc69571bf77: mov    %r12d,0xc(%rax)          ; x = 0 (%r12 happens to hold 0)
                                                  ; ------- /system init --------
                                                  ; -------- user init ----------
0x00007fc69571bf7b: mov    %rax,%rbp
0x00007fc69571bf7e: mov    %rbp,%rsi
0x00007fc69571bf81: xchg   %ax,%ax
0x00007fc69571bf83: callq  0x00007fc68e269be0       ; call doSomething()
0x00007fc69571bf88: movl   $0x2a,0xc(%rbp)          ; x = 42
                                                  ; ------ /user init ------
```

因为优化器不能确定是否需要 `x`，所以就得假设最糟糕的情况，首先执行系统初始，然后执行用户初始化。

## 观察

虽然教科书中的定义很合理，而且字节码也反映了相同的定义，但是优化器可能会做魔法般的操作来优化性能，只要不产生意外的行为。从编译器的角度来看，这是一个小优化，但是从概念的角度来看，这将会打破理论上的“阶段”边界。
