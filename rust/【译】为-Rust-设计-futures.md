原文链接：[Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)

07 Sep 2016 · Aaron Turon

我[最近写了](http://aturon.github.io/blog/2016/08/11/futures/)一篇关于 Rust 中异步 I/O 重要性和新 [futures](https://github.com/alexcrichton/futures-rs) 库目标的文章。本文将通过解释该库的核心*设计*来深化这个主题。如果你想了解更多该库的*用法*，你不得不等一下；我们正在非常积极地开发 [Tokio 栈](http://aturon.github.io/blog/2016/08/26/tokio/)，一旦它稳定下来我们将会讲更多。

总的来说，**目标是没有性能损失的、鲁棒的、符合人机工学的异步 I/O**：

*   **鲁棒性**：该库对错误处理、取消、超时、背压以及编写鲁棒的服务的其它典型问题应该有充分的考虑。这就是 Rust，当然我们也[保证线程安全](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)。

*   **符合人机工学**：该库应该使得编写异步代码尽可能轻松——理想情况下应该像编写同步代码一样容易，但是具有更强的表达性。虽然后者需要  [`async`/`await`](https://en.wikipedia.org/wiki/Await) 才能完全实现，但是 futures 库提供了一种表达和组合异步计算的高级方式，类似于 Rust 成功的 [`Iterator` API](https://static.rust-lang.org/doc/master/std/iter/trait.Iterator.html)。

*   **零成本**：用该库编写的代码应该编译成相当于（或者优于）“手写的”服务实现，后者通常使用手写的状态机和仔细的内存管理。

实现这些目标需要结合 Rust 中现有的技术，以及一些关于如何构建 futures 库的新想法；本文将涵盖这两方面。简而言之：

*   **利用 Rust 的 trait 和闭包实现人机工学和成本避免**。Rust 中的 traits 和闭包*不*需要堆分配和动态分发——我们充分利用了这一点。我们还使用 trait 系统对 futures API 进行了简单方便的打包。

*   **将核心 `Future` 抽象设计为 *需求驱动*，而不是面向回调**。（按照异步 I/O 的术语，也就是遵循“准备就绪”风格，而不是“完成”风格。）这意味着将 futures 组合在一起并不会创建中间回调。正如我们将看到的，这种方法对背压和取消也有好处。

*   **提供一个 *任务* 抽象，类似于一个绿色线程，这驱动 future 完成**。将 futures 存放在任务中使得库代码可以编译成传统模型，也就是说，使用大状态机做为大量底层事件的回调。

让我们来一探究竟吧！

## 背景：Rust 的 traits

> 我们先快速回顾一下 Rust 的 traits。如果你想要更多关于这些主题的材料，你可以看一下更长的 [traits 综述](https://blog.rust-lang.org/2015/05/11/traits.html)。

要理解 futures 是如何设计的，你需要对 Rust 的 traits 有基本的了解。在这里我并不想做一个完整的介绍，但是我将会尽力介绍最相关的重点。

traits 提供了 Rust 中唯一的*接口*概念，这意味着 trait 是一个可以应用于许多具体类型的抽象。例如，这是一个简化的哈希 trait：

```
trait Hash {
    fn hash(&self) -> u64;
}

```

该 trait 规定实现它的类型必须提供一个 `hash` 方法，该方法[借用](http://blog.skylight.io/rust-means-never-having-to-close-a-socket/) `self` 并发返回一个 `u64`。要实现这个 trait，你需要为这个方法提供一个具体定义，就像下面这个简单的：

```
impl Hash for bool {
    fn hash(&self) -> u64 {
        if *self { 0 } else { 1 }
    }
}

impl Hash for i32 { ... } // etc

```

一旦提供了这些实现，你可以像 `true.hash()` 这样直接调用该方法。但通常是通过*泛型*调用该方法，这是 traits 真正作为抽象的方式：

```
fn print_hash<T: Hash>(t: &T) {
    println!("The hash is {}", t.hash())
}

```

函数 `print_hash` 对于未知类型 `T` 是通用的，但是需要 `T` 实现了 `Hash` trait。这意味着我们可以使用 `bool` 和 `i32` 类型的值：

```
print_hash(&true);   // instantiates T = bool
print_hash(&12);     // instantiates T = i32

```

**泛型会被编译掉，实现静态分发**。就像 C++ 的模板，编译器将会为上述代码生成*两个版本*的 `print_hash` 方法，每个版本对应一个具体的参数类型。这意味着对 `t.hash()` 的内部调用——实际使用的抽象——是零成本的：这将被编译为对相关实现的直接静态调用：

```
// The compiled code:
__print_hash_bool(&true);  // invoke specialized bool version directly
__print_hash_i32(&12);     // invoke specialized i32 version directly

```

对于使得 futures 这样的抽象没有开销，编译成非泛型代码是至关重要的：大部分情况下，非泛型代码也会被内联，让编译器生成和优化大块代码，这些代码类似于你曾经用低级的、“手卷的”风格编写的代码。

Rust 中的闭包也是这样工作的——实际上，闭包就是 traits。特别是，这意味着创建一个闭包不需要进行堆分配，调用一个闭包是静态分发的，就像上面的 `hash` 方法。

最后，traits *也*可以作为“对象”使用，这将导致 trait 方法被*动态*分发（编译器无法直接知道将使用哪个实现）。trait 对象的优势在于*异构集合*，在这种集合中，你需要将许多具有不同底层类型但都实现了相同 trait 的对象组合在一起。trait 对象必须总是在指针后面，在实践中通常需要堆分配。

## 定义 futures

现在，让我们转向 futures。 [之前的文章](http://aturon.github.io/blog/2016/08/11/futures/)给出了一个 future 的非正式定义：

> 本质上，一个 future 表示一个还没准备好的值。通常来说，future *完成*（值准备好了）是由于其它地方产生的事件。

显然，我们想要将 futures 定义为某种 trait，是因为有许多不同种类的“还没准备好的值”（比如，套接字上的数据、RPC 调用的返回值等）。但是我们如何表示“还没准备好”的部分呢？

### 错误的开始：回调（即基于完成的）方法

从我们调研过的每个现有 futures 实现中，我们发现有一个非常标准的方式来描述 futures：作为订阅*回调*以通知 future 完成的函数。

*   **注意**：在异步 I/O 世界中，这种接口有时被称为*基于完成的*接口，因为事件在操作完成时被标记；[Windows 的 IOCP](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365198(v=vs.85).aspx) 就是基于该模型。

在 Rust 中，回调模型将引出如下的 trait：

```
trait Future {
    // The type of value produced by the future
    type Item;

    // Tell the future to invoke the given callback on completion
    fn schedule<F>(self, f: F) where F: FnOnce(Self::Item);
}

```

这里的 `FnOnce` 是闭包的 trait，表示最多被调一次。因为 `schedule` 使用了泛型，所以它将会静态分发对闭门的调用。

很不幸，**这种方法在每次 future 组合时几乎都会强制分配，并且经常强制动态分发**，尽管我们尽了最大的努力来避免这些开销。

想要了解原因，让我们考虑一种组合两个 futures 的基本方式：

```
fn join<F, G>(f: F, g: G) -> impl Future<Item = (F::Item, G::Item)>
    where F: Future, G: Future

```

该函数接收两个 futures，`f` 和 `g`，返回一个新的 future，输出两者的结果。这个 `join` 的 future 只有在*两个*底层 futures 都完成的时候才完成，但是允许底层 futures 在此之前并发执行。

我们如何使用上述 `Future` 定义实现 `join` 呢？这个 `join` 的 future 将会提供一个单独的回调 `both_done`。但是底层的 futures 想要各自的回调 `f_done` 和 `g_done`，只接收自己的结果。很显然，我们在这里需要某种*共享*：我们需要构建 `f_done` 和 `g_done`，以便任意一个可以调用 `both_done`，也得确保包含适当的同步。考虑到涉及的类型签名，根本没办法不进行分配（在 Rust 中，我们在这里需要使用 `Arc`）。

这类问题在许多 futures 组合子中都会出现。

另一个问题是像事件循环这样的事件源需要调用任意不同类型的回调——例如上面提到的异构情况。举一个具体的例子，当套接字准备好读取时，这个事件需要分发给*某个*回调，通常需要混合不同的 futures 来处理不同的套接字。为此，你最终需要为事件循环*在每次 future 想要监听事件时*进行堆分配回调，并且动态分发通知给这些回调。

TL;DR，我们无法使“标准” future 抽象提供零成本组合，并且据我们所知，没有“标准”实现可以做到。

### 成功的方法：需求驱动的（即基于准备就绪的）方法

经过一番深思熟虑，我们得出了一个新的“需求驱动”的 future 定义。这是一个**简化**的版本，忽略了[真实 trait](http://alexcrichton.com/futures-rs/futures/trait.Future.html) 的错误处理：

```
// A *simplified* version of the trait, without error-handling
trait Future {
    // The type of value produced on success
    type Item;

    // Polls the future, resolving to a value if possible
    fn poll(&mut self) -> Async<Self::Item>;
}

enum Async<T> {
    /// Represents that a value is immediately ready.
    Ready(T),

    /// Represents that a value is not ready yet, but may be so later.
    NotReady,
}

```

这里的 API 改变很直观：与 future 在完成时主动调用回调不同，一个外部组件必须*轮询* future 以驱动它完成。future 可以通过返回 `Async::NotReady`（一个 `EWOULDBLOCK` 的抽象）来表示它还没准备好，必须稍后重新轮询。

*   **注意**：在异步 I/O 世界，这种接口有时被称为*基于准备就绪*的接口，因为事件通知基于操作的“准备就绪”（例如，套接字上的字节准备就绪），然后尝试完成操作；[Linux 的 epoll](http://man7.org/linux/man-pages/man7/epoll.7.html) 就是基于该模型。（该模型也可以表达完成，通过将操作的完成看做 future 准备好轮询的信号。）

通过消除所有中间回调，我们已经解决了前一个版本 trait 的几个关键问题。但是我们引入了一个新的问题：在 `NotReady` 返回之后，谁来轮询该 future，以及何时进行轮询？

让我们举一个具体的例子。如果一个 future 尝试从套接字读取字节，该套接字可能还没准备好读取，在这种情况下 future 返回 `NotReady`。*无论如何*，一旦套接字准备就绪，我们必须安排 future 稍后“唤醒”（通过调用 `poll`）。这种唤醒是事件循环的任务。但是现在我们需要某些方式将事件循环中的信息传回来，以继续轮询 future。

这个解决方案形成了另一个主要组件：任务。

### 基石：任务

***任务*是被执行的 future**。future 几乎总是由一系列其它 futures 组成，就像这个来自原文的例子：

```
id_rpc(&my_server).and_then(|id| {
    get_row(id)
}).map(|row| {
    json::encode(row)
}).and_then(|encoded| {
    write_string(my_socket, encoded)
})

```

关键是像 `and_then`、`map` 和 `join` 这种组合 futures 成为更大 futures 的函数与*执行* futures 的函数之间是有差异的，如：

*   `wait` 方法，它只是将 future 当作固定在当前线程的任务执行，阻塞该线程直至产生并返回结果。

*   线程池中的 `spawn` 方法，它将 future 作为池中的独立任务来执行。

这些*执行*函数创建包含 future 的任务，负责轮询它。在 `wait` 的情况下，轮询会立即进行；对于 `spawn`，轮询会在任务*调度*给工作线程时进行。

尽管轮询开始了，如果任意一个内部 futures 产生一个 `NotReady`，它将使整个任务暂停——这个任务可能需要等待某个事件发生后才能继续。在同步 I/O 中，这是线程将会阻塞的情况。任务提供了一个与该模型对等的行为：**在将自身注册为其等待事件的回调函数之后**，任务通过让出它的执行线程来实现“阻塞”。

回到从套接字读取数据的例子，对于一个 `NotReady`，任务可以添加到事件循环的分发表中，使得在套接字准备就绪时，它将会被唤醒，此时将重新 `poll` 它的 future。但是至关重要的是任务实例在它执行的 future 的生命周期中保持不变——**所以不需要分配来创建或注册回调**。

类似于线程，任务提供了 `park`/`unpark` API 用于“阻塞”和唤醒：

```
/// Returns a handle to the current task to call unpark at a later date.
fn park() -> Task;

impl Task {
    /// Indicate that the task should attempt to poll its future in a timely fashion.
    fn unpark(&self);
}

```

阻塞一个 future 就是使用 `park` 获取任务的句柄，将 `Task` 放到某个感兴趣事件的唤醒队列中，然后返回 `NotReady`。当感兴趣的事件发生时，`Task` 句柄可以用于唤醒任务，例如，重新调度它在线程池中执行。`park`/`unpark` 准确的机制因任务执行器而异。

在某种程度上，任务模型是“绿色”（即轻量）线程的一个实例：我们将潜在的大量异步任务调度到数量少得多的实际 OS 线程上，这些任务中的大部分在绝大多数时间会在某个事件上被阻塞。然而，与 Rust [旧的绿线程模型](https://github.com/aturon/rfcs/blob/remove-runtime/active/0000-remove-runtime.md)有一个本质的区别：**任务不需要它们自己的栈**。实际上，任务需要的所有数据都包含在它的 future 中。这意味着可以巧妙地避开动态栈增长和栈交换的问题，在没有任何运行时系统协助的情况下给我们提供真正轻量级的任务。

也许令人惊讶的是，**任务中的 future 会被编译成状态机**，所以每次任务唤醒继续轮询的时候，它从当前状态继续执行——就像基于 [mio](http://github.com/carllerche/mio)的手写代码一样。这一点在例子中很容易看到，所以让我们再看看 `join`。

### 例子：需求驱动模型中的 `join`

为了实现 `join` 函数，我们将引入一个新的具体类型，`Join`，用于追踪必要的状态：

```
fn join<F: Future, G: Future>(f: F, g: G) -> Join<F, G> {
    Join::BothRunning(f, g)
}

enum Join<F: Future, G: Future> {
    BothRunning(F, G),
    FirstDone(F::Item, G),
    SecondDone(F, G::Item),
    Done,
}

impl<F, G> Future for Join<F, G> where F: Future, G: Future {
    type Item = (F::Item, G::Item);

    fn poll(&mut self) -> Async<Self::Item> {
        // navigate the state machine
    }
}

```

首先要注意的是，`Join` 是一个*枚举*，它的变量表示 “join 状态机”中的状态：

*   `BothRunning`： 这两个底层 futures 仍在执行。
*   `FirstDone`： 第一个 future 已经产生了值，但是第二个仍在执行。
*   `SecondDone`： 第二个 future 已经产生了值，但是第一个仍在执行。
*   `Done`： 两个 futures 都完成了，并且他们的值都返回了。

Rust 中的枚举不需要任何指针或堆分配；相反，枚举的大小是最大变量的大小。这正是我们想要的——大小表示这个小状态机的“高水位线”。

这里的 `poll` 方法将尝试通过 `poll` 相应的底层 futures 来执行状态机。

回想一下，`join` 的目的是允许两个 futures 并发处理，竞相完成。例如，两个 futures 可能各自表示线程池中并行执行的子任务。当这些子任务仍在运行时，`poll` 它们的 futures 将返回 `NotReady`，实际上“阻塞”了 `Join` future，同时为环绕的任务存储一个句柄，以便在它们完成时将其唤醒。这两个子任务可以竞相*唤醒* `Task`，但是这可以：**唤醒任务的 `unpark` 方法是线程安全的，保证任务在任何 `unpark` 调用后至少 `poll` 一次它的 future**。因此，同步在任务层面是一次性处理的，不需要 `join` 这样的组合子自己分配或处理同步。

*   你可能已经注意到了 `poll` 接收 `&mut self`，这意味着一个给定的 future 不能被并发地 `poll`——在轮询时 future 对内容有唯一的访问权。`unpark` 同步可以保证这一点。

最后一点，`join` 这样的组合子包含“小”状态机，但是因为其中某些状态涉及附加的 futures，所以它允许*嵌套*附加的状态机。换句话说，在进入 `Join` 状态机之前，`poll` 其中一个底层 futures 可能涉及逐句通过*它的*状态机。**事实上 `Future` trait 不需要堆分配或动态分发，这是使其高效的关键。**

一般来说，由任务运行的“大” future——由组合子连接的一大串 futures 组成——就以这种方式体现了“大”嵌套状态机。再一次，Rust 的枚举表示意味着所需的空间是“大”状态机中最大的状态大小。这个“大” future 的空间由任务*一次性*分配，要么在栈上（用于 `wait` 执行器），要么在堆上（用于 `spawn`）。终究，数据必须驻留在*某个地方*——但关键是要避免随着状态机执行而不断分配，而是要预先为整体留出空间。

## 大规模 futures

我们已经了解了需求驱动 futures 的基本原理，但是仍有一些关于*鲁棒性*的担忧。事实证明需求驱动模型可以很自然地解决这些担忧。让我们看一下其中几个重要的担忧。

### 取消

futures 经常用于表示并发执行的大量工作。有时很明显这个工作不再需要了，可能是由于超时，或者客户端关闭了连接，或者以其它方式找到了所需的应答。

在这种情况下，你想要某种形式的*取消*：告诉 future 停止执行的能力，因为你对其结果不再感兴趣了。

在需求驱动模型中，取消基本上就是“falls out”。你所要做的就是停止轮询 future，而不是“dropping”它（Rust 的术语，指的是销毁数据）。这样做通常是 `Join` 这样的嵌套状态机的自然结果。如果 futures 的计算需要一些特殊测操作来取消（比如取消一个 RPC 调用），那么可以在它们的 `Drop` 实现中提供该逻辑。

### 背压

大规模使用 futures（以及相近的 streams）的另一个重要方面是*背压*：系统中某个部分的过载组件减慢来自其它组件输入的能力。例如，如果服务器有积压的数据库事务，它就应该减慢接收新请求。

与取消类似，背压基本上  falls out 我们的 futures、streams 模型。这是因为任务可能会被返回 `NotReady` 的 future/stream 无限期地“阻塞”，并被通知稍后继续轮询。例如数据库事务，如果事务排队本身表示为 future，那么数据库服务可以通过返回 `NotReady` 来减慢请求。通常来说这些 `NotReady` 会通过系统向后级联，例如，让背压从数据库服务流回特定的客户端连接，然后回到整个连接管理器。这种级联是需求驱动模型的自然结果。

### 传递唤醒的原因

如果你熟悉 [epoll](http://man7.org/linux/man-pages/man7/epoll.7.html) 这样的接口，你可能已经注意到对于 `park`/`unpark` 模型少了一些东西：它无法让任务知道*为什么*被唤醒。

对于并发轮询大量其它 futures 的某种 futures 来说这是一个问题——你不希望必须重新轮询*所有*才能发现哪个 sub-future 实际上能够执行。

为了解决这个问题，该库提供了一种“epoll for everyone”的功能：将“unpark 事件”与给定 `Task` 句柄关联起来的能力。也就是说，可能到处都有指向同一任务的各种句柄，所有这些句柄都可以用于唤醒任务，但是每个带有不同的 unpark 事件。当唤醒的时候，任务中 future 可以检查这些 unpark 事件以确定发生了什么。请看[文档](http://alexcrichton.com/futures-rs/futures/task/fn.with_unpark_event.html)以获取等多详细信息。

## 总结

现在我们已经看过了 Rust futures 和 streams 库背后的核心设计原则。总结一下，可以归纳为几个核心理念：

*   将执行的 futures 封装为*任务*，这将做为单一的、永久的“回调”。

*   以需求驱动的方式实现 futures，而不是面向回调的方式。

*   使用 Rust 的 trait 系统，使得组合的 futures 可以转化为大状态机。

总之，这些理念产生了一个鲁棒的、符合人机工学的、零成本的 futures 库。

正如我在文章开头提到的，我们正积极地在基本 futures 库上层进行开发——这一层包含了特定的 I/O 模型（比如 [mio](http://github.com/carllerche/mio)），也提供了构建服务的更高级的工具。这些是 Tokio 项目的一部分，你可以从[我之前的文章](http://aturon.github.io/blog/2016/08/26/tokio/)中读到更多关于整体愿景的内容。随着这些 API 稳定，期待看到更多介绍它们的文章！
