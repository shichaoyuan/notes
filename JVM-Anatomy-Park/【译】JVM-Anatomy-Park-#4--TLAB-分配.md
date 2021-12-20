原文地址：[JVM Anatomy Park #4: TLAB allocation](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/)

## 问题

TLAB 分配是什么？指针碰撞（Pointer-bump）分配又是什么？总之谁负责分配对象？

## 理论

当我们执行 `new MyClass()` 的时候，运行时环境将会为新建的对象分配内存。用于分配的 GC （内存管理器）示例接口非常简单：

```
 ref Allocate(T type);
 ref AllocateArray(T type, int size);
```

当然，由于编写内存管理器的语言通常与运行的语言不同（比如 JVM 运行 Java 语言，但是 HotSpot JVM 是用 C++ 编写的），所以代码中真实的接口比较晦涩。新建对象的 Java 代码需要转化为本地 VM 代码。这个过程的成本高么？可能会。内存管理器需要处理多线程并发申请内存的情况么？当然。

为了优化多线程并发申请的场景，我们一次性为线程分配整*块*内存，用完了之后再向 VM 申请新的一块。在 Hotspot 中这些内存块被称为线程本地分配缓冲区（TLABs），为了支持基于内存块的分配又构建了一套复杂的机制。从空间上看，TLABs 是在线程本地的，这意味着 TLAB 就是接受本地线程分配的缓冲区。TLAB 仍然是 Java 堆的一部分，线程可以将本地新建对象的引用传递给 TLAB 外部的字段，等等。

所有知名的 OpenJDK GCs 都支持 TLAB 分配。这部分 VM 代码抽象的相当好。所有的 Hotspot 编译器都支持 TLAB 分配，对象分配逻辑生成的机器码通常是这样的：

```
0x00007f3e6bb617cc: mov    0x60(%r15),%rax        ; TLAB "current"
0x00007f3e6bb617d0: mov    %rax,%r10              ; tmp = current
0x00007f3e6bb617d3: add    $0x10,%r10             ; tmp += 16 (object size)
0x00007f3e6bb617d7: cmp    0x70(%r15),%r10        ; tmp > tlab_size?
0x00007f3e6bb617db: jae    0x00007f3e6bb61807     ; TLAB is done, jump and request another one
0x00007f3e6bb617dd: mov    %r10,0x60(%r15)        ; current = tmp (TLAB is fine, alloc!)
0x00007f3e6bb617e1: prefetchnta 0xc0(%r10)        ; ...
0x00007f3e6bb617e9: movq   $0x1,(%rax)            ; store header to (obj+0)
0x00007f3e6bb617f0: movl   $0xf80001dd,0x8(%rax)  ; store klass to (obj+8)
0x00007f3e6bb617f7: mov    %r12d,0xc(%rax)        ; zero out the rest of the object
```

分配逻辑内联在生成代码中，所以并不需要调用 GC 来分配对象。如果申请分配的对象耗尽了 TLAB，或者对象比 TLAB 还大，那么我们采用一条“慢路径”，要么在此处直接分配，要么返回一个新的 TLAB。绝大多数*正常*的分配逻辑*仅仅是*将 TLAB 当前的指针增加对象的大小，然后继续执行。

这就是这种分配机制有时被称为*“指针碰撞分配（pointer bump allocation）”*的原因。指针碰撞分配需要一块连续的内存用于分配，但是这又引入了内存压缩的需要。留意一下 CMS 在老年代是如何从空闲列表分配内存的，受益于指针碰撞分配的方式，使得并发清除成为可能。新生代更少的存活对象将会支付空闲列表分配的成本。

在实验中我们可以通过 `-XX:-UseTLAB` 关闭 TLAB 机制。所有的对象分配将会执行下述本地代码：

```
-   17.12%     0.00%  org.openjdk.All  perf-31615.map
   - 0x7faaa3b2d125
      - 16.59% OptoRuntime::new_instance_C
         - 11.49% InstanceKlass::allocate_instance
              2.33% BlahBlahBlahCollectedHeap::mem_allocate  <---- entry point to GC
              0.35% AllocTracer::send_allocation_outside_tlab_event
```

...但是通常来说这不是一个好主意。

## 实验

像往常一样，让我们构建一个实验来观察 TLAB 分配。因为所有 GC 实现都有这个特性，所以出于最小化运行环境影响的目的，我们采用实验性的 [Epsilon GC](http://openjdk.java.net/jeps/8174901)。事实上它*只*实现了分配，所为为实验提供了很好的研究平台。

迅速设计一个工作负载：分配 50M 个对象（为什么不呢？），运行在 JMH 的 SingleShot 模式，这样就不会做性能统计和分析。你也可以单独执行这样的测试，只是使用 SingleShot 实在太方便了。

```
@Warmup(iterations = 3)
@Measurement(iterations = 3)
@Fork(3)
@BenchmarkMode(Mode.SingleShotTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class AllocArray {
    @Benchmark
    public Object test() {
        final int size = 50_000_000;
        Object[] objects = new Object[size];
        for (int c = 0; c < size; c++) {
            objects[c] = new Object();
        }
        return objects;
    }
}
```

这个测试用例在单线程中分配了 50M 个对象。基于经验，我们配置了 20GB 的堆内存，并且至少迭代执行6次。实验性的 `-XX:EpsilonTLABSize` 配置项用于精确控制 TLAB 的大小。其它 OpenJDK GC 采用[*自适应的* TLAB 大小调整](https://blogs.oracle.com/daviddetlefs/entry/tlab_sizing_an_annoying_little)策略，这种方式将会基于分配压力和其它因素调整 TLAB 的大小。

闲话少说，让我们看一下结果：

```
Benchmark                     Mode  Cnt     Score    Error   Units

# Times, lower is better                                            # TLAB size
AllocArray.test                 ss    9   548.462 ±  6.989   ms/op  #      1 KB
AllocArray.test                 ss    9   268.037 ± 10.966   ms/op  #      4 KB
AllocArray.test                 ss    9   230.726 ±  4.119   ms/op  #     16 KB
AllocArray.test                 ss    9   223.075 ±  2.267   ms/op  #    256 KB
AllocArray.test                 ss    9   225.404 ± 17.080   ms/op  #   1024 KB

# Allocation rates, higher is better
AllocArray.test:·gc.alloc.rate  ss    9  1816.094 ± 13.681  MB/sec  #      1 KB
AllocArray.test:·gc.alloc.rate  ss    9  2481.909 ± 35.566  MB/sec  #      4 KB
AllocArray.test:·gc.alloc.rate  ss    9  2608.336 ± 14.693  MB/sec  #     16 KB
AllocArray.test:·gc.alloc.rate  ss    9  2635.857 ±  8.229  MB/sec  #    256 KB
AllocArray.test:·gc.alloc.rate  ss    9  2627.845 ± 60.514  MB/sec  #   1024 KB
```

我们在单线程中实现 2.5 GB/sec 的分配速度。由于一个对象16字节，所以这意味着每秒 160 百万个对象。在多线程的工作负载下，分配速率可能达到每秒几十 GB。当然一旦 TLAB 变小，分配成本将会升高，分配速率将会下降。很不幸我们不能将 TLAB 设置为 1KB 以下，这是因为 Hotspot 的实现需要浪费一些空间，但是我们可以完全关闭 TLAB 机制，看一下对性能的影响：

```
Benchmark                      Mode  Cnt     Score   Error    Units

# -XX:-UseTLAB
AllocArray.test                  ss    9  2784.988 ± 18.925   ms/op
AllocArray.test:·gc.alloc.rate   ss    9   580.533 ±  3.342  MB/sec
```

我咧个去！分配速率下降了5倍，执行时间变大了10倍！这还是没有多线程并发申请内存的情况（可能会发生原子性竞争），或者是需要查找可用内存位置的场景（比如说尝试从空闲列表中分配）。对于 Epsilon 来说，分配的逻辑仅仅是一个 compare-and-set —— 因为它通过指针移动分配内存。如果你再增加一个线程 —— 总共两个执行线程 —— 而且关闭 TLAB，性能就更糟糕了：

```
Benchmark                            Mode  Cnt           Score       Error   Units

# TLAB = 4M (default for Epsilon)
AllocArray.test                        ss    9         407.729 ±     7.672   ms/op
AllocArray.test:·gc.alloc.rate         ss    9        4190.670 ±    45.909  MB/sec

# -XX:-UseTLAB
AllocArray.test                        ss    9        8490.585 ±   410.518   ms/op
AllocArray.test:·gc.alloc.rate         ss    9         422.960 ±    19.320  MB/sec
```

现在性能下降了20倍。随着线程数增加，性能将进一步下降！

## 观察

TLAB 是内存分配机制的骨干：它消除了分配器的并发瓶颈，提供了更快的分配方式，改善了整体的性能。有一种想法很有趣：TLAB将会造成更频繁的 GC，原因仅仅是分配的太快了！相反，如果内存管理器没有快速分配内存的方式，那么将会隐藏内存回收的性能问题。当我们对比内存管理器的时候，需要充分理解分配和回收两部分，以及两种之间的关系。
