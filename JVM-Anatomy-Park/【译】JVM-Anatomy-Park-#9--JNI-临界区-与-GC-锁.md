原文地址：[JVM Anatomy Park #9: JNI Critical and GC Locker](https://shipilev.net/jvm-anatomy-park/9-jni-critical-gclocker/)

## 问题

JNI `Get*Critical` 如何与 GC 协同？GC Locker 是什么？

## 理论

如果你熟悉 JNI，那么你会知道有两组方法可以获取数组内容。一组是 `Get<PrimitiveType>Array*` 方法，另一组是[这些](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetPrimitiveArrayCritical_ReleasePrimitiveArrayCritical)：

> ```
> void * GetPrimitiveArrayCritical(JNIEnv *env, jarray array, jboolean *isCopy);
> void ReleasePrimitiveArrayCritical(JNIEnv *env, jarray array, void *carray, jint mode);
> ```
> 
> 这两个方法的语义与 Get/Release*ArrayElements 方法类似。如果可能，VM 将返回原始数组的指针；否则返回一个副本。然而，在使用上存在很多限制。
>
> —— 《JNI 指南》 第四章：JNI 方法

这样做的好处很明显：与其给你一个 Java 数组的副本，VM 可以直接给你返回一个指针，这样就能提高性能。当然这样做也有很多坑，下面将一一罗列：

> 调用 GetPrimitiveArrayCritical 方法之后，本地代码在调用 ReleasePrimitiveArrayCritical 之前不能执行太长时间。我们需要将这两次调用之间的代码当做“临界区”看待。在临界区中，本地代码不能调用其它的 JNI 方法，也不能执行引起当前线程阻塞等待其它线程的系统调用。（例如，当前线程不能读取其它线程写的流）
> 
> 这些限制使得本地代码更可能获取非拷贝的数组，即使 VM 不支持钉住。例如，当本地代码持有 GetPrimitiveArrayCritical 返回的指针时，VM 可能会暂时关闭垃圾收集。
>
> —— 《JNI 指南》 第四章：JNI 方法

这一段读起来的意思好像是，在临界区执行的时候 VM 将会停止 GC。

实际上对于 VM 来说唯一的强不变式是维护在“临界区”持有的对象不被移动。有很多不同的实现策略可以尝试:

1. 当持有临界区对象的时候**完全关闭 GC**。这就是最简单的复制策略，因为这将不会影响接下来的 GC。缺点是你不得不无限期的阻塞 GC（只能寄希望于用户足够快的“释放”），这将会造成很多问题。
2. **钉住对象**，在收集的时候忽略它。如果收集器期望分配连续的空间，或者期望处理整个堆子空间，那么就比较难实现了。例如，如果你将对象分配在新生代，那么就不能简单的“忽略”收集了。你也不能移动对象，因为这就打破了不变式。
3. **钉住包含对象的子空间**。如果 GC 的粒度是整代，那么也很难实现。但是如果你的堆是分块的，那么你可以钉住单个块，让 GC 忽略这个块，这样就能实现不变式了。

我们曾经看到有些人依赖 JNI 临界区暂时关闭 GC，但是这仅仅对第一种策略有效，实际上并不是每个收集器都采用这种最简单的策略。

我们可以通过代码验证么？

## 实验

像往常一样，我们可以这样构建测试用例，在 JNI 临界区获取 `int[]` 数组，然后**故意忽略**释放数组的建议。相反，我们在获取和释放之间分配并持有大量对象：

```
public class CriticalGC {

  static final int ITERS = Integer.getInteger("iters", 100);
  static final int ARR_SIZE = Integer.getInteger("arrSize", 10_000);
  static final int WINDOW = Integer.getInteger("window", 10_000_000);

  static native void acquire(int[] arr);
  static native void release(int[] arr);

  static final Object[] window = new Object[WINDOW];

  public static void main(String... args) throws Throwable {
    System.loadLibrary("CriticalGC");

    int[] arr = new int[ARR_SIZE];

    for (int i = 0; i < ITERS; i++) {
      acquire(arr);
      System.out.println("Acquired");
      try {
        for (int c = 0; c < WINDOW; c++) {
          window[c] = new Object();
        }
      } catch (Throwable t) {
        // omit
      } finally {
        System.out.println("Releasing");
        release(arr);
      }
    }
  }
}
```

本地代码部分：

```
#include <jni.h>
#include <CriticalGC.h>

static jbyte* sink;

JNIEXPORT void JNICALL Java_CriticalGC_acquire
(JNIEnv* env, jclass klass, jintArray arr) {
   sink = (*env)->GetPrimitiveArrayCritical(env, arr, 0);
}

JNIEXPORT void JNICALL Java_CriticalGC_release
(JNIEnv* env, jclass klass, jintArray arr) {
   (*env)->ReleasePrimitiveArrayCritical(env, arr, sink, 0);
}
```

