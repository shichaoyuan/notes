原文链接：[Fearless Concurrency with Rust](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)

Apr. 10, 2015 · Aaron Turon

Rust 项目旨在解决这两个棘手的问题：
*   如何进行安全的系统编程？
*   如何使得并发更容易？

起初，这两个问题看起来是互不相关的，但是使我们惊讶的是，最终的解决方案是一致的：**使得 Rust 安全的方法同样可以帮助正面处理并发**。

内存安全漏洞和并发漏洞通常都可以归结于访问了不应该访问的数据。Rust 的秘密武器是*所有权*，这是系统程序员试图遵循的一种访问控制原则，而 Rust 编译器将为你静态检查。

对于内存安全性，这意味着你可以在没有垃圾收集器的情况下编程，*而且*不需要害怕段错误，因为 Rust 将会捕捉你的错误。

对于并发，这意味着你可以选择各种各样的范式（消息传递，共享状态，无锁，纯函数式），而 Rust 将会帮助你避免常见的错误。

这是 Rust 中的并发：

*   [管道](http://static.rust-lang.org/doc/master/std/sync/mpsc/index.html) 传递发送消息的所有权，所以你可以将指针从一个线程发送到另一个线程，而不用担心两个线程争用指针指向的数据。**Rust 的管道强制执行线程隔离。**

*   [锁](http://static.rust-lang.org/doc/master/std/sync/struct.Mutex.html) 知道它保护的是什么数据，Rust 保证只有在持有锁的时候才能访问该数据。状态永远不会意外共享。**“锁数据，而不是代码”在 Rust 中被强制执行。**

*   每种数据类型知道它是否可以在线程之间[传递](http://static.rust-lang.org/doc/master/std/marker/trait.Send.html)或者被多个线程[访问](http://static.rust-lang.org/doc/master/std/marker/trait.Sync.html)，Rust 保证这种安全用法；没有数据争用，即使对无锁数据结构来说。**线程安全不仅仅是文档；这是法律。**

*   你甚至可以在线程之间[共享栈帧](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html)，Rust 将会静态地保证帧在其它线程使用时保持活跃状态。**即使是最大胆的共享模式，Rust 也能保证安全**。

所有这些好处都来自 Rust 的所有权模型，实际上锁、管道、无锁数据结构等等都定义在库中，而不是核心语言中。这意味着 Rust 实现并发的方法是*开放的*：新的库可以采用新的范式，处理新的漏洞，只需使用 Rust 的所有权特性添加 APIs 即可。

本文的目的就是让你初步了解这是如何实现的。

### []()背景：所有权

> 我们将会从 Rust 所有权和借用系统的概述开始。如果你对这些已经很熟悉了，那么你可以略过这两个“背景”部分，直接跳到并发部分。如果你想要更深入的介绍，那么我强烈推荐 [Yehuda Katz 的文章](http://blog.skylight.io/rust-means-never-having-to-close-a-socket/)。另外 [Rust book](http://doc.rust-lang.org/book/ownership.html) 介绍了所有的细节。

在 Rust 中，所有的值都有“所有作用域”，传递或者返回一个值意味着传递所有权（“移动它”）到一个新的作用域。当作用域结束时仍然持有的值将会自动销毁。

让我们看几个简单的例子。假设创建一个向量，添加一些元素：

```
fn make_vec() {
    let mut vec = Vec::new(); // owned by make_vec's scope
    vec.push(0);
    vec.push(1);
    // scope ends, `vec` is destroyed
}

```

创建值的作用域自然也会持有该值。在这个例子中，`make_vec` 的函数体是 `vec` 的所有作用域。所有者可以对 `vec` 做任意操作，包括通过 push 修改它。在作用域的结尾，`vec` 仍然被持有，所以它会自动释放。

如果向量被返回或传递，事情会变得更有趣：

```
fn make_vec() -> Vec<i32> {
    let mut vec = Vec::new();
    vec.push(0);
    vec.push(1);
    vec // transfer ownership to the caller
}

fn print_vec(vec: Vec<i32>) {
    // the `vec` parameter is part of this scope, so it's owned by `print_vec`

    for i in vec.iter() {
        println!("{}", i)
    }

    // now, `vec` is deallocated
}

fn use_vec() {
    let vec = make_vec(); // take ownership of the vector
    print_vec(vec);       // pass ownership to `print_vec`
}

```

现在，就在 `make_vec` 的作用域结束之前，`vec` 通过返回被移出；它没有被销毁。像 `use_vec` 这样的调用函数然后就会接收向量的所有权。

另一方面，`print_vec` 函数接受一个 `vec` 参数，向量的所有权就由调用者传递*给*了它。由于 `print_vec` 函数没有再传递所有权，所以在该作用域结束的位置，向量就被销毁了。

一旦所有权交出了，值就不能再使用了。例如，考虑这个 `use_vec` 的变体：

```
fn use_vec() {
    let vec = make_vec();  // take ownership of the vector
    print_vec(vec);        // pass ownership to `print_vec`

    for i in vec.iter() {  // continue using `vec`
        println!("{}", i * 2)
    }
}

```

如果你把这个版本提供给编译器，那么你将会收到一个错误：

```
error: use of moved value: `vec`

for i in vec.iter() {
         ^~~

```

编译器说 `vec` 不再可用了；所有权已经被转移到其它地方。这很好，因为此时向量已经被释放了！

灾难也就可以避免了。

### []()背景：借用

到目前为止这并不完全令人满意，因为 `print_vec` 函数的目的并不是销毁给定的向量。我们真正想要的是赋予 `print_vec` 函数对向量*临时*访问的权力，然后可以继续使用向量。

这时就需要*借用*了。如果你在 Rust 中访问了一个值，那么你可以将访问权借给调用的函数。**Rust 将会检查租约不会比被借的对象活的更久**。

想要借用一个值，你得生成一个对它的引用（一种指针），使用 `&` 操作符：

```
fn print_vec(vec: &Vec<i32>) {
    // the `vec` parameter is borrowed for this scope

    for i in vec.iter() {
        println!("{}", i)
    }

    // now, the borrow ends
}

fn use_vec() {
    let vec = make_vec();  // take ownership of the vector
    print_vec(&vec);       // lend access to `print_vec`
    for i in vec.iter() {  // continue using `vec`
        println!("{}", i * 2)
    }
    // vec is destroyed here
}

```

现在 `print_vec` 接受一个向量的引用，`use_vec` 函数通过 `&vec` 借出向量。因为借用是临时的，所以 `use_vec` 函数仍然持有向量的所有权；在 `print_vec` 调用返回后（对 `vec` 的租约过期了），`use_vec` 函数仍然可以继续使用该向量。

每个引用在有限的作用域内是有效的，编译器将自动确定这一点。引用有两种形式：

*   不可变引用 `&T`，这允许共享，但是不能修改。对于同一个值，可以同时存在多个 `&T` 引用，但是当这些引用有效时，值不能被修改。

*   可变引用 `&mut T`，这允许修改，但是不能共享。如果存在一个对值的 `&mut T` 引用，那么此时就不能存在其它有效的引用，但是该值是可以修改的。

Rust 在编译期检查这些规则；借用没有运行时开销。

为什么有两种引用？考虑这样一个函数：

```
fn push_all(from: &Vec<i32>, to: &mut Vec<i32>) {
    for i in from.iter() {
        to.push(*i);
    }
}

```

该函数迭代一个向量中的元素，添加到另一个向量中。迭代器在向量的当前位置和最终位置维护一个指针，一步一步向另一个方向移动。

如果函数的两个参数传入同一个向量会怎样呢？

```
push_all(&vec, &mut vec)

```

这将会带来灾难！由于我们添加元素到向量中，所以向量偶尔需要调整大小，分配一块新的内存，将元素复制到这里。迭代器将留下一个指向旧内存的悬空指针，这将导致内存不安全（伴随着段错误或者更糟的情况）。

幸运的是，Rust 保证**只要一个可变借用是有效的，那么就不能借用了**，输出的错误信息：

```
error: cannot borrow `vec` as mutable because it is also borrowed as immutable
push_all(&vec, &mut vec);
                    ^~~

```

灾难也就可以避免了。

### []()消息传递

既然我们已经简单介绍了 Rust 的所有权，那么让我们看看这对并发意味着什么。

并发编程有许多种风格，其中特别简单的一种是消息传递，其中线程或 actors 通过互相发送消息进行通信。这种风格的支持者强调了它将共享与通信联系在一起的方式：

> 不要通过共享内存通信；相反，要通过通信共享内存。
>
> --[Effective Go](http://golang.org/doc/effective_go.html)

**Rust 的所有权使得可以很容易将这个建议转换为编译器检查规则**。考虑下述管道 API（[Rust 标准库中的管道](http://static.rust-lang.org/doc/master/std/sync/mpsc/index.html)有些许不同）：

```
fn send<T: Send>(chan: &Channel<T>, t: T);
fn recv<T: Send>(chan: &Channel<T>) -> T;

```

管道泛化了传输的数据类型（API 中的 `<T: Send>` 部分）。`Send` 意味着 `T` 必须能在线程之间安全地发送；我将在文章稍后部分继续讨论这一点，但现在只需要知道 `Vec<i32>` 是 `Send` 即可。

和通常一样，在 Rust 中将一个 `T` 值传给 `send` 函数意味着将所有权转移给它。这一事实具有深远的影响：这意味着下述代码将生成编译器错误。

```
// Suppose chan: Channel<Vec<i32>>

let mut vec = Vec::new();
// do some computation
send(&chan, vec);
print_vec(&vec);

```

在这里，线程创建了一个向量，发送给了另一个线程，然后继续使用该向量。接收到向量的线程可能会在继续运行时对其进行修改，所以在这里调用 `print_vec` 函数会导致数据争用，或者就此而言，导致释放后使用漏洞。

相反，Rust 编译器会在调用 `print_vec` 处产生一个错误信息：

```
Error: use of moved value `vec`

```

灾难也就可以避免了。

### []()锁

处理并发问题的另一种方式是让线程通过被动的共享状态通信。

共享状态并发名声不好。很容易忘记获取锁，或者在错误的时间修改了错误的数据，从而导致灾难性的后果——以至于很多人完全避免使用这种方式。

Rust 的观点是这样的：

1.  然而，共享状态并发是一种基础的编程风格，对于系统代码、最大化性能和实现其它风格的并发都是必须的。

2.  这个问题实际上与*意外*共享状态有关。

Rust 旨在为你提供直接克服共享状态并发的工具，无论你使用有锁还是无锁技术。

在 Rust 中，由于所有权，线程自动互相“隔离”。只有在线程具有可变访问权时，写操作才可能发生，要么持有数据，要么持有数据的可变借用。无论在哪种情况下，**保证在那时线程唯一的访问数据**。为了了解这是如何实现的，让我们看一下锁。

记住，可变借用不能与其它借用同时发生。锁通过运行时同步提供相同的保证（“互斥”）。这就生成了一个与 Rust 所有权系统直接挂钩的锁 API。

这是一个简化的版本（[标准库中版本](http://static.rust-lang.org/doc/master/std/sync/struct.Mutex.html)具有更好的人机工学）：

```
// create a new mutex
fn mutex<T: Send>(t: T) -> Mutex<T>;

// acquire the lock
fn lock<T: Send>(mutex: &Mutex<T>) -> MutexGuard<T>;

// access the data protected by the lock
fn access<T: Send>(guard: &mut MutexGuard<T>) -> &mut T;

```

这个锁 API 在许多方面是与众不同的。

首先，`Mutex` 类型通过 `T` 泛化**锁保护的数据**类型。当你创建一个 `Mutex` 时，数据的所有权就被传递*到*了 mutex 中，立即放弃了对它的访问权。（锁在第一次创建时就解锁了）

接下来，你可以使用 `lock` 函数阻塞线程，直到获取锁为止。该函数提供了一个返回值 `MutexGuard<T>`，这也不太寻常。`MutexGuard` 在销毁的时候自动释放锁；没有单独的 `unlock` 函数。

访问锁的唯一方式是通过 `access` 函数，将 guard 的可变借用转换为数据的可变借用（带有一个更短的租约）：

```
fn use_lock(mutex: &Mutex<Vec<i32>>) {
    // acquire the lock, taking ownership of a guard;
    // the lock is held for the rest of the scope
    let mut guard = lock(mutex);

    // access the data by mutably borrowing the guard
    let vec = access(&mut guard);

    // vec has type `&mut Vec<i32>`
    vec.push(3);

    // lock automatically released here, when `guard` is destroyed
}

```

这里有两个关键因素：

*   `access` 函数返回的可变引用不能比 `MutexGuard` 活得更久。

*   只有在 `MutexGuard` 销毁的时候，锁才会被释放。

其结果是**Rust 强制执行锁准则：除非你获取了锁，否则不能访问锁保护的数据**。任何违背该准则的尝试将会生成编译器错误。例如，考虑下面这个有漏洞的“重构”：

```
fn use_lock(mutex: &Mutex<Vec<i32>>) {
    let vec = {
        // acquire the lock
        let mut guard = lock(mutex);

        // attempt to return a borrow of the data
        access(&mut guard)

        // guard is destroyed here, releasing the lock
    };

    // attempt to access the data outside of the lock.
    vec.push(3);
}

```

Rust 将会生成一个错误，指出问题：

```
error: `guard` does not live long enough
access(&mut guard)
            ^~~~~

```

灾难也就可以避免了。

### []()线程安全与“Send”

通常会将一些数据类型区分为“线程安全”的，而另一些不是。线程安全数据结构使用了足够多的内部同步机制，以确保多个线程可以安全地并发使用。

例如，Rust 为引用计数提供了两种“智能指针”：

*   `Rc<T>` 提供了普通读写的引用计数。这不是线程安全的。

*   `Arc<T>` 提供了*原子*操作的引用计数。这是线程安全的。

`Arc` 使用的硬件原子操作比 `Rc` 使用的普通操作更耗费资源，所以使用 `Rc` 比使用 `Arc` 更有优势。另外，`Rc<T>` 永远不要从一个线程迁移到另一个线程，因为这将导致竞态条件破坏计数。

通常来说，唯一的资源是仔细的文档；大部分语言在线程安全和线程不安全类型之间没有*语义*上的区别。

在 Rust 中，数据类型分为两类：[`Send`](http://static.rust-lang.org/doc/master/std/marker/trait.Send.html) 类，意味着可以安全地从一个线程移动到另一个线程，`!Send` 类，意味着不能安全地移动。如果一个类型的所有组件都是 `Send`，那么该类型也是——这覆盖了大部分类型。虽然某些基本类型本身不是线程安全的，也可以像 `Arc` 那样显式将一个类型标记为 `Send`，对编译器说：“相信我；我已经验证了必要的同步。”

自然而然的，`Arc` 是 `Send`，而 `Rc` 不是。

我们已经看到 `Channel` 和 `Mutex` API 只对 `Send` 数据起作用。Since they are the point at which data crosses thread boundaries, they are also the point of enforcement for `Send`.

把所有这些放在一起，Rust 程序员可以放心地获得 `Rc` 以及其它线程*不安全*类型的好处，如果意外地从一个线程发送到另一个，Rust 编译器将会输出：

```
`Rc<Vec<i32>>` cannot be sent between threads safely

```

灾难也就可以避免了。

### []()共享栈：“scoped”

*注意：这里提到的 API 是旧的，已经从标准库移除了。你可以从 [`crossbeam`](https://crates.io/crates/crossbeam) ([`scope()`的文档](http://aturon.github.io/crossbeam-doc/crossbeam/fn.scope.html)) 和 [`scoped_threadpool`](https://crates.io/crates/scoped_threadpool) ([`scoped()`的文档](http://kimundi.github.io/scoped-threadpool-rs/scoped_threadpool/index.html#examples:))中找到相同的功能*

到目前为止，我们看到的所有的模式都涉及在堆上创建线程间共享的数据结构。但是如果我们想要开启一些线程，使用栈帧上的数据呢？这将会很危险：

```
fn parent() {
    let mut vec = Vec::new();
    // fill the vector
    thread::spawn(|| {
        print_vec(&vec)
    })
}

```

子线程获取 `vec` 的一个引用，而 `vec` 驻留在 `parent` 的栈帧中。当 `parent` 退出时，栈帧也被弹出了，但是子线程并不知情。哎哟！

为了消除这种内存不安全的情况，Rust 基本的线程创建 API 看起来有点儿像这样：

```
fn spawn<F>(f: F) where F: 'static, ...

```

`'static` 限制大概的意思是，闭包中不允许借用的数据。这意味着像上面 `parent` 这样的函数将会产生一个错误：

```
error: `vec` does not live long enough

```

从本质上捕获 `parent` 栈帧弹出的可能性。灾难也就可以避免了。

但是也有另一种方式保证安全：确保父栈帧保持不变，直到子线程完成为止。这是 *fork-join* 编程的模式，通常用于分治并行算法。Rust 通过提供一个["scoped"](https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html)线程创建变体支持该场景：

```
fn scoped<'a, F>(f: F) -> JoinGuard<'a> where F: 'a, ...

```

与上面的 `spawn` API 相比有两点关键不同：

*   使用参数 `'a`，而不是 `'static`。该参数表示闭包 `f` 中所有借用的作用域。

*   返回值是一个 `JoinGuard`。顾名思义，`JoinGuard` 确保父线程连接（等待）子线程，通过在析构函数中执行隐式连接（如果还没有显式执行）。

将 `'a` 包含在 `JoinGuard` 中确保 `JoinGuard` **不能逃离避免借用数据的作用域**。换句话说，Rust 保证父线程等待子线程在弹出栈帧之前完成。

因此，通过调整我们之前的例子，我们可以修复漏洞，并满足编译器：

```
fn parent() {
    let mut vec = Vec::new();
    // fill the vector
    let guard = thread::scoped(|| {
        print_vec(&vec)
    });
    // guard destroyed here, implicitly joining
}

```

所以在 Rust 中，你可以自由地将栈数据借用到子线程中，相信编译器会检查是否足够同步。

### []()数据争用

至此，我们已经看到了足够多的证据，可以大胆地发表一个关于 Rust 并发方法的强有力的声明：**编译器防止所有的 *数据争用*。**


> 数据争用指的是任何涉及写操作的非同步并发访问。

这里的同步包括低层的原子指令。本质上，这是一种方法，你不会意外地在线程之间“共享状态”；所有对状态的（可变）访问必须通过*某种*形式的同步。

数据争用仅仅是一种（非常重要）竞态条件，但是通过阻止数据争用，Rust 通常还可以阻止其它更微妙的争用。例如，在不同的位置*原子性*更新很重要：其它线程要么看到所有更新，要么看不到任何更新。在 Rust 中，拥有对相关位置的 `&mut` 访问就能**保证更新的原子性**，因为其它线程不可能并发读访问。

有必要暂停一下，从更广泛的语言环境中考虑这个保证。许多语言通过垃圾回收提供内存安全。但是垃圾回收并不能阻止数据争用。

Rust 使用所有权和借用提供两个核心价值主张：

*   没有垃圾回收的内存安全
*   没有数据争用的并发

### []()未来

当 Rust 刚开始的时候，它将管道直接添加到语言中，对并发采取了非常固执己见的立场。

在今天的 Rust 中，并发*完全是*一个库的事情；本文介绍的所有内容，包括 `Send`，都定义在标准库中，也可以在外部库中定义。

这是非常令人兴奋的，因为这意味着 Rust 的并发可以不断演化，拥抱新的范式，处理新的漏洞。像 [syncbox](https://github.com/carllerche/syncbox) 和 [simple_parallel](https://github.com/huonw/simple_parallel) 这样的库正在迈出第一步，我们希望在接下来几个月里在这个领域大力投资。敬请期待！

