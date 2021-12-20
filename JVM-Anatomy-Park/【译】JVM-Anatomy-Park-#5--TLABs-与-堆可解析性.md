原文地址：[JVM Anatomy Park #5: TLABs and Heap Parsability](https://shipilev.net/jvm-anatomy-park/5-tlabs-and-heap-parsability/)


## 问题

你曾经遇到过无法解释的大`int[]`数组么？这些并没有分配的对象仍然会消耗堆内存么？这里面有需要回收的对象么？

## 理论

在 GC 理论中，可靠的收集器需要维护一个重要的属性——*堆的可解析性*，也就是将堆塑造成可以解析对象、字段等的形式，而且不需要复杂的元数据的支持。举个例子，在 OpenJDK 中许多遍历堆的分析任务就像这样，仅仅是一个简单的循环：

```
HeapWord* cur = heap_start;
while (cur < heap_used) {
  object o = (object)cur;
  do_object(o);
  cur = cur + o->size();
}
```

就是这个样子！如果堆是可解析的，那么我们可以假设已经分配的堆从头到尾是一个连续的对象流。严格来说这并不是一个必要属性，但是这个属性使得 GC 实现、测试和调试更简单。

考虑[线程本地分配缓冲区 (TLAB) 机制](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/)：现在每个线程有用于本地分配的 TLAB。从 GC 的角度理解，这意味着*整个* TLAB 都被分配了。GC 很难知道本地线程在做什么操作：它们正在移动 TLAB 指针？总之 TLAB 指针当前在哪里？可能线程只是在本地寄存器维护指针（虽然在 OpenJDK 中并不是这样），而外部系统是看不到这个数据的。所以这里就有个问题：外部系统**无法**精确得知 TLAB 内部的情况。

我们可以通过停止线程来中止 TLAB 分配，然后精确的遍历堆内存。但是有一个更方便的*技巧*：为什么我们不通过插入*填充对象*的办法使得堆内存可解析呢？也就是，如果我们有下述 TLAB：

```plain
 ...........|===================           ]............
            ^                  ^           ^
        TLAB start        TLAB used   TLAB end
```

我们可以停止线程，然后*请求线程*在 TLAB 的空闲部分中分配填充对象，使得这部分堆内存可解析：

```plain
 ...........|===================!!!!!!!!!!!]............
            ^                  ^           ^
        TLAB start        TLAB used   TLAB end
```

选什么作为填充对象呢？当然是长度可变的对象。为什么不选`int[]`数组呢？注意，填充这种对象仅仅是写一个数组的头部，使得遍历堆内存的机制可以工作，跳过原来空闲的部分。一旦线程恢复 TLAB 分配，那么仅仅复写填充部分即可，就像什么都没发生过。

顺便说一下，这也简化了*清理*堆内存的过程。如果我们需要清理对象，那么只需要在原处复写填充对象即可，这样就能保持堆内存可以被正常的遍历。

## 实验

我们可以在实际的例子中观察到这个机制么？当然可以。我们起很多线程分配自己的 TLAB，然后主线程持续分配对象耗尽 Java 堆内存，最终程序将会以 OutOfMemoryException 崩溃，这会触发堆转储（heap dump）。

工作负载就像这样：

```
import java.util.*;
import java.util.concurrent.*;

public class Fillers {
  public static void main(String... args) throws Exception {
    final int TRAKTORISTOV = 300;
    CountDownLatch cdl = new CountDownLatch(TRAKTORISTOV);
    for (int t = 0 ; t < TRAKTORISTOV; t++) {
      new Thread(() -> allocateAndWait(cdl)).start();
    }
    cdl.await();
    List<Object> l = new ArrayList<>();
    new Thread(() -> allocateAndDie(l)).start();
  }

  public static void allocateAndWait(CountDownLatch cdl) {
    Object o = new Object();  // Request a TLAB
    cdl.countDown();
    while (true) {
      try {
        Thread.sleep(1000);
      } catch (Exception e) {
        break;
      }
    }
    System.out.println(o); // Use the object
  }

  public static void allocateAndDie(Collection<Object> c) {
    while (true) {
      c.add(new Object());
    }
  }
}
```

为了控制 TLAB 的大小，我们继续使用 [Epsilon GC](http://openjdk.java.net/jeps/8174901)。启动参数为 `-Xmx1G -Xms1G -XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC -XX:+HeapDumpOnOutOfMemoryError`，这样程序将会很快崩溃，并且输出堆转储。

在 [Eclipse Memory Analyzer (MAT)](http://www.eclipse.org/mat/) 中打开这个堆转储——我非常喜欢这个工具——我们可以看到下述类直方图：

```plain
Class Name                                 |   Objects | Shallow Heap |
-----------------------------------------------------------------------
                                           |           |              |
int[]                                      |     1,099 |  814,643,272 |
java.lang.Object                           | 9,181,912 |  146,910,592 |
java.lang.Object[]                         |     1,521 |  110,855,376 |
byte[]                                     |     6,928 |      348,896 |
java.lang.String                           |     5,840 |      140,160 |
java.util.HashMap$Node                     |     1,696 |       54,272 |
java.util.concurrent.ConcurrentHashMap$Node|     1,331 |       42,592 |
java.util.HashMap$Node[]                   |       413 |       42,032 |
char[]                                     |        50 |       37,432 |
-----------------------------------------------------------------------
```

`int[]` 消耗了最多堆内存！这是我们的填充对象。当然，这个实验有一些注意事项。

首先，我们配置 Epsilon 具有静态的 TLAB 大小。高性能的收集器将会*自适应* TLAB 的大小，当线程已经分配了一些对象，但是仍然占用很多 TLAB 内存的时候，这种自适应的机制将会最小化无效内存的占用。这也是不要设置太大 TLAB 的一个原因。如果设置了较大的 TLAB ，那么在一个持续分配对象的线程中仍然可能观察到填充对象，但是这并不是真正的对象。

然后，我们通过配置 MAT 来展示不可达的对象。从定义上来说，填充对象是不可达的。它们出现在堆转储中仅仅是堆可解析属性的副作用。这些对象并不是*真的*存在，一个成熟的堆转储分析器将会为你过滤掉这些对象——这也就是 900 MB 对象就能耗尽 1G 堆内存的一个原因。

## 观察

TLAB 很有趣，堆可解析性也很有趣。将两者组合在一起就更有趣了，有时候会泄漏一些内部的机制。如果你遇到一些令人惊讶的现象，那么你可能是看到一些聪明的小技巧了！
