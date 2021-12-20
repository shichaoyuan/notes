原文地址：[JVM Anatomy Park #15: Just-In-Time Constants](https://shipilev.net/jvm-anatomy-park/15-just-in-time-constants/)

## 问题

当然，优化器可以利用程序中的常量。JVM 有优化么？

## 理论

当然，基于常量的优化是最有效的优化。在编译的时候处理完，运行时就不需要处理了，这是无法超越的优化方式。但是什么是常量？看起来普通字段不是常量：它们随时都可能改变。那么`final`字段呢？它们是不变的。但是，因为实例字段是对象状态的一部分，所以`final` 实例字段的值也依赖对象的*身份*：

```
class M {
  final int x;
  M(int x) { this.x = x; }
}

M m1 = new M(1337);
M m2 = new M(8080);

void work(M m) {
  return m.x; // what to compile in here, 1337 or 8080?
}
```

因此，这是理所当然的，如果我们在不知道参数对象身份的情况下编译方法 `work` <sup>[1]</sup>，那么我们唯一可以相信的是**static final** 字段：由于`final`所以是不可变的，另外我们确定知道“持有对象”的身份，因为它是由类持有的，而不是个别的对象。

我们可以从实践中观察么？

## 实践

考虑这个 JMH 测试用例：

```
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class JustInTimeConstants {

    static final long x_static_final = Long.getLong("divisor", 1000);
    static       long x_static       = Long.getLong("divisor", 1000);
           final long x_inst_final   = Long.getLong("divisor", 1000);
                 long x_inst         = Long.getLong("divisor", 1000);

    @Benchmark public long _static_final() { return 1000 / x_static_final; }
    @Benchmark public long _static()       { return 1000 / x_static;       }
    @Benchmark public long _inst_final()   { return 1000 / x_inst_final;   }
    @Benchmark public long _inst()         { return 1000 / x_inst;         }

}
```

这个测试用例经过精心的构建，使得编译器可以利用除数是常量这一事实，进行除法优化。如果我们执行测试用例，那么结果将类似这样：

```
Benchmark                          Mode  Cnt  Score   Error  Units
JustInTimeConstants._inst          avgt   15  9.670 ± 0.014  ns/op
JustInTimeConstants._inst_final    avgt   15  9.690 ± 0.036  ns/op
JustInTimeConstants._static        avgt   15  9.705 ± 0.015  ns/op
JustInTimeConstants._static_final  avgt   15  1.899 ± 0.001  ns/op
```

使用 `-prof perfasm`探究测试用例中最热的指令，这将会揭示一些实现细节，以及为什么某些测试更快的原因。

`_inst` 与 `_inst_final` 的结果在意料之内：程序读取字段，然后做为除数。大部分时钟周期耗费在整数除法：

```
# JustInTimeConstants._inst / _inst_final hottest loop
0.21%              mov    0x40(%rsp),%r10
0.02%            │  mov    0x18(%r10),%r10    ; get field x_inst / x_inst_final
                 |  ...
0.13%            │  idiv   %r10               ; ldiv
76.59%   95.38%  │  mov    0x38(%rsp),%rsi    ; prepare and consume the value (JMH infra)
0.40%            │  mov    %rax,%rdx
0.10%            │  callq  CONSUME
                 |  ...
1.51%            │  test   %r11d,%r11d        ; call @Benchmark again
                 ╰  je     BACK
```

`_static` 的结果比较有趣：程序从静态字段驻留的本地类镜像读取数据。因为运行时知道当前在处理哪个类（静态字段访问是静态解析的！），所以我们内联常量指针到镜像，通过预定义的偏移量访问字段。但是，因为我们不知道字段的值是什么 —— 代码生成之后字段的值可以修改 —— 所以仍然需要做相同的整数除法：

```
# JustInTimeConstants._static hottest loop
0.04%              movabs $0x7826385f0,%r10  ; native mirror for JustInTimeConstants.class
0.02%            │  mov    0x70(%r10),%r10    ; get static x_static
                 |  ...
0.02%            │  idiv   %r10               ;*ldiv
72.78%   95.51%  |  mov    0x38(%rsp),%rsi    ; prepare and consume the value (JMH infra)
0.38%            │  mov    %rax,%rdx
0.04%    0.06%   │  data16 xchg %ax,%ax
         0.02%   │  callq  CONSUME
                 |  ...
0.13%            │  test   %r11d,%r11d        ; call @Benchmark again
                 ╰  je     BACK
```

`_static_final` 的结果是最有趣的。JIT 编译器精确知道要处理的数值，所以可以激进地优化。因此循环计算只是重用预计算的值即可，"1000 / 1000"也就是是”1“<sup>[2]</sup>：

```
# JustInTimeConstants._static_final hottest loop
1.36%    1.40%     mov    %r8,(%rsp)
7.73%    7.40%   │  mov    0x8(%rsp),%rdx       ; <--- slot holding the "long" constant "1"
0.45%    0.51%   │  mov    0x38(%rsp),%rsi      ; prepare and consume the value (JMH infra)
3.59%    3.24%   │  nop
1.44%    0.54%   │  callq  CONSUME
                 | ...
3.46%    2.37%   │  test   %r10d,%r10d          ; call @Benchmark again
                 ╰  je     BACK
```

所以性能可以用编译器通过 `static final` 常量化的能力解释。

## 观察

注意在这个例子中，*字节码编译器*（例如 javac）不知道 `static final` 字段的值，因为这个字段是由*运行时值*初始化的。当 JIT 编译开始的时候，类已经成功的初始化了，数值已经确定了，所以就可以利用了！这实际上是*即时常量*。这样就可以开发非常高效，但是运行时可调的代码：实际上，整个过程可以被认为是基于预处理器的断言的替换。<sup>[3]</sup>我经常会忘记 C++ 中这种技巧，在 C++ 中编译全部都是提前的，因此如果你想要关键代码依赖运行时选项，那么你必须具有创造性。<sup>[4]</sup>

其中很重要的部分是解释器和分层编译。类初始化器通常是冷代码，因为它们只执行一次。但是，当我们加载和初始化类，第一次访问字段的时候，更重要的是处理类初始化中“懒加载”的部分。解释器或者基准 JIT 编译器（比如 Hotspot 中的 C1）将会处理这些。当优化 JIT 编译器（比如 Hotspot 中的 C2）执行相同的方法时，重新编译的方法所需要的类已经完全初始化了，其中`static final`字段已经明确知道值了。

* * *

[1] 这里不排除基于流的优化，比如调用内联的`work(new M(4242))`

[2] 用 `int` 替换 `long` 做相同的测试将会输出 `mov $0x1, %edx`，但是我懒得重新格式化所有列出的汇编代码。

[3] 这并不是完全奏效，因为默认的内联启发式算法仍然需要考虑方法的字节码长度，而不管其中有多少死代码。这个问题将通过增量内联等逐步改善。

[4] 这几乎不可避免的转变为模板或者元编程，我们喜欢编写，但是讨厌调试。
