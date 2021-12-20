原文地址：[JVM Anatomy Park #8: Local Variable Reachability](https://shipilev.net/jvm-anatomy-park/8-local-var-reachability/)

## 问题

存储在本地变量中的引用在离开作用域之后就会被回收。是这样么？

## 理论

这种想法植根于很多程序员的 C/C++ 经验，因为手册中就是这样写的：

> 1.  Local objects explicitly declared auto or register or not explicitly declared static or extern have automatic storage duration. The storage for these objects lasts until the block in which they are created exits.
>     
>     
> 2.  [Note: these objects are initialized and destroyed as described in 6.7. ]
>     
>     
> 3.  If a named automatic object has initialization or a destructor with side effects, it shall not be destroyed before the end of its block, nor shall it be eliminated as an optimization even if it appears to be unused, except that a class object or its copy may be eliminated as specified in 12.8.
>
>  — C++98 Standard 3.7.2 "Automatic storage duration"


这是一个很有用的语言特性，因为这将对象生命周期绑定到语法代码块。例如，可以这样：

```
void method() {
  ...something...

  {
     MutexLocker ml(mutex);
     ...something under the lock...
  } // ~MutexLocker unlocks

  ...something else...
}
```

从 C++ 转过来的程序员可能希望 Java 中也有这一特性。虽然 Java 中没有析构方法，但是 Java 也有相应的机制检测对象是否可达，并且根据软、弱、虚引用和终结器做出操作。*然而*，Java 中的语法代码块并不是像 C++ 那样。例如：

> “对程序转换做优化的目的是减少可达对象的数量，使其少于人们以为的可达对象的数量。例如，Java 编译器或代码生成器可以选择将不再使用的变量或参数设置为 null，从而导致这种对象的存储空间可能随后很快就会被回收。”
>
> — Java 语言规范 8 12.6.1 “实现终结”

这真的那么重要么？

## 实验

很容易展示 Java  与 C++ 的不同。以这个类为例：

```
public class LocalFinalize {
    ...
    private static volatile boolean flag;

    public static void pass() {
        MyHook h1 = new MyHook();
        MyHook h2 = new MyHook();

        while (flag) {
            // spin
        }

        h1.log();
    }

    public static class MyHook {
       public MyHook() {
           System.out.println("Created " + this);
       }

       public void log() {
           System.out.println("Alive " + this);
       }

       @Override
       protected void finalize() throws Throwable {
           System.out.println("Finalized " + this);
       }
    }
}
```

你可能天真地认为 `h2` 的生命周期直到 `pass` 方法结束。因为在 `flag` 为 `true` 的时候，一直在执行等待循环，所以对象将不会被终结。

现在的问题是，我们想要方法被编译，然后观察有趣的行为。为了实现这一点，我们可以执行两次：第一次进入方法，循环等待一会儿，然后退出。这将会编译这个方法，因为循环体会执行很多次，足以触发编译。然后第二次进入方法，但是不再退出循环。

可以这样来做：

```
public static void arm() {
    new Thread(() -> {
        try {
             Thread.sleep(5000);
             flag = false;
        } catch (Throwable t) {}
    }).start();
}

public static void main(String... args) throws InterruptedException {
    System.out.println("Pass 1");
    arm();
    flag = true;
    pass();

    System.out.println("Wait for pass 1 finalization");
    Thread.sleep(10000);

    System.out.println("Pass 2");
    flag = true;
    pass();
}
```

我们还需要启动一个后台线程反复触发 GC，这样就能触发终结。好了，设置完了（[完整的代码在这里](https://shipilev.net/jvm-anatomy-park/8-local-var-reachability/LocalFinalize.java)），让我们执行一下：

```
$ java -version
java version "1.8.0_101"
Java(TM) SE Runtime Environment (build 1.8.0_101-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.101-b13, mixed mode)

$ java LocalFinalize
Pass 1
Created LocalFinalize$MyHook@816f27d     # h1 created
Created LocalFinalize$MyHook@87aac27     # h2 created
Alive LocalFinalize$MyHook@816f27d       # h1.log called

Wait for pass 1 finalization
Finalized LocalFinalize$MyHook@87aac27   # h1 finalized
Finalized LocalFinalize$MyHook@816f27d   # h2 finalized

Pass 2
Created LocalFinalize$MyHook@3e3abc88    # h1 created
Created LocalFinalize$MyHook@6ce253f1    # h2 created
Finalized LocalFinalize$MyHook@6ce253f1  # h2 finalized (!)
```

哎哟。因为编译器知道 `h2` 的最后一次使用就是在分配之后，所以就直接终结了。因此当垃圾收集器检测变量存活时—— 在稍后的循环中，它再也不会认为 `h2` 还活着了。因此垃圾收集器认为  `MyHook` 实例已经死了，并且执行终结方法。因为 `h1` 在循环之后还会使用，所以它是可达的，不能执行终结方法。

实际上这是一个伟大的特性，因为它使得 GC 在方法退出之前就可以回收本地分配的大缓存，比如：

```
void processAndWait() {
  byte[] buf = new byte[1024 * 1024];
  writeToBuf(buf);
  processBuf(buf); // last use!
  waitForTheDeathOfUniverse(); // oops
}
```

## 更深入些

实际上，你可以从反汇编代码中观察技术细节。首先，字节码甚至没有提及本地变量，存储 `h2` 实例的槽位1一直保留到了方法结束。

```
$ javap -c -v -p LocalFinalize.class

  public static void pass();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=0
         0: new           #17                 // class LocalFinalize$MyHook
         3: dup
         4: invokespecial #18                 // Method LocalFinalize$MyHook."<init>":()V
         7: astore_0
         8: new           #17                 // class LocalFinalize$MyHook
        11: dup
        12: invokespecial #18                 // Method LocalFinalize$MyHook."<init>":()V
        15: astore_1
        16: getstatic     #10                 // Field flag:Z
        19: ifeq          25
        22: goto          16
        25: aload_0
        26: invokevirtual #19                 // Method LocalFinalize$MyHook.log:()V
        29: return
```

保留调试数据（`javac -g`）编译将会输出本地变量表（LVT），本地变量的生命周期“看起来”一直持续到方法结束：

```
  public static void pass();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=0
         0: new           #17                 // class LocalFinalize$MyHook
         3: dup
         4: invokespecial #18                 // Method LocalFinalize$MyHook."<init>":()V
         7: astore_0
         8: new           #17                 // class LocalFinalize$MyHook
        11: dup
        12: invokespecial #18                 // Method LocalFinalize$MyHook."<init>":()V
        15: astore_1
        16: getstatic     #10                 // Field flag:Z
        19: ifeq          25
        22: goto          16
        25: aload_0
        26: invokevirtual #19                 // Method LocalFinalize$MyHook.log:()V
        29: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            8      22     0    h1   LLocalFinalize$MyHook; //  8 + 22 = 30
           16      14     1    h2   LLocalFinalize$MyHook; // 16 + 14 = 30
```

这可能会让你认为可达性扩展到了方法结束，因为通常来说“作用域”是由 LVT 定义的。但是实际上不是这样的，因为优化器可以判断本地变量不会再被使用了，然后做相应的优化。在我们当前的测试用例中就是这样（在伪代码中）：

```
public static void pass() {
    MyHook h1 = new MyHook();
    MyHook h2 = new MyHook();

    while (flag) {
        // spin
        // <gc 安全点>
        // 这里，编译的代码知道机器寄存器和栈中的引用。
        // 到那时，”h2“ 不再被使用了。因此，GC 认为它已经死了。
    }

    h1.log();
}
```

从 `-XX:+PrintAssembly` 的输出中可以看到一些迹象：

```
data16 data16 xchg %ax,%ax   ; ImmutableOopMap{r10=Oop rbp=Oop}
                              ;*goto {reexecute=1 rethrow=0 return_oop=0}
                              ; - LocalFinalize::pass@22 (line 43)

LOOP:
test   %eax,0x15ae2bca(%rip)  # 0x00007f30868ff000
                              ; *goto {reexecute=0 rethrow=0 return_oop=0}
                              ; - LocalFinalize::pass@22 (line 43)
                              ;   {poll}

movzbl 0x70(%r10),%r8d        ;*getstatic flag {reexecute=0 rethrow=0 return_oop=0}
                              ; - LocalFinalize::pass@16 (line 43)

test   %r8d,%r8d
jne    LOOP                   ;*ifeq {reexecute=0 rethrow=0 return_oop=0}
                              ; - LocalFinalize::pass@19 (line 43)
```

`ImmutableOopMap{r10=Oop rbp=Oop}` 表示 `%r10` 和`%rbp` 持有”普通对象指针“。`%r10` 持有 `this` —— 看一下如何通过它读取 `flag`，`%rbp` 持有 `h1` 的引用。`h2` 的引用就找不到了。

## 替代方案

扩展本地变量的可达性到特定程序点，这个需求可以通过再次使用变量达到目的。然而，如果没有可观测的副作用，这是很难实现的。举例来说，”仅仅“调用方法，或者传递本地变量是不够的，因为方法可能会被内联，相同的优化又开始了。从 Java 9 开始，[`java.lang.ref.Reference::reachabilityFence`](http://download.java.net/java/jdk9/docs/api/java/lang/ref/Reference.html#reachabilityFence-java.lang.Object-) 提供了需要的语义。

如果你“仅仅”想要拥有 C++ 那样的“代码块退出时释放” —— 离开代码块时做一些操作 —— 那么使用 `try-finally` 更合适。

## 观察

Java 本地变量的可达性不是由语法块定义的，可达性*至少*到最后一次使用，也可能*精确*到最后一次使用。使用对象不可达时的通知机制（终结方法，弱、软、虚引用）可能会成为“早期”检查的受害者，因为这次可能还没执行到上次到达的方法或代码块结束位置，对象就不可达了。
