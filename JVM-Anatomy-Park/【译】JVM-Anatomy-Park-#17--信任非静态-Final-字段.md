原文地址：[JVM Anatomy Park #17: Trust Nonstatic Final Fields](https://shipilev.net/jvm-anatomy-park/17-trust-nonstatic-final-fields/)

## 问题

JVM 是否可以信任非静态 `final` 字段？

## 理论

正如我们在[《#15: 即时常量》](https://shipilev.net/jvm-anatomy-park/15-just-in-time-constants/)中提到的，编译器信任 `static final` 字段，因为这个值不依赖特定对象，而且是不能改变的。但是如果我们准确的知道对象身份，例如*引用自身*是 `static final`，那么我们可以信任它的 `final` 实例字段么？例如：

```
class M {
  final int x;
  M(int x) { this.x = x; }
}

static final M KNOWN_M = new M(1337);

void work() {
  // We know exactly the slot that holds the variable, can we just
  // inline the value 1337 here?
  return KNOWN_M.x;
}
```

棘手的问题是，如果字段被**修改**，会发生什么？《Java 语言规范》*允许*看不到这样的更新，因为字段是 `final` 的。很不幸，真正的框架设法依靠更强的行为：字段更新将会被看到。下面将要进行的实验激进地优化了这些场景，当写操作发生的时候就[尝试](https://bugs.openjdk.java.net/browse/JDK-8058164)逆优化。当前的状态是只有某些内部类是[隐式信任](http://hg.openjdk.java.net/jdk/jdk/file/2c1af559e922/src/hotspot/share/ci/ciField.cpp#l203)的：

```
static bool trust_final_non_static_fields(ciInstanceKlass* holder) {
  if (holder == NULL)
    return false;
  if (holder->name() == ciSymbol::java_lang_System())
    // Never trust strangely unstable finals:  System.out, etc.
    return false;
  // Even if general trusting is disabled, trust system-built closures in these packages.
  if (holder->is_in_package("java/lang/invoke") || holder->is_in_package("sun/invoke"))
    return true;
  // Trust VM anonymous classes. They are private API (sun.misc.Unsafe) and can't be serialized,
  // so there is no hacking of finals going on with them.
  if (holder->is_anonymous())
    return true;
  // Trust final fields in all boxed classes
  if (holder->is_box_klass())
    return true;
  // Trust final fields in String
  if (holder->name() == ciSymbol::java_lang_String())
    return true;
  // Trust Atomic*FieldUpdaters: they are very important for performance, and make up one
  // more reason not to use Unsafe, if their final fields are trusted. See more in JDK-8140483.
  if (holder->name() == ciSymbol::java_util_concurrent_atomic_AtomicIntegerFieldUpdater_Impl() ||
      holder->name() == ciSymbol::java_util_concurrent_atomic_AtomicLongFieldUpdater_CASUpdater() ||
      holder->name() == ciSymbol::java_util_concurrent_atomic_AtomicLongFieldUpdater_LockedUpdater() ||
      holder->name() == ciSymbol::java_util_concurrent_atomic_AtomicReferenceFieldUpdater_Impl()) {
    return true;
  }
  return TrustFinalNonStaticFields;
}
```

当提供实验性的 `-XX:+TrustFinalNonStaticFields` 参数的时候，常规的 `final` 字段才被信任。

## 实践

我们可以在实践中观察么？修改 [《#15: 即时常量》](https://shipilev.net/jvm-anatomy-park/15-just-in-time-constants/)中的 JMH 测试用例，使用对象中的 `final` 字段，而不是对象本身：

```
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class TrustFinalFields {

    static final T t_static_final;
    static       T t_static;
           final T t_inst_final;
                 T t_inst;

    static {
        t_static_final = new T(1000);
        t_static = new T(1000);
    }

    {
        t_inst_final = new T(1000);
        t_inst = new T(1000);
    }

    static class T {
        final int x;

        public T(int x) {
            this.x = x;
        }
    }

    @Benchmark public int _static_final() { return 1000 / t_static_final.x; }
    @Benchmark public int _static()       { return 1000 / t_static.x;       }
    @Benchmark public int _inst_final()   { return 1000 / t_inst_final.x;   }
    @Benchmark public int _inst()         { return 1000 / t_inst.x;         }

}
```

在我的机器上，这个测试用例输出：

```
Benchmark                       Mode  Cnt  Score   Error  Units
TrustFinalFields._inst          avgt   15  4.316 ± 0.003  ns/op
TrustFinalFields._inst_final    avgt   15  4.317 ± 0.002  ns/op
TrustFinalFields._static        avgt   15  4.282 ± 0.011  ns/op
TrustFinalFields._static_final  avgt   15  4.202 ± 0.002  ns/op
```

看起来 `static final` 并没有太大帮助。确实如此，你可以看一下生成的指令：

```
0.02%   ↗  movabs $0x782b67520,%r10   ; {oop(a 'org/openjdk/TrustFinalFields$T';)}
        │  mov    0x10(%r10),%r10d    ; get field $x
        │  ...
0.19%   │  cltd
0.02%   │  idiv   %r10d               ; idiv
        │  ...
0.16%   │  test   %r11d,%r11d         ; check and run @Benchmark again
        ╰  je     BACK
```

对象被信任位于堆中的给定位置（`$0x782b67520`），但是我们不信任这个字段！配置参数 `-XX:+TrustFinalNonStaticFields`，再运行：

```
Benchmark                       Mode  Cnt  Score    Error  Units
TrustFinalFields._inst          avgt   15  4.318 ±  0.001  ns/op
TrustFinalFields._inst_final    avgt   15  4.317 ±  0.003  ns/op
TrustFinalFields._static        avgt   15  4.290 ±  0.002  ns/op
TrustFinalFields._static_final  avgt   15  1.901 ±  0.001  ns/op  # <--- !!!
```

这次 `final` 字段被折叠了，就像我们在 perfasm 输出中看到的：

```
3.04%   ↗  mov    %r10,(%rsp)
        │  mov    0x38(%rsp),%rsi
8.26%   │  mov    $0x1,%edx           ; <--- constant folded to 1
        │  ...
0.04%   │  test   %r11d,%r11d         ; check and run @Benchmark again
        ╰  je     BACK
```

## 观察

如果要信任实例 final 字段，那么必须知道当前操作的对象。但即使是这样，当我们确定它不会破坏应用程序时，我们可以务实地这样做 —— 至少对已知的系统类可以。对于来自核心库的 `MethodHandle`、`VarHandle`、`Atomic*FieldUpdaters` 等高性能实现，折叠 final 字段为常量是实现高性能的基石。应用程序可以尝试使用实验性的 VM 参数，但是来自不正常应用程序的潜在危害可能会严重抑制性能收益。
