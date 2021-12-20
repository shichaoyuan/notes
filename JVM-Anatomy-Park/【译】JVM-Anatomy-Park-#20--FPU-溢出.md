原文地址：[JVM Anatomy Park #20: FPU Spills](https://shipilev.net/jvm-anatomy-park/20-fpu-spills/)

## 问题

当代码中完全没有浮点数或者向量操作的时候，我在 JVM 生成的 x86 机器码中居然也看到了 XMM 寄存器的使用。这是怎么回事？

## 理论

浮点运算单元（FPU）和向量单元（vector unit）普遍存在于现代 CPU 中，在许多情况下它们为 FPU 特定操作提供了寄存器。例如 Intel x86_64 中的 SSE 和 AVX 增设了 XMM、YMM 和 ZMM 寄存器，这些寄存器可以用于配合更宽的指令。

虽然非向量指令集并不与向量和非向量寄存器正交（例如，在 x86_64 中我们不能在 XMM 寄存器上使用通用的 IMUL），但是这些寄存器仍然提供了一个有趣的**存储**功能：我们可以在其中临时存储数据，即使这些数据不用于向量操作。<sup>[1]</sup>

关于寄存器分配。寄存器分配器的职责是，维护在特定的编译单元（例如，方法）中程序需要的所有操作数的程序表示，并且映射这些虚操作数到实际的机器寄存器 —— 为它们*分配寄存器*。在许多真实的程序中，在给定程序位置，虚操作数的数量会大于可用机器寄存器的数量。在那时，寄存器分配器需要将某些操作数放到寄存器之外的其它位置 —— 比如放到栈上 —— 也就是，*溢出*操作数。

现在在 x86_64 中有 16 个通用寄存器（并不是所有的都可用），在大部分现代的机器上还会有 16 个 AVX 寄存器。我们可以溢出到 XMM 寄存器，而不是栈上么？是的，我们可以！这有什么好处么？

## 实践

考虑这个简单的 JMH 测试用例。我们用一种特别的方式构建测试用例（假设 Java 具有预处理能力，为简单起见）：

```
import org.openjdk.jmh.annotations.*;

import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class FPUSpills {

    int s00, s01, s02, s03, s04, s05, s06, s07, s08, s09;
    int s10, s11, s12, s13, s14, s15, s16, s17, s18, s19;
    int s20, s21, s22, s23, s24;

    int d00, d01, d02, d03, d04, d05, d06, d07, d08, d09;
    int d10, d11, d12, d13, d14, d15, d16, d17, d18, d19;
    int d20, d21, d22, d23, d24;

    int sg;
    volatile int vsg;

    int dg;

    @Benchmark
#ifdef ORDERED
    public void ordered() {
#else
    public void unordered() {
#endif
        int v00 = s00; int v01 = s01; int v02 = s02; int v03 = s03; int v04 = s04;
        int v05 = s05; int v06 = s06; int v07 = s07; int v08 = s08; int v09 = s09;
        int v10 = s10; int v11 = s11; int v12 = s12; int v13 = s13; int v14 = s14;
        int v15 = s15; int v16 = s16; int v17 = s17; int v18 = s18; int v19 = s19;
        int v20 = s20; int v21 = s21; int v22 = s22; int v23 = s23; int v24 = s24;
#ifdef ORDERED
        dg = vsg; // Confuse optimizer a little
#else
        dg = sg;  // Just a plain store...
#endif
        d00 = v00; d01 = v01; d02 = v02; d03 = v03; d04 = v04;
        d05 = v05; d06 = v06; d07 = v07; d08 = v08; d09 = v09;
        d10 = v10; d11 = v11; d12 = v12; d13 = v13; d14 = v14;
        d15 = v15; d16 = v16; d17 = v17; d18 = v18; d19 = v19;
        d20 = v20; d21 = v21; d22 = v22; d23 = v23; d24 = v24;
    }
}
```

