原文地址：[JVM Anatomy Park #14: Constant Variables](https://shipilev.net/jvm-anatomy-park/14-constant-variables/)

## 问题

`final` 实例字段被当做常量来处理？

## 理论

如果你读过《Java 语言规范》中描述 [`final` 变量](https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.12.4)基本语义的章节，那么你会发现一个诡异的段落：

> 常量变量是指用常量表达式（§15.28）初始化的简单类型或 String 类型的 final 变量。无论一个变量是否是常量变量，都涉及相关的类初始化（§12.4.1），二进制兼容性（§13.1, §13.4.9）和明确赋值（§16）。
> — 《Java 语言规范》  4.12.4

精彩！这在实践中可以观察到吗？

## 实践

考虑这段代码。它将输出什么？

```
import java.lang.reflect.Field;

public class ConstantValues {

    final int fieldInit = 42;
    final int instanceInit;
    final int constructor;

    {
        instanceInit = 42;
    }

    public ConstantValues() {
        constructor = 42;
    }

    static void set(ConstantValues p, String field) throws Exception {
        Field f = ConstantValues.class.getDeclaredField(field);
        f.setAccessible(true);
        f.setInt(p, 9000);
    }

    public static void main(String... args) throws Exception {
        ConstantValues p = new ConstantValues();

        set(p, "fieldInit");
        set(p, "instanceInit");
        set(p, "constructor");

        System.out.println(p.fieldInit + " " + p.instanceInit + " " + p.constructor);
    }

}
```

在我们的机器上，输出：

```
42 9000 9000
```

换句话说，即使我们已经重写了“fieldInt”字段，我们也观察不到新值。更令人困惑的是，另外两个变量看起来是更新成功了。这个困惑的答案是，另外两个变量是*空白 final 字段（blank final fields）*，而第一个字段是*常量变量（constant variable）*。如果你探究上述类生成的字节码，那么：

```
$ javap -c -v -p ConstantValues.class
...

final int fieldInit;
  descriptor: I
  flags: ACC_FINAL
  ConstantValue: int 42  <---- oh...

final int instanceInit;
  descriptor: I
  flags: ACC_FINAL

final int constructor;
  descriptor: I
  flags: ACC_FINAL

...
public static void main(java.lang.String...) throws java.lang.Exception;
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC, ACC_VARARGS
  Code:
     ...
     41: bipush        42   // <--- Oh wow, inlined fieldInit field
     43: invokevirtual #18  // StringBuilder.append
     46: ldc           #19  // String " "
     48: invokevirtual #20  // StringBuilder.append
     51: aload_1
     52: getfield      #3   // Field instanceInit:I
     55: invokevirtual #18  // StringBuilder.append
     58: ldc           #19  // String ""
     60: invokevirtual #20  // StringBuilder.append
     63: aload_1
     64: getfield      #4   // Field constructor:I
     67: invokevirtual #18  // StringBuilder.append
     70: invokevirtual #21  // StringBuilder.toString
     73: invokevirtual #22  // System.out.println
```

难怪无法更新“fieldInit”字段：*javac* 已经内联了它的值，JVM 不可能折回重写字节码。

这个优化是由字节码编译器自己完成的。这有明显的性能收益：JIT 编译器不需要做复杂的分析就可以利用常量变量的常量性。但是，像往常一样，这是有代价的。除了对二进制兼容性的影响（例如，我们使用新值重新编译，会发生什么？）—— [JLS 中的相关章节](https://docs.oracle.com/javase/specs/jls/se8/html/jls-13.html#jls-13.4.9)简要讨论了二进制兼容性 —— 这对底层性能测试也有有趣的影响。例如，如果试图量化实例字段上 `final` 修饰符带来的性能改善，那么我们可能需要测量那些最微不足道的东西：

```
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class FinalInitBench {
    // Too lazy to actually build the example class with constructor that initializes
    // final fields, like we have in production code. No worries, we shall just model
    // this with naked fields. Right?

    final int fx = 42;  // Compiler complains about initialization? Okay, put 42 right here!
          int x  = 42;

    @Benchmark
    public int testFinal() {
        return fx;
    }

    @Benchmark
    public int test() {
        return x;
    }
}
```

使用自身的初始化器初始化 final 字段默默地产生了意想不到的效果！运行这个测试用例，使用“perfnorm”分析器查看低层性能计数器，你将会得到一个诡异的结果：`final`字段的访问性能更好一些，**并且**使用了更少加载指令！<sup>[1]</sup>

```
Benchmark                                  Mode  Cnt   Score    Error  Units
FinalInitBench.test                        avgt    9   1.920 ±  0.002  ns/op
FinalInitBench.test:CPI                    avgt    3   0.291 ±  0.039   #/op
FinalInitBench.test:L1-dcache-loads        avgt    3  11.136 ±  1.447   #/op
FinalInitBench.test:L1-dcache-stores       avgt    3   3.042 ±  0.327   #/op
FinalInitBench.test:cycles                 avgt    3   7.316 ±  1.272   #/op
FinalInitBench.test:instructions           avgt    3  25.178 ±  2.242   #/op

FinalInitBench.testFinal                   avgt    9   1.901 ±  0.001  ns/op
FinalInitBench.testFinal:CPI               avgt    3   0.285 ±  0.004   #/op
FinalInitBench.testFinal:L1-dcache-loads   avgt    3   9.077 ±  0.085   #/op  <--- !
FinalInitBench.testFinal:L1-dcache-stores  avgt    3   4.077 ±  0.752   #/op
FinalInitBench.testFinal:cycles            avgt    3   7.142 ±  0.071   #/op
FinalInitBench.testFinal:instructions      avgt    3  25.102 ±  0.422   #/op
```

这是因为生成的代码中根本没有字段加载指令，实际使用的是内联的常量：

```
# test
...
1.02%    1.02%  mov    0x10(%r10),%edx ; <--- get field x
2.50%    1.79%  nop
1.79%    1.60%  callq  CONSUME
...

# testFinal
...
8.25%    8.21%  mov    $0x2a,%edx      ; <--- just use inlined "42"
1.79%    0.56%  nop
1.35%    1.19%  callq  CONSUME
...
```

这本身不是问题，但是*空白*final字段的测试结果会有所不同，而这更接近真实的使用场景。所以：

```
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class FinalInitCnstrBench {
    final int fx;
    int x;

    public FinalInitCnstrBench() {
        this.fx = 42;
        this.x = 42;
    }

    @Benchmark
    public int testFinal() {
        return fx;
    }

    @Benchmark
    public int test() {
        return x;
    }
}
```

输出了更合理的结果，两个测试方法输出了相同的性能：<sup>[2]</sup>

```
Benchmark                                            Mode  Cnt   Score    Error  Units
FinalInitCnstrBench.test                             avgt    9   1.922 ±  0.003  ns/op
FinalInitCnstrBench.test:CPI                         avgt    3   0.289 ±  0.049   #/op
FinalInitCnstrBench.test:L1-dcache-loads             avgt    3  11.171 ±  1.429   #/op
FinalInitCnstrBench.test:L1-dcache-stores            avgt    3   3.042 ±  0.031   #/op
FinalInitCnstrBench.test:cycles                      avgt    3   7.301 ±  0.445   #/op
FinalInitCnstrBench.test:instructions                avgt    3  25.235 ±  1.732   #/op

FinalInitCnstrBench.testFinal                        avgt    9   1.919 ±  0.002  ns/op
FinalInitCnstrBench.testFinal:CPI                    avgt    3   0.287 ±  0.014   #/op
FinalInitCnstrBench.testFinal:L1-dcache-loads        avgt    3  11.170 ±  1.104   #/op
FinalInitCnstrBench.testFinal:L1-dcache-stores       avgt    3   3.039 ±  0.864   #/op
FinalInitCnstrBench.testFinal:cycles                 avgt    3   7.278 ±  0.394   #/op
FinalInitCnstrBench.testFinal:instructions           avgt    3  25.314 ±  0.588   #/op
```

## 观察

Java 中的常量比较复杂，有许多有趣的极端情况。*字节码编译器*特殊处理的常量变量是一个极端情况。如果不是在构造方法中初始化字段，那么低层性能测试的结果可能会让你吃惊。为了捕捉和量化这些极端情况，所以在 JMH 中添加了 "perfasm" 和 "perfnorm" 分析器，用以分析测试结果。

* * *

[1] 实际上也减少了一对 load-store 指令，这是更好的注册器分配的副作用。
[2] 实际上，即时编译器的工作方式更合理，这是下一篇博文的主题。
