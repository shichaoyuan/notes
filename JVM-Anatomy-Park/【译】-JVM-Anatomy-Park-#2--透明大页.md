原文地址：[JVM Anatomy Park #2: Transparent Huge Pages](https://shipilev.net/jvm-anatomy-park/2-transparent-huge-pages/)

## 问题

大页（Large Pages）是什么？透明大页（Transparent Huge Pages）又是什么？它们有什么用？！

## 理论

现在虚拟内存被视为理所当然的特性。只有很少人还记得直接操作物理内存的“实模式（real mode）”编程，与此相反，每个进程拥有自己的*虚拟*地址空间，这段空间将会被映射到实际的物理内存。该特性使得两个进程可以在*相同的*虚拟地址`0x42424242`上具有不同的数据，因为相同的虚拟地址会被映射到*不同的*物理地址。所以当程序访问虚拟地址时，某个组件将会把虚拟地址转换为物理地址。

这部分的功能通常是由操作系统与硬件协同完成的，操作系统负责维护[“页表（page table）”](https://en.wikipedia.org/wiki/Page_table)，而硬件通过“页表移动（page table walk）”转换地址。以页的粒度维护地址转换还是比较容易的，然而这并不高效，因为**每次**内存访问都需要做地址转换！因此这里又对最近的转换增加了缓存——[转换查找缓存 (TLB)](https://en.wikipedia.org/wiki/Translation_lookaside_buffer)。TLB 通常非常小，少于100条记录，因为它需要像 L1 缓存一样快，甚至更快。对于许多工作负载来说，TLB不命中以及相应的页表移动将会非常耗时。

虽然我们不能把 TLB 做大，但是我们可以把**页**变大！大部分硬件支持4K大小的基本页，以及2M/4M/1G的“大页”。拥有更大的页可以将页表缩小，使得页表移动的成本更低。

在 Linux 中至少有两种方式实现更大的页：

*   [**hugetlbfs**](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)。占用部分系统内存，将其暴露为虚拟文件系统，让应用程序从其中`mmap(2)`。这种方式需要操作系统配置和应用程序协同修改。这也是一种“要么全部，要么没有”的方式：hugetlbfs （持久化部分）分配的空间不能被正常的进程使用。
*   [**Transparent Huge Pages (THP)**](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)。这种方式对应用程序来说是透明的，应用可以像平常那样分配内存。理想情况下，应用程序不需要做任何改动。但是实际上这种方式存在空间成本（因为对某些小对象也会分配整个大页）和时间成本（因为有时候THP需要整理内存碎片）。好在存在一个妥协的办法：应用程序可以通过`madvise(2)`告诉 Linux 在何处使用 THP。

至于为什么在命名上交替使用了“large”和“huge”，那我就不清楚了。不管怎样 OpenJDK 两种方式都支持：

```
$ java -XX:+PrintFlagsFinal 2>&1 | grep Huge
  bool UseHugeTLBFS             = false      {product} {default}
  bool UseTransparentHugePages  = false      {product} {default}
$ java -XX:+PrintFlagsFinal 2>&1 | grep LargePage
  bool UseLargePages            = false   {pd product} {default}
```

`-XX:+UseHugeTLBFS` 内存映射 Java 堆到 hugetlbfs，

`-XX:+UseTransparentHugePages`仅仅`madvise` Java 堆应该使用 THP。这是一个便捷的方式，因为我们知道 Java 堆很大，基本是连续的，可以最大程度享受到大页的好处。

`-XX:+UseLargePages` 是一个通用的配置，用于启动任意可用的方式。在 Linux 中，该参数启动 hugetlbfs，而不是 THP。我猜测这可能是历史原因，毕竟 hugetlbfs 更早出现。

某些应用程序在开启大页之后却造成了[性能下降](https://bugs.openjdk.java.net/browse/JDK-8024838)。（很有趣的是人们通过手动内存管理来避免 GC，然而却由于 THP 的内存碎片整理造成了突增的高延时！）我的直觉是 THP 对大部分生命周期较短的应用程序会造成性能下降，因为内存整理的时间相对于应用程序较短的生命周期比重更明显。

## 实验

我们可以检验大页带来的好处么？当然，让我们以一个通常的工作负载为例，分配然后随机访问`byte[]`数组：

```
public class ByteArrayTouch {

    @Param(...)
    int size;

    byte[] mem;

    @Setup
    public void setup() {
        mem = new byte[size];
    }

    @Benchmark
    public byte test() {
        return mem[ThreadLocalRandom.current().nextInt(size)];
    }
}
```

（完整的代码在[这里](https://shipilev.net/jvm-anatomy-park/2-transparent-huge-pages/ByteArrayTouch.java)）


我们知道随着数组变大，系统的性能开始受 L1 缓存不命中的影响，然后受L2缓存不命中的影响，然后受L3缓存不命中的影响，以及其他的影响。这里我们通常会忽略 TLB 不命中的影响。

在执行测试用例之前，我们需要确定使用多大的堆内存。在我的机器上，L3缓存有 8M，所以 100M 大小的数组就可以超出缓存。这意味着通过`-Xmx1G -Xms1G`分配1G内存肯定就足够了。这也为我们分配 hugetlbfs 提供了指导。

所以，确保设置了下述参数：

```
# HugeTLBFS should allocate 1000*2M pages:
sudo sysctl -w vm.nr_hugepages=1000

# THP to "madvise" only (some distros have an opinion about defaults):
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

我喜欢通过“madvise”使用 THP，因为这使得我可以“选择性”的使用收益较高的部分内存。


执行环境 i7 4790K, Linux x86_64, JDK 8u101：

```
Benchmark               (size)  Mode  Cnt   Score   Error  Units

# Baseline
ByteArrayTouch.test       1000  avgt   15   8.109 ± 0.018  ns/op
ByteArrayTouch.test      10000  avgt   15   8.086 ± 0.045  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.831 ± 0.139  ns/op
ByteArrayTouch.test   10000000  avgt   15  19.734 ± 0.379  ns/op
ByteArrayTouch.test  100000000  avgt   15  32.538 ± 0.662  ns/op

# -XX:+UseTransparentHugePages
ByteArrayTouch.test       1000  avgt   15   8.104 ± 0.012  ns/op
ByteArrayTouch.test      10000  avgt   15   8.060 ± 0.005  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.193 ± 0.086  ns/op // !
ByteArrayTouch.test   10000000  avgt   15  17.282 ± 0.405  ns/op // !!
ByteArrayTouch.test  100000000  avgt   15  28.698 ± 0.120  ns/op // !!!

# -XX:+UseHugeTLBFS
ByteArrayTouch.test       1000  avgt   15   8.104 ± 0.015  ns/op
ByteArrayTouch.test      10000  avgt   15   8.062 ± 0.011  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.303 ± 0.133  ns/op // !
ByteArrayTouch.test   10000000  avgt   15  17.357 ± 0.217  ns/op // !!
ByteArrayTouch.test  100000000  avgt   15  28.697 ± 0.291  ns/op // !!!
```

这里观察到一些现象：
1. 当数组比较小时，缓存和TLB都合适，那么性能相对于基准线没有差异。
2. 随着数组增大，缓存不命中成为影响性能的主要因素，所以在三种场景下耗时都增加了。
3. 随着数组增加，TLB不命中也产生了影响，所以通过设置大页改善了一些！
4. `UseTHP`和`UseHTLBFS`两者的性能改善基本上是一样的，因为它们提供的功能对应用程序来说是一样的。

为了验证 TLB 不命中的猜想，我们可以观察硬件计数器。JMH的`-prof perfnorm`提供了操作粒度的数值。

```
Benchmark                                (size)  Mode  Cnt    Score    Error  Units

# Baseline
ByteArrayTouch.test                   100000000  avgt   15   33.575 ±  2.161  ns/op
ByteArrayTouch.test:cycles            100000000  avgt    3  123.207 ± 73.725   #/op
ByteArrayTouch.test:dTLB-load-misses  100000000  avgt    3    1.017 ±  0.244   #/op  // !!!
ByteArrayTouch.test:dTLB-loads        100000000  avgt    3   17.388 ±  1.195   #/op

# -XX:+UseTransparentHugePages
ByteArrayTouch.test                   100000000  avgt   15   28.730 ±  0.124  ns/op
ByteArrayTouch.test:cycles            100000000  avgt    3  105.249 ±  6.232   #/op
ByteArrayTouch.test:dTLB-load-misses  100000000  avgt    3   ≈ 10⁻³            #/op
ByteArrayTouch.test:dTLB-loads        100000000  avgt    3   17.488 ±  1.278   #/op
```

让我们开始分析吧！在基准测试中每次操作都会 dTLB load miss，但是启动了THP之后却很少。

当然，伴随着启动 THP，你也需要承担内存碎片整理的成本。我们可以将这个成本转移到 JVM 启动时，这样就可以避免程序运行时突然的卡顿，具体的方法是通过`-XX:+AlwaysPreTouch`参数控制 JVM 启动时访问堆中的每个内存页。通常来说预访问是个不错的选择。

有趣的事情发生了：通过设置`-XX:+UseTransparentHugePages`可以使`-XX:+AlwaysPreTouch`更快完成，因为 JVM 可以以更大的步长（每2M一个字节）访问堆，而不是默认情况下较小的步长（每4K一个字节）。进程关闭后释放内存也会更快，这种优势一直延续到[并行释放补丁](https://lwn.net/Articles/715501/)进入发行版的内核。


使用 4TB 大小的堆内存：

```
$ time java -Xms4T -Xmx4T -XX:-UseTransparentHugePages -XX:+AlwaysPreTouch
real    13m58.167s
user    43m37.519s
sys     1011m25.740s

$ time java -Xms4T -Xmx4T -XX:+UseTransparentHugePages -XX:+AlwaysPreTouch
real    2m14.758s
user    1m56.488s
sys     73m59.046s
```

占用和释放 4TB 的内存确实需要执行很长时间！

## 观察

大页是一个改善程序性能的小技巧。Linux 内核中的透明大页更容易使用。JVM 支持的透明大页也很容易启用。通常来说使用大页是一个好主意，特别是在你的应用程序占用大量数据和堆内存的情况下。