我们需要生成合适的头文件，将本地代码编译链接成库文件，然后确保 JVM 可以加载库文件。所有的文件都打包在[这里](https://shipilev.net/jvm-anatomy-park/9-jni-critical-gclocker/critical.zip)。

### Parallel/CMS

首先观察 Parallel 收集器的行为：

```
$ make run-parallel
java -Djava.library.path=. -Xms4g -Xmx4g -verbose:gc -XX:+UseParallelGC CriticalGC
[0.745s][info][gc] Using Parallel
...
[29.098s][info][gc] GC(13) Pause Young (GCLocker Initiated GC) 1860M->1405M(3381M) 1651.290ms
Acquired
Releasing
[30.771s][info][gc] GC(14) Pause Young (GCLocker Initiated GC) 1863M->1408M(3381M) 1589.162ms
Acquired
Releasing
[32.567s][info][gc] GC(15) Pause Young (GCLocker Initiated GC) 1866M->1411M(3381M) 1710.092ms
Acquired
Releasing
...
1119.29user 3.71system 2:45.07elapsed 680%CPU (0avgtext+0avgdata 4782396maxresident)k
0inputs+224outputs (0major+1481912minor)pagefaults 0swaps
```

注意在“Acquired”与“Released”之间没有发生 GC，所以实现的细节就很容易猜到了。确凿的证据是“GCLocker Initiated GC”。[GCLocker](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/gc/shared/gcLocker.hpp) 是阻止 JNI 临界区发生 GC 的**锁**。看一下 OpenJDK 代码中的[相关片段](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/prims/jni.cpp#l3173)：

```
JNI_ENTRY(void*, jni_GetPrimitiveArrayCritical(JNIEnv *env, jarray array, jboolean *isCopy))
  JNIWrapper("GetPrimitiveArrayCritical");
  GCLocker::lock_critical(thread);   // <--- acquire GCLocker!
  if (isCopy != NULL) {
    *isCopy = JNI_FALSE;
  }
  oop a = JNIHandles::resolve_non_null(array);
  ...
  void* ret = arrayOop(a)->base(type);
  return ret;
JNI_END

JNI_ENTRY(void, jni_ReleasePrimitiveArrayCritical(JNIEnv *env, jarray array, void *carray, jint mode))
  JNIWrapper("ReleasePrimitiveArrayCritical");
  ...
  // The array, carray and mode arguments are ignored
  GCLocker::unlock_critical(thread); // <--- release GCLocker!
  ...
JNI_END
```

当尝试执行 GC 的时候，JVM 将会检查这个锁是否被持有。如果某个线程持有锁，那么就不能继续执行 GC，至少在 Parallel、CMS 和 G1 中是这样。当下一个临界区 JNI 操作结束时“释放”了锁，VM 将会检查是否有 GCLocker 阻塞的 GC，如果有那么就[触发 GC](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/gc/shared/gcLocker.cpp#l138)。这就产生了“GCLocker Initiated GC”。

### G1

当然，因为我们正在玩火 —— 在 JNI 临界区做奇怪的事情 —— 所以随时可能爆炸。再观察一下 G1 收集器的行为：

```
$ make run-g1
java -Djava.library.path=. -Xms4g -Xmx4g -verbose:gc -XX:+UseG1GC CriticalGC
[0.012s][info][gc] Using G1
<HANGS>
```

哎哟！程序中止了。`jstack` 显示处于 `RUNNABLE` 状态，但是正在等待某个奇怪的条件：

```
"main" #1 prio=5 os_prio=0 tid=0x00007fdeb4013800 nid=0x4fd9 waiting on condition [0x00007fdebd5e0000]
   java.lang.Thread.State: RUNNABLE
  at CriticalGC.main(CriticalGC.java:22)
```

最简单的查找线索的方法是执行 “fastdebug” 构建，那将会停止在这个有趣的断言上：

```
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  Internal Error (/home/shade/trunks/jdk9-dev/hotspot/src/share/vm/gc/shared/gcLocker.cpp:96), pid=17842, tid=17843
#  assert(!JavaThread::current()->in_critical()) failed: Would deadlock
#
Native frames: (J=compiled Java code, A=aot compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [libjvm.so+0x15b5934]  VMError::report_and_die(...)+0x4c4
V  [libjvm.so+0x15b644f]  VMError::report_and_die(...)+0x2f
V  [libjvm.so+0xa2d262]  report_vm_error(...)+0x112
V  [libjvm.so+0xc51ac5]  GCLocker::stall_until_clear()+0xa5
V  [libjvm.so+0xb8b6ee]  G1CollectedHeap::attempt_allocation_slow(...)+0x92e
V  [libjvm.so+0xba423d]  G1CollectedHeap::attempt_allocation(...)+0x27d
V  [libjvm.so+0xb93cef]  G1CollectedHeap::allocate_new_tlab(...)+0x6f
V  [libjvm.so+0x94bdba]  CollectedHeap::allocate_from_tlab_slow(...)+0x1fa
V  [libjvm.so+0xd47cd7]  InstanceKlass::allocate_instance(Thread*)+0xc77
V  [libjvm.so+0x13cfef0]  OptoRuntime::new_instance_C(Klass*, JavaThread*)+0x830
v  ~RuntimeStub::_new_instance_Java
J 87% c2 CriticalGC.main([Ljava/lang/String;)V (82 bytes) ...
v  ~StubRoutines::call_stub
V  [libjvm.so+0xd99938]  JavaCalls::call_helper(...)+0x858
V  [libjvm.so+0xdbe7ab]  jni_invoke_static(...) ...
V  [libjvm.so+0xdde621]  jni_CallStaticVoidMethod+0x241
C  [libjli.so+0x463c]  JavaMain+0xa8c
C  [libpthread.so.0+0x76ba]  start_thread+0xca
```

仔细观察调用链，我们可以重建发生的事情：尝试分配新的对象，因为没有 [TLABs](https://shipilev.net/jvm-anatomy-park/4-tlab-allocation/) 满足分配，所以尝试获取新的 TLAB。然后发现没有可用的 TLABs，尝试分配，失败，发现需要等待 GCLocker 才能开始 GC。进入 `stall_until_clear` 方法等待锁。。。但是因为线程一直持有 GCLocker，这里的等待将会导致死锁。[爆炸](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/gc/shared/gcLocker.cpp#l95)。

这是符合规范的，因为这个测试用例尝试在获取释放代码块中间分配对象。离开 JNI 方法而不调用 `release` 是错误的。在没有离开 JNI 方法之前，不调用 JNI 是不能进行分配的，而这违反了“不可调用 JNI 方法”的准则。


你可以调整测试用例以避免这样的问题，但是你会发现 GCLocker 将会延迟收集，这意味着仅剩很少空间的时候才会开始 GC，而这将会导致 Full GC。哎哟。

### Shenandoah

就像理论描述的那样，分块的收集器可以钉住持有对象的特定内存块，让特定的内存块在 JNI 临界区释放前避免收集。[Shenandoah](https://wiki.openjdk.java.net/display/shenandoah/Main) 当前就是这样实现的。

```
$ make run-shenandoah
java -Djava.library.path=. -Xms4g -Xmx4g -verbose:gc -XX:+UseShenandoahGC CriticalGC
...
Releasing
Acquired
[3.325s][info][gc] GC(6) Pause Init Mark 0.287ms
[3.502s][info][gc] GC(6) Concurrent marking 3607M->3879M(4096M) 176.534ms
[3.503s][info][gc] GC(6) Pause Final Mark 3879M->1089M(4096M) 0.546ms
[3.503s][info][gc] GC(6) Concurrent evacuation  1089M->1095M(4096M) 0.390ms
[3.504s][info][gc] GC(6) Concurrent reset bitmaps 0.715ms
Releasing
Acquired
....
41.79user 0.86system 0:12.37elapsed 344%CPU (0avgtext+0avgdata 4314256maxresident)k
0inputs+1024outputs (0major+1085785minor)pagefaults 0swaps
```

注意，在 JNI 临界区持有期间，CC 周期从开始到结束。Shenandoah 仅仅钉住持有数组的内存块，而其他的内存块正常进行收集。当 JNI 临界区持有的对象在被收集的内存块中时也可以执行 GC，首先排除对应的内存块，然后钉住这个块（也就是将它排除出收集集合）。这就能实现*不用 GCLocker* 的 JNI 临界区，因此也没有 GC 延迟。

## 观察

处理 JNI 临界区需要 VM 的帮助，或者关闭 GC，或者采用 GCLocker 类似的机制，或者钉住包含对象的子空间，或者仅仅钉住对象。不同的 GCs 采用不同的策略处理 JNI 临界区，某个收集器的副作用 —— 比如延迟 GC 周期 —— 可能并不会出现在另一个收集器中。

请注意规范中：*在临界区中，本地代码不能调用其它 JNI 方法*，这仅仅是最低的要求。上述测试表明，在规范允许的范围内，实现的质量决定了打破规范时的严重程度。某些 GC 更宽松，而某些会更严格。如果你想要保证可移植性，那么请遵循规范，而不是实现细节。

如果你依赖实现细节（这**是**一个坏主意），使用 JNI 遇到了这些问题，那么需要理解收集器的处理策略，并且选择合适的 GC。
