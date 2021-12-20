原文地址：[Java Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/)

> Gory details you sometimes wondered about, but then did not really wanted to know about

## 1\. 导言

一个 Java 对象占用多少内存，这是一个经常被提及的问题。由于缺失 `sizeof` 操作<sup>[[1]()]</sup>，人们只能去猜测，或者诉诸于传闻。在这篇文章中，我们将会尝试一探 Java 对象的究竟。关于对象占用空间的许多技巧将会变得显而易见，运行时的一些奇怪现象将会得到解释，一些底层的性能行为将会更清晰。

这篇文章有点儿长，所以你可能想分段阅读。文中的每一章基本上是相互独立的，你可以随时阅读。与其他文章相比，这篇文章没有很仔细的评审，随着大家阅读文章指出问题，文章也会被更新修正。所以你需要自担这些风险。

## 2\. 更深入的设计和实践问题 (DDIQ)

在一些章节，你可能会看到关于设计实现问题讨论的侧边条。这些并不能保证回答所有问题，但是尝试回答最常见的问题。这些答案基于*我自己的理解*，所有可能会不准确不完整。如果你对本文有疑问，那么发邮件给我，这可能会产生另一个 DDIQ 侧边条。把这当做”听众问题“即可。

>**DDIQ: 我们确实需要阅读这些侧边条么？**
>
> 并不是。但是这些侧边条可能会让你更好的理解其中的原因。第一次读的时候可以忽略这些侧边条。

## 3\. 研究方法

本文基于 Hotspot JVM，也就是 OpenJDK 及其衍生版中的默认 JVM。如果你不清楚执行的是哪个 JVM，那么很可能是 Hotspot。

### 3.1\. 工具

首先需要选择一个趁手的工具。理解工具能做什么和不能做什么是很重要的。