测试用例一次性读写多对字段。优化器[实际上不受特定程序顺序的约束](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/#wishful-hb-actual)。的确，这就是我们在 `unordered` 测试中观察到的：

```
Benchmark                                  Mode  Cnt   Score    Error  Units

FPUSpills.unordered                        avgt   15   6.961 ±  0.002  ns/op
FPUSpills.unordered:CPI                    avgt    3   0.458 ±  0.024   #/op
FPUSpills.unordered:L1-dcache-loads        avgt    3  28.057 ±  0.730   #/op
FPUSpills.unordered:L1-dcache-stores       avgt    3  26.082 ±  1.235   #/op
FPUSpills.unordered:cycles                 avgt    3  26.165 ±  1.575   #/op
FPUSpills.unordered:instructions           avgt    3  57.099 ±  0.971   #/op
```

大约有 26 个 load-store 指令对，大致对应测试用例中的 25 对读写操作。但是我们没有 25 个通用寄存器！perfasm 的输出表明，优化器合并了邻近的 load-store 指令对，所以寄存器的压力很低：

```
  0.38%    0.28%    movzbl 0x94(%rcx),%r9d
                  │  ...
  0.25%    0.20%  │  mov    0xc(%r11),%r10d    ; getfield s00
  0.04%    0.02%  │  mov    %r10d,0x70(%r8)    ; putfield d00
                  │  ...
                  │  ... (transfer repeats for multiple vars) ...
                  │  ...
                  ╰  je     BACK
```

此时此刻，我们想要欺骗一下优化器，制造一点儿混淆，使得所有的加载操作在存储操作前执行完。这是 `ordered` 测试的场景，我们可以看到加载和存储操作是分开执行的：首先执行所有加载操作，然后执行所有存储操作。当所有加载操作完成时，寄存器的压力最高，但是此时存储操作还没有开始。即使那样，相对于 `unordered` 也没有明显的性能区别：

```
Benchmark                                  Mode  Cnt   Score    Error  Units

FPUSpills.unordered                        avgt   15   6.961 ±  0.002  ns/op
FPUSpills.unordered:CPI                    avgt    3   0.458 ±  0.024   #/op
FPUSpills.unordered:L1-dcache-loads        avgt    3  28.057 ±  0.730   #/op
FPUSpills.unordered:L1-dcache-stores       avgt    3  26.082 ±  1.235   #/op
FPUSpills.unordered:cycles                 avgt    3  26.165 ±  1.575   #/op
FPUSpills.unordered:instructions           avgt    3  57.099 ±  0.971   #/op

FPUSpills.ordered                          avgt   15   7.961 ±  0.008  ns/op
FPUSpills.ordered:CPI                      avgt    3   0.329 ±  0.026   #/op
FPUSpills.ordered:L1-dcache-loads          avgt    3  29.070 ±  1.361   #/op
FPUSpills.ordered:L1-dcache-stores         avgt    3  26.131 ±  2.243   #/op
FPUSpills.ordered:cycles                   avgt    3  30.065 ±  0.821   #/op
FPUSpills.ordered:instructions             avgt    3  91.449 ±  4.839   #/op
```

这是*因为*我们设法将操作数溢出到了 *XMM 寄存器*，而不是栈上：

```
  3.08%    3.79%    vmovq  %xmm0,%r11
                  │  ...
  0.25%    0.20%  │  mov    0xc(%r11),%r10d    ; getfield s00
  0.02%           │  vmovd  %r10d,%xmm4        ; <--- FPU SPILL
  0.25%    0.20%  │  mov    0x10(%r11),%r10d   ; getfield s01
  0.02%           │  vmovd  %r10d,%xmm5        ; <--- FPU SPILL
                  │  ...
                  │  ... (more reads and spills to XMM registers) ...
                  │  ...
  0.12%    0.02%  │  mov    0x60(%r10),%r13d   ; getfield s21
                  │  ...
                  │  ... (more reads into registers) ...
                  │  ...
                  │  ------- READS ARE FINISHED, WRITES START ------
  0.18%    0.16%  │  mov    %r13d,0xc4(%rdi)   ; putfield d21
                  │  ...
                  │  ... (more reads from registers and putfileds)
                  │  ...
  2.77%    3.10%  │  vmovd  %xmm5,%r11d        : <--- FPU UNSPILL
  0.02%           │  mov    %r11d,0x78(%rdi)   ; putfield d01
  2.13%    2.34%  │  vmovd  %xmm4,%r11d        ; <--- FPU UNSPILL
  0.02%           │  mov    %r11d,0x70(%rdi)   ; putfield d00
                  │  ...
                  │  ... (more unspills and putfields)
                  │  ...
                  ╰  je     BACK
```

注意，我们对一些操作数使用通用寄存器（GPRs），但是当通用寄存器耗尽时，操作数就会溢出了。这里的“因果关系”定义不明确，因为我们似乎是*首先*溢出了，然后使用 GPRs，但是这只是错误的表象，因为寄存器分配器可以直接对所有寄存器进行分配。<sup>[2]</sup>

XMM 溢出的延迟看起来很小：尽管我们为溢出声明了更多指令，但是指令执行非常高效，填补了流水线的缺口：34 个附加的指令，意味着溢出了大约 17 对，但是只增加了 4 个周期。注意，4/34 = ~0.11 clk/insn 这样计算 CPI 是不正确的，这超出了当前 CPU 的能力。但是性能的改善是真实的，因为我们使用了之前没用过的执行指令。

如果没有比较的对象，那么所谓的效率就没有意义了。但是这里，我们有！我们可以用 `-XX:-UseFPUForSpilling` 使得 Hotspot 避免使用 FPU 溢出，这让我们可以了解使用 XMM 溢出获得了多大收益：

```
Benchmark                                  Mode  Cnt   Score    Error  Units

# Default
FPUSpills.ordered                          avgt   15   7.961 ±  0.008  ns/op
FPUSpills.ordered:CPI                      avgt    3   0.329 ±  0.026   #/op
FPUSpills.ordered:L1-dcache-loads          avgt    3  29.070 ±  1.361   #/op
FPUSpills.ordered:L1-dcache-stores         avgt    3  26.131 ±  2.243   #/op
FPUSpills.ordered:cycles                   avgt    3  30.065 ±  0.821   #/op
FPUSpills.ordered:instructions             avgt    3  91.449 ±  4.839   #/op

# -XX:-UseFPUForSpilling
FPUSpills.ordered                          avgt   15  10.976 ±  0.003  ns/op
FPUSpills.ordered:CPI                      avgt    3   0.455 ±  0.053   #/op
FPUSpills.ordered:L1-dcache-loads          avgt    3  47.327 ±  5.113   #/op
FPUSpills.ordered:L1-dcache-stores         avgt    3  41.078 ±  1.887   #/op
FPUSpills.ordered:cycles                   avgt    3  41.553 ±  2.641   #/op
FPUSpills.ordered:instructions             avgt    3  91.264 ±  7.312   #/op
```

哎呀，看到每次操作增长的 load/store 计数器了吗？这些就是栈溢出：栈虽然很快，但是也是在内存中呀，因此需要访问 L1 缓存中的栈空间。此时也需要溢出大约 17 对，但是现在增加了 11 个周期。L1 缓存的吞吐量是这里的限制因素。

最后，我们可以看一下 `-XX:-UseFPUForSpilling` 的 perfasm 输出：

```
  2.45%    1.21%    mov    0x70(%rsp),%r11
                  │  ...
  0.50%    0.31%  │  mov    0xc(%r11),%r10d    ; getfield s00
  0.02%           │  mov    %r10d,0x10(%rsp)   ; <--- stack spill!
  2.04%    1.29%  │  mov    0x10(%r11),%r10d   ; getfield s01
                  │  mov    %r10d,0x14(%rsp)   ; <--- stack spill!
                  │  ...
                  │  ... (more reads and spills to stack) ...
                  │  ...
  0.12%    0.19%  │  mov    0x64(%r10),%ebp    ; getfield s22
                  │  ...
                  │  ... (more reads into registers) ...
                  │  ...
                  │  ------- READS ARE FINISHED, WRITES START ------
  3.47%    4.45%  │  mov    %ebp,0xc8(%rdi)    ; putfield d22
                  │  ...
                  │  ... (more reads from registers and putfields)
                  │  ...
  1.81%    2.68%  │  mov    0x14(%rsp),%r10d   ; <--- stack unspill
  0.29%    0.13%  │  mov    %r10d,0x78(%rdi)   ; putfield d01
  2.10%    2.12%  │  mov    0x10(%rsp),%r10d   ; <--- stack unspill
                  │  mov    %r10d,0x70(%rdi)   ; putfield d00
                  │  ...
                  │  ... (more unspills and putfields)
                  │  ...
                  ╰  je     BACK
```

是的，栈溢出的位置与 XMM 溢出的位置类似。

## 观察

FPU 溢出是缓解寄存器压力的好方法。这不需要增加通用操作的可用寄存器数量，而是为溢出提供了更快的临时存储空间：因此当我们仅仅需要一些额外的溢出槽时，我们可以避免切换到基于 L1 缓存的栈上。

有时这会导致有趣的性能偏差：如果在某些关键路径上没有使用 FPU 溢出，那么我们可能会看到性能下降。例如，慢路径 GC 屏障调用（该调用会清空FPU寄存器）可能会让编译器返回使用基于栈的溢出，而不会尝试高性能的操作。

在 Hotspot 中，`-XX:+UseFPUForSpilling` 默认支持带有 SSE 的 x86 平台，ARMv7 和 AArch64。所以无论你是否知道这个优化，这对多大部分程序都有效。

* * *

[1] 这种技术的极端案例是使用向量寄存器做为行缓冲区！

[2] 某些寄存器分配器可能执行*线性*分配 —— 提高寄存器分配速度，使生成的代码更高效
