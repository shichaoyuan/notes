原文地址：[JVM Anatomy Park #10: String.intern()](https://shipilev.net/jvm-anatomy-park/10-string-intern/)

## 问题

`String.intern()` 究竟是如何工作的？我应该避免使用它吗？

## 理论

如果你曾经研究过 `String` 的文档，那么你肯定留意过一个有趣的方法：

> ```
> public String intern()
> ```
> 
> 返回字符串对象的规范化表示形式。一个初始为空的字符串池，它由类 `String` 私有地维护。
> 当调用 intern 方法时，如果池已经包含一个等于此 `String` 对象的字符串（用 `equals(Object)`方法确定），则返回池中的字符串。否则，将此 `String` 对象添加到池中，并返回此 `String` 对象的引用。
>
> — JDK Javadoc java.lang.String

这段文档的意思好像是： String 提供了字符串池的用户访问入口，我们可以利用这个机制来优化内存使用，是这样么？然而这伴随着一个缺点：在 OpenJDK 中 `String.intern()` 是本地方法，它实际上会调用到 JVM，最终会驻留本地 JVM 字符串池中的 String。由于字符串驻留是 JDK-VM 接口的一部分，所以 VM 本地代码和 JDK 代码需要对 String 对象的引用达成共识。

这种实现将会产生下述影响：

1. 每次执行 `intern()` 都需要跨越 JDK-JVM 接口，这将浪费不少时间。
2. 性能受制于*本地*哈希表的实现，这部分的开发还是比较滞后的，尤其是在并发访问的情况下。
3. 因为字符串是来自本地 VM 结构的引用，所以它们是 GC 根集合的一部分。在很多场景下，这需要在 GC 停顿过程中处理不少附加工作。

这影响大么？

## 实验：吞吐量

我们又构建了一个简单的实验。使用 `HashMap` 和 `ConcurrentHashMap` 可以实现去重和驻留逻辑，因此可以这样构建 [JMH](http://openjdk.java.net/projects/code-tools/jmh/) 测试用例：

```
@State(Scope.Benchmark)
public class StringIntern {

    @Param({"1", "100", "10000", "1000000"})
    private int size;

    private StringInterner str;
    private CHMInterner chm;
    private HMInterner hm;

    @Setup
    public void setup() {
        str = new StringInterner();
        chm = new CHMInterner();
        hm = new HMInterner();
    }

    public static class StringInterner {
        public String intern(String s) {
            return s.intern();
        }
    }

    @Benchmark
    public void intern(Blackhole bh) {
        for (int c = 0; c < size; c++) {
            bh.consume(str.intern("String" + c));
        }
    }

    public static class CHMInterner {
        private final Map<String, String> map;

        public CHMInterner() {
            map = new ConcurrentHashMap<>();
        }

        public String intern(String s) {
            String exist = map.putIfAbsent(s, s);
            return (exist == null) ? s : exist;
        }
    }

    @Benchmark
    public void chm(Blackhole bh) {
        for (int c = 0; c < size; c++) {
            bh.consume(chm.intern("String" + c));
        }
    }

    public static class HMInterner {
        private final Map<String, String> map;

        public HMInterner() {
            map = new HashMap<>();
        }

        public String intern(String s) {
            String exist = map.putIfAbsent(s, s);
            return (exist == null) ? s : exist;
        }
    }

    @Benchmark
    public void hm(Blackhole bh) {
        for (int c = 0; c < size; c++) {
            bh.consume(hm.intern("String" + c));
        }
    }
}
```

这个测试用例尝试驻留一些字符串，实际上只有在第一次循环的时候保存字符串，接下来的循环仅仅是通过现有的映射检查字符串。参数 `size` 控制驻留的字符串数量，因此限制了我们需要处理的字符串表的大小。这也是驻留器的通常用法。

使用 JDK 8u131 执行：

```
Benchmark             (size)  Mode  Cnt       Score       Error  Units

StringIntern.chm           1  avgt   25       0.038 ±     0.001  us/op
StringIntern.chm         100  avgt   25       4.030 ±     0.013  us/op
StringIntern.chm       10000  avgt   25     516.483 ±     3.638  us/op
StringIntern.chm     1000000  avgt   25   93588.623 ±  4838.265  us/op

StringIntern.hm            1  avgt   25       0.028 ±     0.001  us/op
StringIntern.hm          100  avgt   25       2.982 ±     0.073  us/op
StringIntern.hm        10000  avgt   25     422.782 ±     1.960  us/op
StringIntern.hm      1000000  avgt   25   81194.779 ±  4905.934  us/op

StringIntern.intern        1  avgt   25       0.089 ±     0.001  us/op
StringIntern.intern      100  avgt   25       9.324 ±     0.096  us/op
StringIntern.intern    10000  avgt   25    1196.700 ±   141.915  us/op
StringIntern.intern  1000000  avgt   25  650243.474 ± 36680.057  us/op
```

哎哟，怎么会这样？`String.intern()` 要慢得多！慢的原因在本地实现中（乡亲们，“本地”并不意味着“更快”），通过 `perf record -g` 工具就可以清楚的看到原因：

```
-    6.63%     0.00%  java     [unknown]           [k] 0x00000006f8000041
   - 0x6f8000041
      - 6.41% 0x7faedd1ee354
         - 6.41% 0x7faedd170426
            - JVM_InternString
               - 5.82% StringTable::intern
                  - 4.85% StringTable::intern
                       0.39% java_lang_String::equals
                       0.19% Monitor::lock
                     + 0.00% StringTable::basic_add
                  - 0.97% java_lang_String::as_unicode_string
                       resource_allocate_bytes
                 0.19% JNIHandleBlock::allocate_handle
                 0.19% JNIHandles::make_local
```

虽然 JNI 转换本身也耗费了不少时间，但是在 [StringTable](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/910e24afc502/src/share/vm/classfile/stringTable.cpp) 中耗费了更多时间。你可以通过 [`-XX:+PrintStringTableStatistics`](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/910e24afc502/src/share/vm/runtime/globals.hpp#l2578) 参数打印下述内容：

```
StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :   1002714 =  24065136 bytes, avg  24.000
Number of literals      :   1002714 =  64192616 bytes, avg  64.019
Total footprint         :           =  88737856 bytes
Average bucket size     :    16.708  ; <---- !!!!!!
```

在链接哈希表中每个桶中有16个元素就已经超负荷了。更糟糕的是，字符串表*不是可变大小的* —— 虽然曾经尝试过改为大小可变的，但是因为“某些原因”放弃了。你可以通过 `-XX:StringTableSize` 设置更大的字符串表缓解这个问题，例如设置为 10M：

```
Benchmark             (size)  Mode  Cnt       Score       Error  Units

# Default, copied from above
StringIntern.chm           1  avgt   25       0.038 ±     0.001  us/op
StringIntern.chm         100  avgt   25       4.030 ±     0.013  us/op
StringIntern.chm       10000  avgt   25     516.483 ±     3.638  us/op
StringIntern.chm     1000000  avgt   25   93588.623 ±  4838.265  us/op

# Default, copied from above
StringIntern.intern        1  avgt   25       0.089 ±     0.001  us/op
StringIntern.intern      100  avgt   25       9.324 ±     0.096  us/op
StringIntern.intern    10000  avgt   25    1196.700 ±   141.915  us/op
StringIntern.intern  1000000  avgt   25  650243.474 ± 36680.057  us/op

# StringTableSize = 10M
StringIntern.intern        1  avgt    5       0.097 ±     0.041  us/op
StringIntern.intern      100  avgt    5      10.174 ±     5.026  us/op
StringIntern.intern    10000  avgt    5    1152.387 ±   558.044  us/op
StringIntern.intern  1000000  avgt    5  130862.190 ± 61200.783  us/op
```

但是这仅仅是一个有待商榷的测试，因为你必须预先知道字符串数量。如果你盲目的将字符串表设置的很大，那么将会浪费内存。即使字符串表的大小与实际使用情况匹配，然而本地调用的成本仍然会耗费很多时间。

## 实验：GC 停顿

但是本地字符串表可能会导致灾难性后果的原因是：它是 GC 根的一部分！这意味着它需要被垃圾收集器扫描和更新。在 OpenJDK 中，这意味着在停顿过程中做繁重的工作。对于 [Shenandoah](https://wiki.openjdk.java.net/display/shenandoah/Main) 来说，停顿时间的长短主要取决于 GC根集合的大小。这是字符串表有 1M 条记录情况下的 GC 情况：

```
$ ... StringIntern -p size=1000000 --jvmArgs "-XX:+UseShenandoahGC -Xlog:gc+stats -Xmx1g -Xms1g"
...
Initial Mark Pauses (G)    = 0.03 s (a = 15667 us) (n = 2) (lvls, us = 15039, 15039, 15039, 15039, 16260)
Initial Mark Pauses (N)    = 0.03 s (a = 15516 us) (n = 2) (lvls, us = 14844, 14844, 14844, 14844, 16088)
  Scan Roots               = 0.03 s (a = 15448 us) (n = 2) (lvls, us = 14844, 14844, 14844, 14844, 16018)
    S: Thread Roots        = 0.00 s (a =    64 us) (n = 2) (lvls, us =    41,    41,    41,    41,    87)
    S: String Table Roots  = 0.03 s (a = 13210 us) (n = 2) (lvls, us = 12695, 12695, 12695, 12695, 13544)
    S: Universe Roots      = 0.00 s (a =     2 us) (n = 2) (lvls, us =     2,     2,     2,     2,     2)
    S: JNI Roots           = 0.00 s (a =     3 us) (n = 2) (lvls, us =     2,     2,     2,     2,     4)
    S: JNI Weak Roots      = 0.00 s (a =    35 us) (n = 2) (lvls, us =    29,    29,    29,    29,    42)
    S: Synchronizer Roots  = 0.00 s (a =     0 us) (n = 2) (lvls, us =     0,     0,     0,     0,     0)
    S: Flat Profiler Roots = 0.00 s (a =     0 us) (n = 2) (lvls, us =     0,     0,     0,     0,     0)
    S: Management Roots    = 0.00 s (a =     1 us) (n = 2) (lvls, us =     1,     1,     1,     1,     1)
    S: System Dict Roots   = 0.00 s (a =     9 us) (n = 2) (lvls, us =     8,     8,     8,     8,    11)
    S: CLDG Roots          = 0.00 s (a =    75 us) (n = 2) (lvls, us =    68,    68,    68,    68,    81)
    S: JVMTI Roots         = 0.00 s (a =     0 us) (n = 2) (lvls, us =     0,     0,     0,     0,     1)
```

每次停顿都超过 13ms，*这仅仅因为*我们增加了更多内容到根集合中。

这也提示某些 GC 实现仅仅在某些情况下做字符串表清理即可。例如，从 JVM 角度来看，如果类没有被卸载，那么清理字符串表的意义不大，因为加载的类是驻留字符串的主要来源。所以至少在 G1 和 CMS 中，这个工作负载将会展现很有趣的行为：

```
public class InternMuch {
  public static void main(String... args) {
    for (int c = 0; c < 1_000_000_000; c++) {
      String s = "" + c + "root";
      s.intern();
    }
  }
}
```

在 CMS 下执行:

```
$ java -XX:+UseConcMarkSweepGC -Xmx2g -Xms2g -verbose:gc -XX:StringTableSize=6661443 InternMuch

GC(7) Pause Young (Allocation Failure) 349M->349M(989M) 357.485ms
GC(8) Pause Initial Mark 354M->354M(989M) 3.605ms
GC(8) Concurrent Mark
GC(8) Concurrent Mark 1.711ms
GC(8) Concurrent Preclean
GC(8) Concurrent Preclean 0.523ms
GC(8) Concurrent Abortable Preclean
GC(8) Concurrent Abortable Preclean 935.176ms
GC(8) Pause Remark 512M->512M(989M) 512.290ms
GC(8) Concurrent Sweep
GC(8) Concurrent Sweep 310.167ms
GC(8) Concurrent Reset
GC(8) Concurrent Reset 0.404ms
GC(9) Pause Young (Allocation Failure) 349M->349M(989M) 369.925ms
```

到目前为止一切顺利。遍历超载的字符串表耗费相当长的时间。但是更要命的情况是通过 `-XX:-ClassUnloading` 关闭类卸载。这[有效地](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/385668275400/src/share/vm/gc/cms/concurrentMarkSweepGeneration.cpp#l2559) [关闭](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/385668275400/src/share/vm/gc/cms/concurrentMarkSweepGeneration.cpp#l5239)了正常 GC 周期中的字符串表清理！你可能已经猜到了接下来的事情：

```
$ java -XX:+UseConcMarkSweepGC -Xmx2g -Xms2g -verbose:gc -XX:-ClassUnloading -XX:StringTableSize=6661443 InternMuch

GC(11) Pause Young (Allocation Failure) 273M->308M(989M) 338.999ms
GC(12) Pause Initial Mark 314M->314M(989M) 66.586ms
GC(12) Concurrent Mark
GC(12) Concurrent Mark 175.625ms
GC(12) Concurrent Preclean
GC(12) Concurrent Preclean 0.539ms
GC(12) Concurrent Abortable Preclean
GC(12) Concurrent Abortable Preclean 2549.523ms
GC(12) Pause Remark 696M->696M(989M) 133.920ms
GC(12) Concurrent Sweep
GC(12) Concurrent Sweep 175.949ms
GC(12) Concurrent Reset
GC(12) Concurrent Reset 0.463ms
GC(14) Pause Full (Allocation Failure) 859M->0M(989M) 1541.465ms  <---- !!!
GC(13) Pause Young (Allocation Failure) 859M->0M(989M) 1541.515ms
```

Full STW GC，我的老朋友。对于 CMS 来说，这个 `ExplicitGCInvokesConcurrentAndUnloadsClasses` 参数可以稍微缓解一下问题，假设用户将会时而调用一下 `System.gc()`。

## 观察

> 注：我们仅仅讨论可以实现驻留和去重的方式，并且假设这或者是改善内存使用的需要，或者是底层 `==` 优化的需要，或者别的隐蔽的需要。这些需求可以接受，也可以拒绝。关于更多 Java 字符串的细节，可以看我另外一个演讲：[“java.lang.String 问答集”](https://shipilev.net/#string-catechism)

对于 OpenJDK 来说，`String.intern()` 方法是通向本地 JVM 字符串表的通道，同时也伴随着警告：吞吐量、内存使用、停顿时间的问题将会阻挡用户。这些警告的影响很容易被低估。手工的去重器和驻留器更可靠，因为它们工作在 Java 层面，仅仅是通常的 Java 对象，可以更好的实现大小调整，当不再需要时可以完全丢弃。GC 协助的[字符串去重](http://openjdk.java.net/jeps/192)可以更好的解决问题。

在大部分我们处理过的项目中，从热路径中移除 `String.intern()`，或者替换为手工的去重器，是非常有效的性能优化方案。在深思熟虑之前请不要使用 `String.intern()`，好吗？