1. 堆转储。dump Java 堆，然后检查，是一个不错的办法。该方法取决于相信堆转储是运行时堆内存的低级表示。但是很不幸它不是：它是从真实的 Java 堆重建的幻影（通过 GC 自身）。如果你看一下[HPROF 数据格式](http://hg.openjdk.java.net/jdk6/jdk6/jdk/raw-file/tip/src/share/demo/jvmti/hprof/manual.html)，你将会明白这实际是很高级的：这不涉及字段偏移，也不涉及头部信息，唯一的好处是带有对象大小信息，[然而这个信息也是有问题的](https://bugs.openjdk.java.net/browse/JDK-8005604)。堆转储特别适用于查看整个对象图，以及对象之间的关联，但是不适合查看对象本身。
2. 通过 [MXBeans](https://docs.oracle.com/javase/7/docs/jre/api/management/extension/com/sun/management/ThreadMXBean.html) 测量释放和分配的内存。当然我们可以分配很多对象，然后看一下消耗了多少内存。只要分配足够多的对象，我们就可以消除 TLAB 分配（和回收）、后台线程虚分配等引发的异常值。但是这不能使我们了解对象内部：我们只能观测对象的外观大小。这是一个做研究的好方法，但是你需要正确地制定和测试假设，以得到一个可以解释各种结果的可感知的对象模型。
3. 诊断 JVM 标志。但是等等，因为 JVM 自身负责创建对象，那么它确切知道对象布局，我们*”只“*需要获取到即可。[`-XX:+PrintFieldLayout`](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/runtime/globals.hpp#l765) 是一个有用的参数。很不幸这个标志仅仅在 debug JVM 版本可用。<sup>[2]</sup>
4. 戳入对象内部的工具。很幸运通过 `Class.getDeclaredFields` 使用 `Unsafe.objectFieldOffset` 可以获取字段位置信息。这会遇到多个警告：第一，该方法通过反射侵入大部分类，而这是被禁止的；第二，`Unsafe.objectFieldOffset` 并不会正式给出偏移，而是一些”cookie“，可以将其传递给其它 `Unsafe` 方法。<sup>[3]</sup>也就是说，这”通常有效“，所以除非我们在做很重要的事情，否则侵入是可以的。一些工具，特别是 [JOL](https://openjdk.java.net/projects/code-tools/jol/)，为我们完成了这些工作。

在本文中，我们将会使用 JOL，因为我们想要看到 Java 对象细粒度的结构。对于我们的需求，使用 JOL-CLI 包很合适，从这里可以获取到：

```
$ wget https://repo.maven.apache.org/maven2/org/openjdk/jol/jol-cli/0.10/jol-cli-0.10-full.jar -O jol-cli.jar
$ java -jar jol-cli.jar
Usage: jol-cli.jar <mode> [optional arguments]*

Available modes:
   internals: Show the object internals: field layout and default contents, object header
...
```

对于目标对象，我们将会尽可能尝试使用 JDK 中的类。这将会使整个事情易于验证，因为你只需要 JOL CLI JAR 以及 JDK 来执行测试。在更复杂的场景中，我们将会使用 [JOL Samples](https://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/)。作为最后的手段，我们将会使用示例类。

### 3.2\. JDKs

当前最普遍的 JDK 版本还是 JDK 8。因此我们也会使用这个版本，所以本文中的结论是直接有用的。直到 JDK 15，字段布局策略没有实质性的改动，在稍后的章节将会详细讨论。JDK 类布局*自身*也可能改变，所以我们仍将尝试不同的 JDK 版本。另外在某些时候我们将同时需要 x86_32 和 x86_64 两个二进制文件。

对我来看，可以使用我自己编译的二进制文件：

```
$ curl https://builds.shipilev.net/openjdk-jdk8/openjdk-jdk8-latest-linux-x86_64-release.tar.xz | tar xJf -; mv j2sdk-image jdk8-64
$ curl https://builds.shipilev.net/openjdk-jdk8/openjdk-jdk8-latest-linux-x86-release.tar.xz    | tar xJf -; mv j2sdk-image jdk8-32
$ curl https://builds.shipilev.net/openjdk-jdk/openjdk-jdk-latest-linux-x86_64-release.tar.xz   | tar xJf -; mv jdk jdk15-64

$ jdk8-64/bin/java -version
openjdk version "1.8.0-builds.shipilev.net-openjdk-jdk8-b51-20200410"
OpenJDK Runtime Environment (build 1.8.0-builds.shipilev.net-openjdk-jdk8-b51-20200410-b51)
OpenJDK 64-Bit Server VM (build 25.71-b51, mixed mode)

$ jdk8-32/bin/java -version
openjdk version "1.8.0-builds.shipilev.net-openjdk-jdk8-b51-20200410"
OpenJDK Runtime Environment (build 1.8.0-builds.shipilev.net-openjdk-jdk8-b51-20200410-b51)
OpenJDK Server VM (build 25.71-b51, mixed mode)

$ jdk15-64/bin/java -version
openjdk version "15-testing" 2020-09-15
OpenJDK Runtime Environment (build 15-testing+0-builds.shipilev.net-openjdk-jdk-b1214-20200410)
OpenJDK 64-Bit Server VM (build 15-testing+0-builds.shipilev.net-openjdk-jdk-b1214-20200410, mixed mode, sharing)
```

## 4\. 数据类型和它们的表示

我们需要从一些基础知识开始。每次执行 JOL "internals"，你将会看到这样的输出（为了简洁起见，在之后将会省略）：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Object
...
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
```

这意味着 Java 引用占用 4 字节（[压缩引用](https://www.jianshu.com/p/4dd3cacef032)），`boolean`/`byte` 占用 1 字节，`char`/`short` 占用 2 字节，`int`/`float` 占用 4 字节，`double`/`long` 占用 8 字节。当做为数组元素时占用相同的空间。

为什么要搞清楚这个？因为 Java 语言规范对数据表示并没有规定，它只规定了这些类型接受什么值。理论上可以为所有基本类型分配 8 字节，只要对于它们的操作满足规范即可。在当前的 Hotspot 中，基本上所有数据类型精确匹配值的范围，除了 `boolean` 类型。例如 `int` 支持的值范围为 `-2147483648` 至 `2147483647`，精确适合 4 字节带符号表示。

就像上面说的，有一个例外，也就是 `boolean`。理论上它仅仅有两个值：`true` 和 `false`，所以它可以用 1 比特表示。实际上 `boolean` 字段和数组元素仍然占用 1 个字节，这有两个原因：Java 内存模型对于单个字段和元素保证 [没有 word tearing 问题](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.6)，这就导致很难处理 1 比特字段，另外字段偏移做为内存寻址，也就是以*字节*为单位，这使得寻址 `boolean` 字段很尴尬。所以为每个 `boolean` 分配 1 字节是实践上的妥协。

>**DDIQ: 但是无论如何要实现 1 比特 boolean 字段和元素的成本是什么？**
>
>大部分现代的硬件不支持原子访问单个比特。对于读操作不是问题，我们可以读整个字节，然后 mask-shift 想要的比特。但是*对于写操作*问题就大了，对于相邻 `boolean` 字段不能被写覆盖（"the absence of word tearing"）。换句话说，两个线程不能执行整个字节的写操作：
>
>```
>Thread 1:
> mov %r1, (loc)  # read the entire byte
 >or %r1, 0x01    # set the 1-st bit
 >mov (loc), %r1  # write the byte back
>
>Thread 2:
 >mov %r2, (loc)  # read the entire byte
 >or %r2, 0x10    # set the 2-nd bit
 >mov (loc), %r2  # write the byte back
>```
>
>...因为这样会丢失写入的数据：一个线程不会向另一个线程通知写操作，并且会覆盖写入的数据，这是一大禁忌。理论上来说你可以这样实现原子性：
>
>```
>Thread 1:
> lock or (loc), 0x01  # set the 1-st bit in-place
>
>Thread 2:
> lock or (loc), 0x10  # set the 2-st bit in-place
>```
>
>...或者执行 CAS 循环，这都有效，但是这将导致一个小小的 `boolean` 写操作有严重的性能问题。

## 5\. Mark Word

继续关注实际的对象结构。让我们以很简单的 `java.lang.Object` 为例，JOL 输出：

```
$ jdk8-64/java -jar jol-cli.jar internals java.lang.Object
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 00 00 00 # Mark word
      4     4        (object header)              00 00 00 00 # Mark word
      8     4        (object header)              00 10 00 00 # (not mark word)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

结果显示前 12 字节是对象头部。很不幸它没有详细解析头部的内部结构，所以我们需要深入 Hotspot 源码。在代码中你将会看到对象头部[包含两部分](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/oop.hpp#l52)：*mark word* 和 *class word*。class word 保存对象类型信息：关联描述类的本地结构。下一章将会讨论该部分。剩下的元数据保存在 [mark word](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/markWord.hpp#l33) 中。

mark word 有许多用处：

1. 存储移动 GC 的元数据（转移和对象年龄）。
2. 存储身份 hash code
3. 存储锁信息

注意每个对象都有一个 mark word，因为对每个 Java 对象的处理逻辑都是一样的。这也是将对象内部结构起始部分做为 mark word 的原因：VM 需要在耗时敏感的逻辑中快速访问这些信息，例如 STW GC。理解 mark word 的使用场景就能得出其占用空间的下限。

### 5.1\. 为移动 GC 存储转移信息

当 GC 移动对象的时候，它需要记录对象新的位置，至少是临时的。GC 将会使用 mark word  编码该信息，以协调迁移和更新引用的工作。这要求 mark word 至少要与 Java 引用表示一样长。基于 Hotspot 中[压缩引用](https://www.jianshu.com/p/4dd3cacef032)的实现方式，这个引用总是未压缩，所以 mark word 至少要与机器指针一样宽。

反过来，这定义了在实现中 mark word 所需的最小内存：32位平台为 4 字节，64位平台为 8 字节。

>**DDIQ: 我们可以在 mark word 中记录压缩引用么？**
>
>当然，可以。但是在堆太大不能压缩或者压缩引用关闭的情况下仍然是个问题。这可以基于运行时检查处理，但是这样的话在本地 GC 代码中每次访问对象都需要检查，这将很不方便。通过一些工程手段也可以缓解问题，但是基于成本与收益权衡，仍然不建议这样做。

>**DDIQ: 我们可以将 GC 转移信息存储在其它地方么？**
>
>当然，我们可以使用对象中的任一部分。然而有一个重要的问题：从 GC 的角度来看，你不仅需要知道对象转移到哪里，*也*需要知道是否已经转移了对象。这意味着需要设定一个*特定的值*表示“不需要转移”，其它值解析为“转移到 X”。如果我们选择对象中任意位置，而该部分已经存在一个*看起来像*“转移到 X”的值，那么 GC 就会出问题。你需要控制值的设置，以避免这样的冲突。例如在早期的 Shenandoah 原型中，用 class word 存储转移信息，而这个实验早就报废了。最终 Shenandoah 的实现使用了与 STW GCs 相同的 mark word。
>
>你也可以像 ZGC 那样，硬着头皮将转移信息完全存储在堆之外。

很不幸我们不能在 Java 应用（JOL 就是一个 Java 应用）中查看包含 GC 转移信息的 mark word，因为要么在 stop-the-world GC 停顿结束后这些信息就没有了，要么并发 GC 屏障将会阻止我们看到旧对象。

### 5.2\. 存储 GC 的年龄信息

但是我们可以展示对象的年龄信息！

```
$ jdk8-32/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_19_Promotion
# Running 32-bit HotSpot VM.

Fresh object is at d2d6c0f8
*** Move  1, object is at d31104a0
  (object header)  09 00 00 00 (00001001 00000000 00000000 00000000)
                                 ^^^^
*** Move  2, object is at d3398028
  (object header)  11 00 00 00 (00010001 00000000 00000000 00000000)
                                 ^^^^
*** Move  3, object is at d3109688
  (object header)  19 00 00 00 (00011001 00000000 00000000 00000000)
                                 ^^^^
*** Move  4, object is at d43c9250
  (object header)  21 00 00 00 (00100001 00000000 00000000 00000000)
                                 ^^^^
*** Move  5, object is at d41453f0
  (object header)  29 00 00 00 (00101001 00000000 00000000 00000000)
                                 ^^^^
*** Move  6, object is at d6350028
  (object header)  31 00 00 00 (00110001 00000000 00000000 00000000)
                                 ^^^^
*** Move  7, object is at a760b638
  (object header)  31 00 00 00 (00110001 00000000 00000000 00000000)
                                 ^^^^
```

留意每次移动都是如何计数的。这就是对象的年龄。在第 7 次移动后，年龄奇怪地停在了 `6`。这满足 `InitialTenuringThreshold=7` 的默认设置。如果你增加这个阈值，那么对象在转移到老年代之前会经历更多次移动。

### 5.3\. 身份 Hash Code

每个 Java 对象都拥有一个 [hash code](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#hashCode--)。如果用户没有定义，那么就会使用[*身份 hash code*](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#identityHashCode-java.lang.Object-)。<sup>[4]</sup>因为给定对象的身份 hash code 在生成后就不会改变，所以需要存储在某个地方。在 Hotspot 中，它存储在对应对象的 mark word 中。基于身份 hash code 可以接受的准确度，它需要 4 个字节来存储。在上一节中已经讨论过 mark word 至少有 4 字节，所以空间是有的。

>**DDIQ: 如何既存储身份 hash code 又存储 GC 转移信息？**
>
>这个答案有地儿巧妙：当 GC 移动对象的时候，它实际上在处理对象的*两个*副本，一个在旧的位置，一个在新的位置。新对象包含所有原始头部信息。旧对象仅仅服务 GC 的需求，因此可以用 GC 元数据覆写头部。Hotspot 中大部分（可能是所有）stop-the-world GCs 都这样工作，全并发的 Shenandoah GC 也这样[工作](https://developers.redhat.com/blog/2019/06/28/shenandoah-gc-in-jdk-13-part-2-eliminating-the-forward-pointer-word/)。

>**DDIQ： 为什么我们需要存储身份 hash code？这如何影响用户定义的 hash code？**
>
>hash code 应该具有两个属性：a) *良好的分布*，这意味着不同对象的值或多或少是不同的； b) 幂等，这意味着具有相同关键对象组件的对象具有相同的哈希码。请注意后者隐含着，如果对象没有更改那些关键对象组件，则其 hash code 也不会改变。
>
>在对象使用后更改 `hashCode`，经常会导致错误。例如，将对象作为键添加到 `HashMap` 中，然后修改其字段，以至于 `hashCode` 也发生变化，这将导致令人惊讶的现象：该对象可能在 map 中根本找不到，因为内部实现将会查找“错误”的桶。同样，hash code 分布不均匀也会导致性能问题，例如返回一个常数。
>
>对于用户指定的 hash code，通过计算用户选择的字段，以满足上述两个属性。基于足够多的字段和字段值，它就可以很好地分布，并且通过计算未更改（例如 `final`）的字段，就可以幂等。在这种情况下，我们不需要将 hash code 存储在任何地方。一些 hash code 实现可能选择将其缓存在另一个字段中，但这不是必需的。
>
>对于身份 hash code，无法保证*存在*用于计算 hash code 的字段，即使我们有一些字段，也无法得知这些字段是否稳定。考虑没有字段的 `java.lang.Object`：它的 hash code 是什么？分配的两个 `Object` 几乎就是互为镜像：它们具有相同的元数据，它们具有相同的（也就是空的）内容。关于它们的唯一区别是分配的地址，但是即使那样，仍然有两个麻烦。首先，地址的熵很低，尤其是像大多数Java GC 所采用 bump-ptr 分配器，地址分布不均。其次，GC *移动* 对象，因此地址不是幂等的。从性能的角度来看，返回常数是不可行的。
>
>因此，当前的实现从内部 PRNG（“分布良好”）计算身份 hash code，并为每个对象存储（“幂等”）。


由身份 hash code 引起的 markword 改变，可以通过 [JOLSample_15_IdentityHashCode](https://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/JOLSample_15_IdentityHashCode.java#l41) 观察到。以 64位 VM 运行：

```
$ jdk8-64/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_15_IdentityHashCode
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

**** Fresh object
org.openjdk.jol.samples.JOLSample_15_IdentityHashCode$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              88 55 0d 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

hashCode: 5ccddd20

**** After identityHashCode()
org.openjdk.jol.samples.JOLSample_15_IdentityHashCode$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 20 dd cd
      4     4        (object header)              5c 00 00 00
      8     4        (object header)              88 55 0d 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

注意 hash code 为 `5ccddd20`。你可以从对象头部观察到：`01 20 dd cd 5c`。`01` 是 mark word 标签，接下来是小端编写的身份 hash code。我们仍然还有 3 字节空闲！由于我们有相对大的 mark word，所以这是可能的。如果在 mark word 仅有 4 字节的 32 位 VM 上运行会怎样呢？

这是结果：

```
$ jdk8-32/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_15_IdentityHashCode
# Running 32-bit HotSpot VM.

**** Fresh object
org.openjdk.jol.samples.JOLSample_15_IdentityHashCode$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              c0 ab 6b a3
Instance size: 8 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

hashCode: 12ddf17

**** After identityHashCode()
org.openjdk.jol.samples.JOLSample_15_IdentityHashCode$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              81 8b ef 96
      4     4        (object header)              c0 ab 6b a3
Instance size: 8 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

很明显对象头部改变了。但是需要敏锐的眼睛才能看到 `12ddf17` 实际在哪里。你在头部看到的是身份 hashcode “向右移动了一位”。所以第一个字节结尾的一比特，输出了 `81`，剩下的转化为 `12ddf17 >> 1 = 96ef8b`。注意，这将身份 hash code 的*值范围*由 32 比特缩小到了“仅仅” 25 比特。

>**DDIQ: 但是等一下，`System.identityHashCode` 返回 `int` 值，所以这是完整的 32 比特 hashcode 么？**
>
> `identityHashCode` 的范围没有明确指定，就是为了实现这种权衡。在 32 位模式下设置整个 32 比特身份 hash code 需要为每个对象添加一个 word，这对内存占用来说是一个问题。该实现允许缩减 hashcode 以适用存储位数。很不幸这又导致了比较看似相同的 Java 代码在 32 位和 64 位执行时的极端状况。

### 5.4\. 锁信息

Java 同步采用了一个[复杂的状态机](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)。由于每个 Java 对象都可以被同步，所以锁状态应该关联到任一 Java 对象。mark word 保存了大部分状态。

这些锁转换的不同部分可以在对象头部看到。例如，当一个 Java 锁*偏向*某个对象时，我们需要记录对象附近锁的信息。这可以通过 [JOLSample_13_BiasedLocking](https://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/JOLSample_13_BiasedLocking.java#l41) 观察：

```
$ jdk8-64/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_13_BiasedLocking
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

**** Fresh object
org.openjdk.jol.samples.JOLSample_13_BiasedLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 00 00 00  # No lock
      4     4        (object header)              00 00 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
org.openjdk.jol.samples.JOLSample_13_BiasedLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 b0 00 80  # Biased lock
      4     4        (object header)              b8 7f 00 00  # Biased lock
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
org.openjdk.jol.samples.JOLSample_13_BiasedLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 b0 00 80 # Biased lock
      4     4        (object header)              b8 7f 00 00 # Biased lock
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

注意我们如何在头部记录对锁描述符的本地指针：`b0 00 80 b8 7f`。这个锁偏向了这个对象。

当锁没有偏向的时候，情况类似，看例子 [JOLSample_14_FatLocking](https://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/JOLSample_14_FatLocking.java#l41)：

```
$ jdk8-64/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_14_FatLocking
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

**** Fresh object
org.openjdk.jol.samples.JOLSample_14_FatLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # No lock
      4     4        (object header)              00 00 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** Before the lock
org.openjdk.jol.samples.JOLSample_14_FatLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              78 19 57 1a  # Lightweight lock
      4     4        (object header)              85 7f 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
org.openjdk.jol.samples.JOLSample_14_FatLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              0a 4b 00 b4  # Heavyweight lock
      4     4        (object header)              84 7f 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
org.openjdk.jol.samples.JOLSample_14_FatLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              0a 4b 00 b4  # Heavyweight lock
      4     4        (object header)              84 7f 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After System.gc()
org.openjdk.jol.samples.JOLSample_14_FatLocking$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              09 00 00 00  # Lock recycled
      4     4        (object header)              00 00 00 00
      8     4        (object header)              c0 07 08 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

在这里我们看到了锁通常的生命周期：首先对象没有锁记录，然后其它线程获取了锁，设置为（轻量）同步锁，然后主线程参与竞争，锁就膨胀了，在解锁后锁信息仍然指向膨胀锁。最后在某个时刻锁收缩了，对象释放了相关的锁。

### 5.5\. 观察： 身份 Hashcode 使得偏向锁失效

当偏向锁生效的时候需要保存身份 hashcode 会怎样？很简单：身份 hashcode 优先，偏向锁失效。在例子 [JOLSample_26_IHC_BL_Conflict](http://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/JOLSample_26_IHC_BL_Conflict.java) 中可以看到：

```
$ jdk8-64/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

**** Fresh object
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 00 00 00  # No lock
      4     4        (object header)              00 00 00 00
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 b0 00 20  # Biased lock
      4     4        (object header)              e5 7f 00 00  # Biased lock
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 b0 00 20  # Biased lock
      4     4        (object header)              e5 7f 00 00  # Biased lock
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

hashCode: 65ae6ba4

**** After the hashcode
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 a4 6b ae  # Hashcode
      4     4        (object header)              65 00 00 00  # Hashcode
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the second lock
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              50 f9 b8 29  # Lightweight lock
      4     4        (object header)              e5 7f 00 00  # Lightweight lock
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the second lock
org.openjdk.jol.samples.JOLSample_26_IHC_BL_Conflict$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 a4 6b ae  # Hashcode
      4     4        (object header)              65 00 00 00  # Hashcode
      8     4        (object header)              f8 00 01 f8
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

在这个例子中，对于新对象偏向锁生效，但是当我们获取它的 `hashCode`，最终计算它的身份 hash code（由于没有重写 `Object.hashCode`），并且将计算的值设置在 mark word。接下来的锁仅能暂时替换身份 hash code，但是一旦（非偏向）锁释放，它又会回来。由于再也不能将偏向锁信息保存在 mark word 中了，所以偏向锁对这个对象失效了。

>**DDIQ： 这种冲突仅影响一个实例么？**
>
>未必。根本问题是解偏向（unbias）的成本很高，所以偏向锁的机制将会尽可能最小化重新偏向的频率。如果该机制检查到某些解偏向很频繁，它可能会决定整类对象在接下来都[应该重新偏向](http://hg.openjdk.java.net/jdk/jdk/file/9a81c0a34bd0/src/hotspot/share/runtime/globals.hpp#l794)，或者[不能偏向](http://hg.openjdk.java.net/jdk/jdk/file/9a81c0a34bd0/src/hotspot/share/runtime/globals.hpp#l800)。

### 5.6\. 观察： 32 位 VMs 改善内存占用

由于 mark word 依赖对应的位数，可想而知 32 位 VM 的对象占用更少空间，*即使不包含涉及的（引用）字段*。这可以通过 32 位和 64 位 VM 中 `Object` 的布局展示：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Object
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              05 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              00 10 00 00  # Class word (compressed)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

```
$ jdk8-32/bin/java -jar jol-cli.jar internals java.lang.Object
# Running 32-bit HotSpot VM.

Instantiated the sample instance via default constructor.

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              48 51 2b a3  # Class word
Instance size: 8 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

64 位 VM 的 mark word 占用 8 (mark word) + 4 (class word) = 12 字节，相对比 32 位 VM 占用 4 (mark word) + 4 (class word) = 8 字节。由于对象以 8 字节对齐，所以向上舍入为 16 和 8 字节。对于这个小对象来说，空间节省 2x！

## 6\. Class Word

从本地机器来看，每个对象就是一堆字节。存在这样的场景，在运行时我们想知道处理的对象的**类型**。下面是一个不完整的场景列表：

1. 运行时类型检查。
2. 判断对象大小。
3. 指定虚调用和接口调用的目标。

class word 也是被压缩的。即使类指针不是 Java 堆引用，也可以采用类似的优化。<sup>[5]</sup>

### 6.1\. 运行时类型检查

Java 是一个类型安全的语言，所以它需要运行时类型检查。class word 保存对象的类型信息，使得编译器可以进行*运行时类型检查*。运行时检查的效率依赖类型元数据的结构。

如果元数据编码为一个简单的表单，那么编译器可以直接内联这些检查。在 Hotspot 中，class word 保存[指向 VM `Klass` 的本地指针](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/oop.hpp#l57) ，其中保存了一些元信息，包括[继承的父类类型，实现的接口](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/klass.hpp#l120)，等。class word 也保存了 *Java mirror*，也就是 `java.lang.Class` 的[关联实例](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/klass.hpp#l138)。这种迂回的实现方式使得 `java.lang.Class` 可以被当做普通的对象看待，在 GC 的时候移动它们不需要更新每个对象的 class word：`java.lang.Class` 可以移动，但是 `Klass` 一直保持在原有位置。

>**DDIQ: 所以，类型检查成本很高?**
>
>在很多情况下，类型或多或少可以从上下文精确获取。例如，对于接受 `MyClass` 参数的方法，我们可以确定参数是 `MyClass` 或其子类。所以通常不需要类型检查。但是如果失败了，那么我们需要访问对象元数据进行运行时检查。例如[devirtualization](https://shipilev.net/blog/2015/black-magic-method-dispatch/#__strong_c2_dynamic_interface_ref_strong) 和检查类型转换。
>
>例如这样的检查类型转换：
>
>```
>private Object o = new MyClass();
>
>@CompilerControl(CompilerControl.Mode.DONT_INLINE)
>@Benchmark
>public MyClass testMethod() {
>  return (MyClass)o;
>}
>```
>
>```
> mov    0x10(%rsi),%rax       ; getfield "o"
> mov    0x8(%rax),%r10        ; get o.<classword>, Klass*
> movabs $0x7f5bc5144c48,%r11  ; load known Klass* for MyClass
> cmp    %r11,%r10             ; checked cast
> jne    0x00007f64004e1b63    ; not equal? go to slowpath, check subclasses there
> ... %rax is definitely MyClass now
>```

>**DDIQ： 所以，可以基于该结构进行优化（intrinics）么？**
>
>是的，实际上， `Object.getClass()` 将会被这样优化：
>
>```
>@CompilerControl(CompilerControl.Mode.DONT_INLINE)
>@Benchmark
>public Class<?> test() {
>  return o.getClass();
>}
>```
>
>```
>  mov    0x10(%rsi),%r10    ; getfield "o"
>  mov    0x8(%r10),%r10     ; get o.<classword>, Klass*
>  mov    0x70(%r10),%r10    ; get Klass._java_mirror, OopHandle
>  mov    (%r10),%rax        ; dereference OopHandle, get java.lang.Class
>   ... %rax is now java.lang.Class instance
>```

### 6.2\. 判断对象大小

确定对象大小采用类似的方法。相对于不知道对象类型的运行时类型检查，分配过程中或多或少更确定分配对象的大小：可以通过使用的构造器类型和数组初始化器等确定。所以在这个场景下不需要访问 classword。

但是在一些本地代码（最著名的就是垃圾回收器）中，可能会像这样遍历[可解析的堆内存](https://www.jianshu.com/p/22f041895d01)：

```
HeapWord* cur = heap_start;
while (cur < heap_used) {
  object o = (object)cur;
  do_object(o);
  cur = cur + o->size();
}
```

对于这样场景，本地代码需要知道当前（没有类型的！）对象的大小，而且希望高效的获取。所以对于本地代码来说，类元数据的组织很重要。在 Hotspot 中，我们可以通过 [layout helper](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/klass.hpp#l89) 访问 class word，这样就能为我们提供对象大小信息。

>**DDIQ： 你说垃圾收集器使用了堆外的内存？**
>
> 是的，Hotspot GCs [需要访问类元数据](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/oop.inline.hpp#l185)以获取对象大小。大部分情况下将会反复获取同一个元数据，但是相关的内存读操作仍然有一定的成本。The wonders of untyped native accesses! 你可以从本地代码的反汇编中看到这一点，例如 `MutableSpace::object_iterate` [这里](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/gc/parallel/mutableSpace.cpp#l228)：
>
>```
>$ objdump -lrdSC ./build/linux-x86_64-server-release/hotspot/variant->server/libjvm/objs/mutableSpace.o
>...
>void MutableSpace::object_iterate(ObjectClosure* cl) {
>...
>#  HeapWord* p = bottom();
>...
>#  while (p < top()) {
>...
># Klass* oopDesc::klass() const {
>#  if (UseCompressedClassPointers) {
>#    return >CompressedKlassPointers::decode_not_null(_metadata._compressed_klass);
>#  } else {
>#    return _metadata._klass;
>...
>  d0:   49 8b 7e 08             mov    0x8(%r14),%rdi  ; get Klass*
>#  int layout_helper() const            { return _layout_helper; }
>  d4:   8b 4f 08                mov    0x8(%rdi),%ecx  ; get layout helper
>#  if (lh > Klass::_lh_neutral_value) {
>  d7:   83 f9 00                cmp    $0x0,%ecx
>  da:   7e 4e                   jle    12a
>#    if (!Klass::layout_helper_needs_slow_path(lh)) {
>  dc:   f6 c1 01                test   $0x1,%cl        ; layout helper *is* size?
>  df:   0f 85 9b 00 00 00       jne    180
>#      s = lh >> LogHeapWordSize;  // deliver size scaled by wordSize
>  e5:   89 c8                   mov    %ecx,%eax
>  e7:   c1 f8 03                sar    $0x3,%eax       ; this is object size now
>#    p += oop(p)->size();
>  ea:   48 98                   cltq
>  ec:   4d 8d 34 c6             lea    (%r14,%rax,8),%r14
>  f0:   49 8b 44 24 38          mov    0x38(%r12),%rax
>#  while (p < top()) {
>...
>#    cl->do_object(oop(p));
>...
> 103:   ff 10                   callq  *(%rax)
>```

### 6.3\.  指定虚调用和接口调用的目标

当运行时系统想要调用对象实例的虚方法和接口方法的时候，它需要确定目标方法在哪里。虽然大部分场景[可以被优化](https://shipilev.net/blog/2015/black-magic-method-dispatch/)，但是仍然有[需要做](https://www.jianshu.com/p/704fce44840f)分发的场景。分发的性能也依赖类元数据的获取，所以这不能被忽略。

### 6.4\. 观察： 压缩引用影响对象头部的内存占用

与 JVM 位数影响 mark word 大小类似，压缩引用的模式可以会影响对象大小，*即使是不考虑引用字段的情况下*。为了展示这个问题，让我们分别在小（1GB）大（64GB）两种堆内存下测试 `java.lang.Integer`。默认情况下小堆的压缩引用会打开，大堆的会关闭。这也意味着压缩类指针也会对应打开和关闭。

```
$ jdk8-64/bin/java -Xmx1g -jar jol-cli.jar internals java.lang.Integer
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via public java.lang.Integer(int)

java.lang.Integer object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00 # Mark word
      4     4        (object header)              00 00 00 00 # Mark word
      8     4        (object header)              de 21 00 20 # Class word
     12     4    int Integer.value                0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

```
$ jdk8-64/bin/java -Xmx64g -jar jol-cli.jar internals java.lang.Integer
# Running 64-bit HotSpot VM.

Instantiated the sample instance via public java.lang.Integer(int)

java.lang.Integer object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00 # Mark word
      4     4        (object header)              00 00 00 00 # Mark word
      8     4        (object header)              40 69 25 ad # Class word
     12     4        (object header)              e5 7f 00 00 # (uncompressed)
     16     4    int Integer.value                0
     20     4        (loss due to the next object alignment)
Instance size: 24 bytes # AHHHHHHH....
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

在 1GB 堆内存的 VM 中，对象头部占用 8 (mark word) + 4 (class word) = 12 字节，对应的 64GB VM 占用 8 (mark word) and + 8 (class word) = 16 字节。如果没有字段，由于对象以 8 字节对齐，那么两者都会向上舍入至 16 字节，但是在测试中*存在*一个 `int` 字段，所以在 64GB 的情况下，在 16 字节的头部之后需要再分配 8 字节，总共占用 24 字节。

## 7\. 头部：数值长度

数组还需要另外一种元数据：数组长度。由于对象类型仅仅编码数组元素类型，我们需要在另外一个地方存储数组长度。

可以从 [JOLSample_25_ArrayAlignment](https://hg.openjdk.java.net/code-tools/jol/file/tip/jol-samples/src/main/java/org/openjdk/jol/samples/JOLSample_25_ArrayAlignment.java#l41) 观察到：

```
$ jdk8-64/bin/java -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_25_ArrayAlignment
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

[J object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              d8 0c 00 00  # Class word
     12     4        (object header)              00 00 00 00  # Array length
     16     0   long [J.<elements>                N/A
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

...

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 07 00 00  # Class word
     12     4        (object header)              00 00 00 00  # Array length
     16     0   byte [B.<elements>                N/A
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 07 00 00  # Class word
     12     4        (object header)              01 00 00 00  # Array length
     16     1   byte [B.<elements>                N/A
     17     7        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 07 00 00  # Class word
     12     4        (object header)              02 00 00 00  # Array length
     16     2   byte [B.<elements>                N/A
     18     6        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 6 bytes external = 6 bytes total

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 07 00 00  # Class word
     12     4        (object header)              03 00 00 00  # Array length
     16     3   byte [B.<elements>                N/A
     19     5        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 5 bytes external = 5 bytes total

...

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 07 00 00  # Class word
     12     4        (object header)              08 00 00 00  # Array length
     16     8   byte [B.<elements>                N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

在 +12 的位置保存的就是数组长度。随着我们分配 0..8 个元素的 `byte[]` 数组，该位置的数据也相应的变化。在数组实例中保存数组长度有利于对象遍历时计算对象大小（前一节讨论过普通对象），另外也有利于高效进行范围检查。

>**DDIQ： 向我们展示一下如何进行数组范围检查？**
>
>在很多场景中，范围检查可以被消除，比如说在热循环中，但是对于数组未知的情况：
>
>```
>private int[] a = new int[100];
>
>@CompilerControl(CompilerControl.Mode.DONT_INLINE)
>@Benchmark
>public int test() {
>  return a[42];
>}
>```
>
>```
> mov    0x10(%rsi),%r10    ; get field "a"
> mov    0x10(%r10),%r11d   ; get a.<arraylength>, at 0x10
> cmp    $0x2a,%r11d        ; compare 42 with arraylength
> jbe    0x00007f139b4398e1 ; equal or greater? jump to slowpath
> mov    0xc0(%r10),%eax    ; read element at (24 + 4*42) = 0xc0
>```

### 7.1\. 观察：数组起始位置是对齐的

上面的例子掩盖了数组布局中重要的问题，被 64位模式下的对齐隐藏了。如果我们以较大的堆运行（或者显式关闭压缩引用）来干扰对齐方式：

```
$ jdk8-64/bin/java -Xmx64g -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_25_ArrayAlignment
# Running 64-bit HotSpot VM.

[J object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              d8 8c b0 a4  # Class word
     12     4        (object header)              98 7f 00 00  # Class word
     16     4        (object header)              00 00 00 00  # Array length
     20     4        (alignment/padding gap)
     24     0   long [J.<elements>                N/A
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total

...

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              00 00 00 00  # Mark word
      8     4        (object header)              68 87 b0 a4  # Class word
     12     4        (object header)              98 7f 00 00  # Class word
     16     4        (object header)              05 00 00 00  # Array length
     20     4        (alignment/padding gap)
     24     5   byte [B.<elements>                N/A
     29     3        (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 4 bytes internal + 3 bytes external = 7 bytes total
...
```

…或者以 32位运行：

```
$ jdk8-32/bin/java  -cp jol-samples.jar org.openjdk.jol.samples.JOLSample_25_ArrayAlignment
# Running 32-bit HotSpot VM.

[J object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              88 47 1b a3  # Class word
      8     4        (object header)              00 00 00 00  # Array length
     12     4        (alignment/padding gap)
     16     0   long [J.<elements>                N/A
Instance size: 16 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total

[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00  # Mark word
      4     4        (object header)              58 44 1b a3  # Class word
      8     4        (object header)              05 00 00 00  # Array length
     12     5   byte [B.<elements>                N/A
     17     7        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total
```
基于实现上的问题，*array base*[以机器字长对齐](https://bugs.openjdk.java.net/browse/JDK-8139457)。如果数组的元素比机器字长大，那么也会尽可能对齐，我们稍后将会详细讨论字段对齐。这意味着数组可能比我们想当然的占用更多空间。

## 8\. 对象对齐

到目前为止，我们暂且忽略了对象对齐的需求，里说当然的声称以 8 字节对齐。那么为什么是 8 字节呢？

有许多因素是的 8 字节对齐很合理。

第一，有时候我们需要原子性更新 mark word，这就为 mark word 的位置增加了限制。对于需要完整更新的 8 字节 mark word —— 例如设置移动指针 —— 就是需要以 8 字节对齐。由于 mark word 位于对象开头，所以整个对象也应该以 8 字节对齐。

>**DDIQ: 我们可以在 32 位平台让对象以 4 字节对齐么？**
>
>就 mark word 而言，可以。但是这并不是我们唯一需要考虑的因素，请看下面的讨论。

第二，对 volatile long/double 的原子性访问也一样，必须对它们进行不可分割的读写。即使没有 volatile 修饰词，也可能由于使用场景需要进行原子性访问，例如通过 `VarHandles`。因此，我们最好接受每个字段必须自然对齐。如果我们*在外部*以 8 对齐对象，那么*在内部*以 8/4/2 对齐字段都不会打破绝对对齐。

>**DDIQ： 这意味着我们可以查看对象字段的定义，然后决定对象应该采取哪种对齐？**
>
>是的，从技术上来讲可以。如果我们解决了 mark word 的对齐问题，并且只有 4 字节的字段，那么我们可以以 4 字节对齐对象。然后这会使分配逻辑很复杂：他需要立即决定是否需要更大的对齐（可以静态执行，由于分配的类型已知），是否需要增加外部的填充（需要动态检查，因为这取决于前面的对象）。这也为堆解析引入了问题。

以 8 字节对齐并不总是一种浪费，因为这使得超过 4GB 的堆可以进行压缩引用。以 4 字节对齐“仅仅”可以使 16GB 的堆进行压缩引用，而 8 字节对齐就可以使 32GB 的堆进行压缩引用。实际上为了扩展压缩引用生效的范围，可以[增加对象对齐至 16 字节](https://www.jianshu.com/p/66eb973d0bdf)。

在 Hotspot 中，从技术上来说对齐时对象自身的一部分：如果我们将*所有对象大小*舍入至 8，那么自然会在某些对象的末端出现*对齐阴影*。分配大小是 8 的倍数的对象不会打破对齐，所以如果我们从正确的起始位置开始（是的，我们可以），那么所有对象都可以保证是对齐的。

让我们以 `java.util.ArrayList` 为例：

```
$ jdk8-64/bin/java -Xmx1g -XX:ObjectAlignmentInBytes=16 -jar jol-cli.jar internals java.util.ArrayList
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

java.util.ArrayList object internals:
 OFFSET  SIZE                 TYPE DESCRIPTION                  VALUE
      0     4                      (object header)              01 00 00 00
      4     4                      (object header)              00 00 00 00
      8     4                      (object header)              46 2e 00 20
     12     4                  int AbstractList.modCount        0
     16     4                  int ArrayList.size               0
     20     4   java.lang.Object[] ArrayList.elementData        []
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

…以  `-XX:ObjectAlignmentInBytes=16` 执行：

```
$ jdk8-64/bin/java -Xmx1g -XX:ObjectAlignmentInBytes=16 -jar jol-cli.jar internals java.util.ArrayList
# Running 64-bit HotSpot VM.
# Using compressed oop with 4-bit shift.
# Using compressed klass with 4-bit shift.
# Objects are 16 bytes aligned.

Instantiated the sample instance via default constructor.

java.util.ArrayList object internals:
 OFFSET  SIZE                 TYPE DESCRIPTION                  VALUE
      0     4                      (object header)              01 00 00 00
      4     4                      (object header)              00 00 00 00
      8     4                      (object header)              93 2e 00 20
     12     4                  int AbstractList.modCount        0
     16     4                  int ArrayList.size               0
     20     4   java.lang.Object[] ArrayList.elementData        []
     24     8                      (loss due to the next object alignment)
Instance size: 32 bytes
```

以 8 字节对齐，`ArrayList` 占用 24 字节，因为对象大小是 8 的倍数。以 16 字节对齐，我们看到了*对齐阴影*：对象末尾丢失了 8 字节以维护下一个对象的对齐。

### 8.1\. 观察： 对齐阴影中的隐藏字段

这种观察立即引出了一个明确的观察：如果某个对象存在对齐阴影，那么我们可以在其中隐藏新字段，*而且不会增加对象的大小*！

比较 `java.lang.Object`：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Object
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              a8 0e 00 00
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

…与 `java.lang.Integer`：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Integer
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via public java.lang.Integer(int)

java.lang.Integer object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                   VALUE
      0     4        (object header)               01 00 00 00
      4     4        (object header)               00 00 00 00
      8     4        (object header)               f0 0e 01 00
     12     4    int Integer.value                 0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

`Object` 有一个 4 字节的对齐阴影，`Integer.value` 字段欣然占用了该部分。最后，`Object` 和 `Integer` 的大小在该 VM 配置下是相同的。

### 8.2\. 观察： 添加小字段就能让实例大小显著增加

这个故事有相反的告诫。假如对象没有对齐阴影：

```
public class A {
  int a1;
}
```

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . A
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                   VALUE
      0     4        (object header)               01 00 00 00
      4     4        (object header)               00 00 00 00
      8     4        (object header)               28 b8 0f 00
     12     4    int A.a1                          0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

如果添加一个 `boolean` 字段会怎样呢？

```
public class B {
  int b1;
  boolean b2; // takes 1 byte, right?
}
```

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . B
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

B object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              28 b8 0f 00
     12     4       int B.b1                         0
     16     1   boolean B.b2                         false
     17     7           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total
```

在这里，我们只需要一个糟糕的字节来分配字段，但是因为我们需要满足对象对齐的要求，我们最终增加了整整 8 字节！有一个小小的安慰，在阴影中剩下的 7 字节添加更多字段将不会增加表面上的对象大小。

## 9\. 字段对齐

在讨论对象对齐的时候，我们已经在前面的章节讨论了该主题。

很多架构不喜欢未对齐的访问。在很多情况下，未对齐访问将会带来性能损失。在有些情况下，未对齐访问将会引发机器异常。然后 Java 内存模型来了，它要求对字段和数组元素进行原子访问，[至少](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.7)是在字段声明为 `volatile` 的情况下。

这迫使大多数实现将字段对齐为自然对齐。对象以 8 字节对齐，这就保证了起始偏移 0 是以 8 字节对齐的，这是所有类型中最大的自然对齐。所以我们“仅仅”需要以自然对齐布局对象中的字段即可。这可以在 `java.lang.Long` 中清楚的看到：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Long
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via public java.lang.Long(long)

java.lang.Long object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 01 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 18 11 01 00
     12     4        (alignment/padding gap)
     16     8   long Long.value                      0
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

在这里 `long value` 放在 +16 的位置，这样就能使它以 8 对齐。注意在这个字段之前有一个空白！

### 9.1\. 观察： 字段对齐空白中的隐藏字段

预告一下字段打包的讨论：由于存在这些字段对齐空白，所以可以在其中隐藏字段。例如，可以在包含 `long` 类中添加另外一个 `int` 字段：

```
public class LongIntCarrier {
  long value;
  int somethingElse;
}
```

…最终对象布局是这样：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . LongIntCarrier
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

LongIntCarrier object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 01 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 28 b8 0f 00
     12     4    int LongIntCarrier.somethingElse    0
     16     8   long LongIntCarrier.value            0
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

对比 `java.lang.Long` 的布局：它们占用了同样多的实例空间，因为新的的 `int` 字段使用了对齐空白。

## 10\. 字段打包

存在多个字段时，将会出现一个新问题：如何在对象中分布字段？这就产生了字段布局器。字段布局器使得每个字段以自然对齐分配，并且尽可能地密实打包。如何准确的实现这一目标很大程度上取决于实现。我们都知道，字段“打包器”可以以声明的顺序放置字段，然后以每个字段的自然对齐填充。当然这会浪费很多内存。

考虑这个类：

```
public class FieldPacking {
  boolean b;
  long l;
  char c;
  int i;
}
```

幼稚的字段打包器可以这样做：

```
$ <32-bit simulation>
FieldPacking object internals:
 OFFSET  SIZE      TYPE DESCRIPTION
      0     4           (object header)
      4     4           (object header)
      8     1   boolean FieldPacking.b
      9     7           (alignment/padding gap)
     16     8      long FieldPacking.l
     24     2      char FieldPacking.c
     26     2            (alignment/padding gap)
     28     4       int FieldPacking.i
Instance size: 32 bytes
```

…然后聪明的打包器将会这样做：

```
$ jdk8-32/bin/java -jar jol-cli.jar internals -cp . FieldPacking
# Running 32-bit HotSpot VM.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

FieldPacking object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              68 91 6f a3
      8     8      long FieldPacking.l               0
     16     4       int FieldPacking.i               0
     20     2      char FieldPacking.c
     22     1   boolean FieldPacking.b               false
     23     1           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 1 bytes external = 1 bytes total
```

… 这样每个对象实例就节省了 8 字节。

>**DDIQ: 字段布局有经验法则么？**
>
>如上所述，字段布局是实现细节。直到最近，Hotspot 的实现还是一个几乎线性的实现。它从大到小布局字段。首先布局 longs/double（需要以 8 对齐），然后是 ints/floats（需要以 4 对齐），然后是 chars/shorts（需要以 2 对齐），最后是 bytes/booleans。以这种方式，我们可以很紧凑的打包整个字段块，但是有一个异常情况：较大数据类型的初始对齐可能会留下小数据类型可以占用的空白 —— 这种情况分开处理。
>
>引用字段要么当做 8 字节字段（没有压缩引用的 64 位模式）处理，要么当做 4 字节字段（32 位模式，或者有压缩引用的 64 位模式）处理。当多个带有引用字段的类是有继承关系时，有一些 GC 相关的技巧：有时将它们聚类在一起可能有好处。

无论如何，我们可以从中得出两个直接的观察。

### 10.1\. 观察： 字段声明顺序 != 字段布局顺序

首先，给定字段声明顺序：

```
public class FieldOrder {
  boolean firstField;
  long secondField;
  char thirdField;
  int fourthField;
}
```

… 这并不保证在内存中是相同的顺序。字段打包器将会重新排列字段以最小化内存占用：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . FieldOrder
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

FieldOrder object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              28 b8 0f 00
     12     4       int FieldOrder.fourthField       0
     16     8      long FieldOrder.secondField       0
     24     2      char FieldOrder.thirdField
     26     1   boolean FieldOrder.firstField        false
     27     5           (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 5 bytes external = 5 bytes total
```

注意看布局器如何按照数据类型大小安排字段：首先 `long` 字段对齐在 +16，然后 `int` 字段应该放在 +24 的问题，但是布局器发现 `long` 字段前面有个空白可以使用，所以它就被放在了 +12，然后 `char` 字段按照自然对齐放在 +24，最后是 boolean 字段放在 +26。

当你想要与基于偏移量访问字段的外部/原始函数交互时，字段打包是一个主要的问题。字段的偏移量依赖字段打包器的实现（是否压缩字段，具体如何处理？），以及运行的环境条件（机器位数，压缩引用模式，对象对齐，等）。

使用 `sun.misc.Unsafe` 访问字段的 Java 代码必须在运行时读取字段偏移量，以获取运行时的实际布局。 假设这些字段与调试会话中的偏移量相同，那么就很难诊断出错误的来源。

### 10.2\. 观察： C 样式的填充不可靠

当涉及到 [False Sharing](https://en.wikipedia.org/wiki/False_sharing) 消除时，人们诉诸于填充关键字段，以实现隔离在自身缓存行的目的。最常见的方法是在被保护字段周围添加一些虚字段声明。而且由于写入这些声明很乏味，所以人们倾向于使用较大的数据类型。所以为了保护被争用的 `byte` 字段，你会看到这种写法：

```
public class LongPadding {
  long l01, l02, l03, l04, l05, l06, l07, l08; // 64 bytes
  byte pleaseHelpMe;
  long l11, l12, l13, l14, l15, l16, l17, l18; // 64 bytes
}
```

你可能期待 `pleaseHelpMe` 字段被两个 `long` 字段块包围。很不幸，字段打包器不这么想：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . CStylePadding
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

LongPadding object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              28 b8 0f 00
     12     1   byte LongPadding.pleaseHelpMe     0  # WHOOPS.
     13     3        (alignment/padding gap)
     16     8   long LongPadding.l01              0
     24     8   long LongPadding.l02              0
     32     8   long LongPadding.l03              0
     40     8   long LongPadding.l04              0
     48     8   long LongPadding.l05              0
     56     8   long LongPadding.l06              0
     64     8   long LongPadding.l07              0
     72     8   long LongPadding.l08              0
     80     8   long LongPadding.l11              0
     88     8   long LongPadding.l12              0
     96     8   long LongPadding.l13              0
    104     8   long LongPadding.l14              0
    112     8   long LongPadding.l15              0
    120     8   long LongPadding.l16              0
    128     8   long LongPadding.l17              0
    136     8   long LongPadding.l18              0
Instance size: 144 bytes
Space losses: 3 bytes internal + 0 bytes external = 3 bytes total
```

使用 `byte` 字段填充会怎样呢？这取决于实现细节，字段打包器以声明顺序处理相同大小/类型的字段，但至少它会起作用：

```
public class BytePadding {
  byte p000, p001, p002, p003, p004, p005, p006, p007;
  byte p008, p009, p010, p011, p012, p013, p014, p015;
  byte PRNG, p017, p018, p019, p020, p021, p022, p023;
  byte p024, p025, p026, p027, p028, p029, p030, p031;
  byte p032, p033, p034, p035, p036, p037, p038, p039;
  byte p040, p041, p042, p043, p044, p045, p046, p047;
  byte p048, p049, p050, p051, p052, p053, p054, p055;
  byte p056, p057, p058, p059, p060, p061, p062, p063;
  byte pleaseHelpMe;
  byte p100, p101, p102, p103, p104, p105, p106, p107;
  byte p108, p109, p110, p111, p112, p113, p114, p115;
  byte p116, p117, p118, p119, p120, p121, p122, p123;
  byte p124, p125, p126, p127, p128, p129, p130, p131;
  byte p132, p133, p134, p135, p136, p137, p138, p139;
  byte p140, p141, p142, p143, p144, p145, p146, p147;
  byte p148, p149, p150, p151, p152, p153, p154, p155;
  byte p156, p157, p158, p159, p160, p161, p162, p163;
}
```

```
$ jdk8-64/bin/java -jar ~/projects/jol/jol-cli/target/jol-cli.jar internals -cp . BytePadding
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

BytePadding object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              28 b8 0f 00
     12     1   byte BytePadding.p000             0
     13     1   byte BytePadding.p001             0
...
     74     1   byte BytePadding.p062             0
     75     1   byte BytePadding.p063             0
     76     1   byte BytePadding.pleaseHelpMe     0 # Good
     77     1   byte BytePadding.p100             0
     78     1   byte BytePadding.p101             0
...
    139     1   byte BytePadding.p162             0
    140     1   byte BytePadding.p163             0
    141     3        (loss due to the next object alignment)
Instance size: 144 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total
```

… 除非你需要保护一个不同类型的字段：

```
public class BytePaddingHetero {
  byte p000, p001, p002, p003, p004, p005, p006, p007;
  byte p008, p009, p010, p011, p012, p013, p014, p015;
  byte p016, p017, p018, p019, p020, p021, p022, p023;
  byte p024, p025, p026, p027, p028, p029, p030, p031;
  byte p032, p033, p034, p035, p036, p037, p038, p039;
  byte p040, p041, p042, p043, p044, p045, p046, p047;
  byte p048, p049, p050, p051, p052, p053, p054, p055;
  byte p056, p057, p058, p059, p060, p061, p062, p063;
  byte pleaseHelpMe;
  int pleaseHelpMeToo; // pretty please!
  byte p100, p101, p102, p103, p104, p105, p106, p107;
  byte p108, p109, p110, p111, p112, p113, p114, p115;
  byte p116, p117, p118, p119, p120, p121, p122, p123;
  byte p124, p125, p126, p127, p128, p129, p130, p131;
  byte p132, p133, p134, p135, p136, p137, p138, p139;
  byte p140, p141, p142, p143, p144, p145, p146, p147;
  byte p148, p149, p150, p151, p152, p153, p154, p155;
  byte p156, p157, p158, p159, p160, p161, p162, p163;
}
```

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . BytePaddingHetero
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

BytePaddingHetero object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                        VALUE
      0     4        (object header)                    01 00 00 00
      4     4        (object header)                    00 00 00 00
      8     4        (object header)                    28 b8 0f 00
     12     4    int BytePaddingHetero.pleaseHelpMeToo  0 # WHOOPS.
     16     1   byte BytePaddingHetero.p000             0
     17     1   byte BytePaddingHetero.p001             0
...
     78     1   byte BytePaddingHetero.p062             0
     79     1   byte BytePaddingHetero.p063             0
     80     1   byte BytePaddingHetero.pleaseHelpMe     0 # Good.
     81     1   byte BytePaddingHetero.p100             0
     82     1   byte BytePaddingHetero.p101             0
...
    143     1   byte BytePaddingHetero.p162             0
    144     1   byte BytePaddingHetero.p163             0
    145     7        (loss due to the next object alignment)
Instance size: 152 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total
```

### 10.3\. @Contended

JDK 库是通过引入[私有的 @Contended 注解](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/java.base/share/classes/jdk/internal/vm/annotation/Contended.java)来消除这个性能敏感问题的。它在 JDK 中使用的也不多，例如在 `java.lang.Thread` 中保存[线程本地的随机数生成器状态](http://hg.openjdk.java.net/jdk/jdk/file/19afeaa0fdbe/src/java.base/share/classes/java/lang/Thread.java#l2059)：

```
public class Thread implements Runnable {
    ...
    // The following three initially uninitialized fields are exclusively
    // managed by class java.util.concurrent.ThreadLocalRandom. These
    // fields are used to build the high-performance PRNGs in the
    // concurrent code, and we can not risk accidental false sharing.
    // Hence, the fields are isolated with @Contended.

    /** The current seed for a ThreadLocalRandom */
    @jdk.internal.vm.annotation.Contended("tlr")
    long threadLocalRandomSeed;

    /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @jdk.internal.vm.annotation.Contended("tlr")
    int threadLocalRandomProbe;

    /** Secondary seed isolated from public ThreadLocalRandom sequence */
    @jdk.internal.vm.annotation.Contended("tlr")
    int threadLocalRandomSecondarySeed;
    ...
}
```

… 这使得字段布局器对它们特殊处理：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals java.lang.Thread
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.

Instantiated the sample instance via default constructor.

java.lang.Thread object internals:
 OFFSET  SIZE         TYPE DESCRIPTION                             VALUE
      0     4              (object header)                         01 00 00 00
      4     4              (object header)                         00 00 00 00
      8     4              (object header)                         48 69 00 00
     12     4          int Thread.priority                         5
     16     8         long Thread.eetop                            0
...
     96     4   j.l.Object Thread.blockerLock                      (object)
    100     4      j.l.UEH Thread.uncaughtExceptionHandler         null
    104   128              (alignment/padding gap)
    232     8         long Thread.threadLocalRandomSeed            0
    240     4          int Thread.threadLocalRandomProbe           0
    244     4          int Thread.threadLocalRandomSecondarySeed   0
    248   128              (loss due to the next object alignment)
Instance size: 376 bytes
Space losses: 129 bytes internal + 128 bytes external = 257 bytes total
```

>**DDIQ： 为什么 `@Contended` 不是一个公开的注解？**
>
>允许使用者构建巨大的“普通”对象存在安全/可靠性上的影响。我们就谈到这里吧。

通过其它实现细节中的捎带，不依赖内部注解也可以实现同样的效果，我们接下来将会讨论。

## 11\. 层次结构中的字段布局

层次结构中的字段布局需要特殊考虑。假如我们有这些类：

```
public class Hierarchy {
  static class A {
    int a;
  }
  static class B extends A {
    int b;
  }
  static class C extends A {
    int c;
  }
}
```

这些类的层次结构将会像这样：

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . Hierarchy\$A
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Hierarchy$A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              28 b8 0f 00
     12     4    int A.a                          0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

Hierarchy$B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              08 ba 0f 00
     12     4    int A.a                          0
     16     4    int B.b                          0
     20     4        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

Hierarchy$C object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              08 ba 0f 00
     12     4    int A.a                          0
     16     4    int C.c                          0
     20     4        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

注意：所有的类的 `A.a` 父类字段都在同样的位置。这使得可以从 `A` 的任意子类直接转为 `A`，然后在不检查 `A` 的实际类型的情况下访问 `a` 字段。也就是说，无论操作的是 `A`、`B` 或 `C` 的实例，对于 `((A)o).a` 总是访问同样的偏移量。

这看起来像*总是首先处理父类字段*。这意味着父类字段总是位于层次结构最前面么？这是一个实现细节：在 JDK 15 之前，答案是“对的”；在 JDK15 之后，答案是“不对”。我们将会通过一些观察对它进行量化。

### 11.1\. 父类空白

在 JDK 15 之前，字段布局器仅仅对当前类局部声明的字段生效。这意味着如果存在子类字段可以占用的父类空白，它们[不会使用](https://bugs.openjdk.java.net/browse/JDK-8024913)。让我们将前面的 `LongIntCarrier` 分割为子类：

```
public class LongIntCarrierSubs {
  static class A {
    long value;
  }
  static class B extends A {
    int somethingElse;
  }
}
```

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . LongIntCarrierSubs\$B
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

LongIntCarrierSubs$B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                  VALUE
      0     4        (object header)              01 00 00 00
      4     4        (object header)              00 00 00 00
      8     4        (object header)              08 ba 0f 00
     12     4        (alignment/padding gap)
     16     8   long A.value                      0
     24     4    int B.somethingElse              0
     28     4        (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 4 bytes internal + 4 bytes external = 8 bytes total
```

可以看到同样由 `long` 对齐造成的空白。理论上来说，`B.somethingElse` 可以使用这部分空间，但是字段布局器的实现使得这不可能。因此，我们将 `B` 字段布局在 `A` 字段后，这浪费了 8 字节。

### 11.2\. 层次空白

JDK 15 之前的另外一个怪事是，字段布局器以*引用大小*的整数单位对字段块进行计数，这使得子类字段块[从更远的偏移开始](https://bugs.openjdk.java.net/browse/JDK-8024912)。带有很多小字段的场景比较明显：

```
public class ThreeBooleanStooges {
  static class A {
    boolean a;
  }
  static class B extends A {
    boolean b;
  }
  static class C extends B {
    boolean c;
  }
}
```

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . ThreeBooleanStooges\$A
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . ThreeBooleanStooges\$B
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . ThreeBooleanStooges\$C
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

ThreeBooleanStooges$A object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              28 b8 0f 00
     12     1   boolean A.a                          false
     13     3           (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total

ThreeBooleanStooges$B object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              08 ba 0f 00
     12     1   boolean A.a                          false
     13     3           (alignment/padding gap)
     16     1   boolean B.b                          false
     17     7           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 3 bytes internal + 7 bytes external = 10 bytes total

ThreeBooleanStooges$C object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              e8 bb 0f 00
     12     1   boolean A.a                          false
     13     3           (alignment/padding gap)
     16     1   boolean B.b                          false
     17     3           (alignment/padding gap)
     20     1   boolean C.c                          false
     21     3           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 6 bytes internal + 3 bytes external = 9 bytes total
```

损失非常显著！每个类实例浪费了 3 字节，然后由于对象对齐有损失了一部分。

在更大的堆或者没有压缩引用的情况下，情况更严重。

```
$ jdk8-64/bin/java -Xmx64g -jar jol-cli.jar internals -cp . ThreeBooleanStooges\$C
# Running 64-bit HotSpot VM.

Instantiated the sample instance via default constructor.

ThreeBooleanStooges$C object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              01 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              b0 89 aa 37
     12     4           (object header)              b0 7f 00 00
     16     1   boolean A.a                          false
     17     7           (alignment/padding gap)
     24     1   boolean B.b                          false
     25     7           (alignment/padding gap)
     32     1   boolean C.c                          false
     33     7           (loss due to the next object alignment)
Instance size: 40 bytes
Space losses: 14 bytes internal + 7 bytes external = 21 bytes total
```

### 11.3\. 观察：层次结构填充技巧

该实现的特殊性允许构造一个相当奇怪的填充技巧，相对于 C 样式的填充更具有弹性。

```
public class HierarchyLongPadding {
  static class Pad1 {
    long l01, l02, l03, l04, l05, l06, l07, l08;
  }
  static class Carrier extends Pad1 {
    byte pleaseHelpMe;
  }
  static class Pad2 extends Carrier {
    long l11, l12, l13, l14, l15, l16, l17, l18;
  }
  static class UsableObject extends Pad2 {};
}
```

…生成:

```
$ jdk8-64/bin/java -jar jol-cli.jar internals -cp . HierarchyLongPadding\$UsableObject
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

HierarchyLongPadding$UsableObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 01 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 c8 bd 0f 00
     12     4        (alignment/padding gap)
     16     8   long Pad1.l01                        0
     24     8   long Pad1.l02                        0
     32     8   long Pad1.l03                        0
     40     8   long Pad1.l04                        0
     48     8   long Pad1.l05                        0
     56     8   long Pad1.l06                        0
     64     8   long Pad1.l07                        0
     72     8   long Pad1.l08                        0
     80     1   byte Carrier.pleaseHelpMe            0
     81     7        (alignment/padding gap)
     88     8   long Pad2.l11                        0
     96     8   long Pad2.l12                        0
    104     8   long Pad2.l13                        0
    112     8   long Pad2.l14                        0
    120     8   long Pad2.l15                        0
    128     8   long Pad2.l16                        0
    136     8   long Pad2.l17                        0
    144     8   long Pad2.l18                        0
Instance size: 152 bytes
Space losses: 11 bytes internal + 0 bytes external = 11 bytes total
```

请看，我们利用了一个怪异的实现细节，将要保护的字段包围在两个类之间，

### 11.4\. Java 15+ 中的层次空白

现在我们进入了 JDK 15，它的[字段布局策略进行了全面改革](https://bugs.openjdk.java.net/browse/JDK-8237767)。父类和层次结构的空白已经被消除。执行前面的例子将会显示：

```
$ jdk15-64/bin/java -jar jol-cli.jar internals -cp . LongIntCarrierSubs\$B
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

LongIntCarrierSubs$B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 05 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 4c 7d 17 00
     12     4    int B.somethingElse                 0
     16     8   long A.value                         0
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

最终，`B.somethingElse` 占用了父类 `A.value` 前面的对齐空白。

层次结构之间空白也没有了：

```
$ jdk15-64/bin/java -jar jol-cli.jar internals -cp . ThreeBooleanStooges\$C
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

ThreeBooleanStooges$C object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                  VALUE
      0     4           (object header)              05 00 00 00
      4     4           (object header)              00 00 00 00
      8     4           (object header)              90 7d 17 00
     12     1   boolean A.a                          false
     13     1   boolean B.b                          false
     14     1   boolean C.c                          false
     15     1           (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 1 bytes external = 1 bytes total
```

完美！

### 11.5\. 观察： 层次结构填充技巧在 JDK 15 中失效了

很不幸，这使得幼稚的层次结构填充技巧失效了！请看：

```
$ jdk15-64/bin/java -jar jol-cli.jar internals -cp . HierarchyLongPadding\$UsableObject
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

HierarchyLongPadding$UsableObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 05 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 08 7c 17 00
     12     1   byte Carrier.pleaseHelpMe            0  # WHOOPS
     13     3        (alignment/padding gap)
     16     8   long Pad1.l01                        0
     24     8   long Pad1.l02                        0
     32     8   long Pad1.l03                        0
     40     8   long Pad1.l04                        0
     48     8   long Pad1.l05                        0
     56     8   long Pad1.l06                        0
     64     8   long Pad1.l07                        0
     72     8   long Pad1.l08                        0
     80     8   long Pad2.l11                        0
     88     8   long Pad2.l12                        0
     96     8   long Pad2.l13                        0
    104     8   long Pad2.l14                        0
    112     8   long Pad2.l15                        0
    120     8   long Pad2.l16                        0
    128     8   long Pad2.l17                        0
    136     8   long Pad2.l18                        0
Instance size: 144 bytes
Space losses: 3 bytes internal + 0 bytes external = 3 bytes total
```

现在 `pleaseHelpMe` 占用了父类中的空白，字段布局器将它提取出来了。哎呀。

我感觉唯一的解决方法是填充最小的数据类型：

```
public class HierarchyBytePadding {
  static class Pad1 {
    byte p000, p001, p002, p003, p004, p005, p006, p007;
    byte p008, p009, p010, p011, p012, p013, p014, p015;
    byte p016, p017, p018, p019, p020, p021, p022, p023;
    byte p024, p025, p026, p027, p028, p029, p030, p031;
    byte p032, p033, p034, p035, p036, p037, p038, p039;
    byte p040, p041, p042, p043, p044, p045, p046, p047;
    byte p048, p049, p050, p051, p052, p053, p054, p055;
    byte p056, p057, p058, p059, p060, p061, p062, p063;
  }

  static class Carrier extends Pad1 {
    byte pleaseHelpMe;
  }

  static class Pad2 extends Carrier {
    byte p100, p101, p102, p103, p104, p105, p106, p107;
    byte p108, p109, p110, p111, p112, p113, p114, p115;
    byte p116, p117, p118, p119, p120, p121, p122, p123;
    byte p124, p125, p126, p127, p128, p129, p130, p131;
    byte p132, p133, p134, p135, p136, p137, p138, p139;
    byte p140, p141, p142, p143, p144, p145, p146, p147;
    byte p148, p149, p150, p151, p152, p153, p154, p155;
    byte p156, p157, p158, p159, p160, p161, p162, p163;
  }

  static class UsableObject extends Pad2 {};
}
```

… 这填满了所有空白，不会使受保护的字段移动：

```
$ jdk15-64/bin/java -jar jol-cli.jar internals -cp . HierarchyBytePadding\$UsableObject
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.

Instantiated the sample instance via default constructor.

HierarchyBytePadding$UsableObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                     VALUE
      0     4        (object header)                 05 00 00 00
      4     4        (object header)                 00 00 00 00
      8     4        (object header)                 08 7c 17 00
     12     1   byte Pad1.p000                       0
     13     1   byte Pad1.p001                       0
...
     74     1   byte Pad1.p062                       0
     75     1   byte Pad1.p063                       0
     76     1   byte Carrier.pleaseHelpMe            0  # GOOD
     77     1   byte Pad2.p100                       0
     78     1   byte Pad2.p101                       0
...
    139     1   byte Pad2.p162                       0
    140     1   byte Pad2.p163                       0
    141     3        (loss due to the next object alignment)
Instance size: 144 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total
```

实际上，这就是 [JMH 现在的做法](http://hg.openjdk.java.net/code-tools/jmh/rev/ee4b8b1f1523)。

这仍然依赖于实现细节，`Pad1` 中的字段将会首先被处理，并且填充父类中的空白。

## 12\. 总结

Java 对象内部很复杂，并且充满静态和动态的权衡折中。Java 对象大小可能根据内部因素改变，例如 JVM 位数、JVM 特性集合等。也可能根据运行时配置改变，例如堆内存大小、压缩引用模式、使用的 GC。

从 JVM 的角度看内存占用，可以看到压缩引用扮演者重要的角色。即使不涉及引用，这也影响 class word 是否压缩。mark word 在 32 位 VM 中会更紧凑，所以这也会改善内存占用。（这还没有提及 VM 本地指针和机器字长范围的类型将会更小）

从 Java（开发者）的角度来看，知道这些对象内部结构可以将字段隐藏在对象对齐阴影中，或者字段对齐空白中，而且不会增加实例的大小。另一方面，仅仅增加一个小字段也可能导致实例大小显著增加，要解释其中的原因将会不可避免的涉及对细粒度对象结构的分析。

最后，但并非最不重要的是，通过一些技巧使得对象布局器按照某种顺序放置是很困难的，这依赖一些实现细节。但是这仍然可用，there are less safer and more safer things to rely on. 无论如何，需要对每次 JDK 升级进行额外的验证。对于 JDK 15 和之后的版本，绝对应该重新验证。

* * *

[1]. 实际上，在 `Instrumentation.getObjectSize` 中有一个，它需要将其附加为 JavaAgent 执行。

[2]. 你仍然可以使用，构建一下，或者从某个地方获取一个 fastdebug 构建。例如[这里](https://builds.shipilev.net/)。

[3]. 通过一些 VM 消费已经使该方法可用，但是最终由于对 Unsafe 的外部依赖太多，而导致不可用。

[4]. 这*不*是对象地址（熵很低），而是某个内部 PRNG 的结果。

[5]. 在 Hotspot 中，`-XX:+CompressedKlassPointers` 基于 `-XX:-CompressedOops`，单这是实现上的限制，不是设计上的。理论上来说，你可以在压缩 oops 的情况下压缩 klass 指针，但是这又会让你维护一个配置。


Last updated 2020-04-21 10:50:39 CEST

