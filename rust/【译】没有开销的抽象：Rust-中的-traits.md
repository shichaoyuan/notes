原文链接：[Abstraction without overhead: traits in Rust](https://blog.rust-lang.org/2015/05/11/traits.html)

May 11, 2015 · Aaron Turon

[之前的文章](http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)已经介绍了 Rust 设计的两大支柱：

*   没有垃圾回收的内存安全
*   没有数据争用的并发

本文将会探讨第三大支柱：

*   **没有开销的抽象**

使得 C++ 非常适合系统编程的信条和品质就是零成本抽象原则：

> C++ 实现遵循零开销原则：不用的东西不需要负担成本 [Stroustrup, 1994]。进一步：你所使用的代码，你不可能手写得更好。
> 
> -- Stroustrup

该信条并不总是适用于 Rust，例如它曾经具有强制的垃圾收集。但是随着时间的推移，Rust 的发展方向越来越低层，现在零成本抽象（zero-cost abstraction）是一个核心原则。

Rust 中抽象的基石是 *traits*：

*   **traits 是 Rust 中唯一的接口概念**。一个 trait 可以被多种类型实现，实际上新的 trait 可以为现有类型提供实现。另一方面，当你想要对未知类型进行抽象时，traits 就是你指定类型一些具体内容的方法。

*   **traits 可以被静态分发**。就像 C++ 的模板一样，你可以让编译器为抽象的每种实例类型生成单独的副本。这又回到了 C++ 的信条“你所使用的代码，你不可能手写得更好”——抽象最终会被完全擦除。

*   **traits 可以被动态分发**。有时候你确实需要一个间接的方式，所以不能在运行时“擦除”抽象。当你想要在运行时分发时，也可以使用*相同*的接口概念 ——trait。

*   **除了简单的抽象，traits 还解决了各种附加问题**。它被用作类型的“标记”，就像[在之前的文章中](http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)介绍的 `Send` 标记。它可以被用作定义“扩展方法”——也就是为外部定义的类型添加方法。它在很大程度上消除了对传统方法重载的需要。并且为操作符重载提供了一种简单的方式。

总而言之，trait 系统是 Rust 的秘密武器，这使得 Rust 既有高级语言的人机工程学和表达感，又保留了对代码执行和数据表示的低层控制。

本文将会从较高的层面逐一介绍以上几点，让你了解如何实现这些设计目标，并且不纠缠于细节。

### []()背景：Rust 中的方法

> 在深入研究 traits 之前，我们需要看一下 Rust 语言的一个小而重要的细节：方法与函数的区别。

Rust 既提供了方法，也提供了独立的函数，这两者是密切相关的：

```
struct Point {
    x: f64,
    y: f64,
}

// a free-standing function that converts a (borrowed) point to a string
fn point_to_string(point: &Point) -> String { ... }

// an "inherent impl" block defines the methods available directly on a type
impl Point {
    // this method is available on any Point, and automatically borrows the
    // Point value
    fn to_string(&self) -> String { ... }
}

```

像上面 `to_string` 这样的方法被称为“固有”方法，这是因为它们：

* 与单个具体的“self”类型绑定（通过 `impl` 代码块头部指定）
* 对该类型的任意值都*自动*可用——也就是，与函数不同，固有方法总是“在作用域内”。

方法的第一个参数总是明确的“self”，具体取决于[所需的所有权级别](http://blog.skylight.io/rust-means-never-having-to-close-a-socket/)，可以是`self`、`&mut self` 或者 `&self`。使用面向对象编程熟悉的 `.` 符号调用方法，基于方法中 `self` 的形式*隐式借用* self 参数。

```
let p = Point { x: 1.2, y: -3.7 };
let s1 = point_to_string(&p);  // calling a free function, explicit borrow
let s2 = p.to_string();        // calling a method, implicit borrow as &p

```

方法及其自动借用是 Rust 人机工程学的一个重要方面，支持下面这样的“流畅”API：

```
let child = Command::new("/bin/cat")
    .arg("rusty-ideas.txt")
    .current_dir("/Users/aturon")
    .stdout(Stdio::piped())
    .spawn();

```

### []()Traits 是接口

接口指定一段代码对另一段代码的期望，使得各自可以独立切换。对于 traits 来说，这个规范主要围绕着方法。

以下面这个简单的哈希 trait 为例：

```
trait Hash {
    fn hash(&self) -> u64;
}

```

为给定类型实现该 trait，你必须提供一个签名匹配的 `hash` 方法：

```
impl Hash for bool {
    fn hash(&self) -> u64 {
        if *self { 0 } else { 1 }
    }
}

impl Hash for i64 {
    fn hash(&self) -> u64 {
        *self as u64
    }
}

```

与 Java、C# 或 Scala 这些语言不同的是，**可以为存在的类型实现新的 traits**（就像上面的 `Hash`）。这意味着可以事后创建抽象，应用在现有的库上。

与固有方法不同的是，trait 方法只有在 trait 存在时才有效。假设 `Hash` 在作用域内，你可以写 `true.hash()`，所以实现一个 trait 将会扩展类型的可用方法。

另外... 就是这样！定义和实现一个 trait 实际上只不过是抽象出一个满足多个类型的公共接口。

### []()静态分发

在使用 trait 这方面，事情就更有趣了。最常见的方法是通过*泛型*：

```
fn print_hash<T: Hash>(t: &T) {
    println!("The hash is {}", t.hash())
}

```

函数 `print_hash` 对于未知类型 `T` 是通用的，但是需要 `T` 实现了 `Hash` trait。这意味着我们可以使用 `bool` 和 `i64` 类型的值：

```
print_hash(&true);      // instantiates T = bool
print_hash(&12_i64);    // instantiates T = i64

```

**泛型会被编译掉，实现静态分发**。就像 C++ 的模板，编译器将会为上述代码生成*两个版本*的 `print_hash` 方法，每个版本对应一个具体的参数类型。这意味着对 `t.hash()` 的内部调用——实际使用的抽象——是零成本的：这将被编译为对相关实现的直接静态调用：

```
// The compiled code:
__print_hash_bool(&true);  // invoke specialized bool version directly
__print_hash_i64(&12_i64);   // invoke specialized i64 version directly

```

这种编译模型对 `print_hash` 之类的函数不是很有用，但是对更实际的哈希用法*非常*有用。假设我们也为相等比较引入一个 trait：

```
trait Eq {
    fn eq(&self, other: &Self) -> bool;
}

```

（这里对 `Self` 的引用将会被解析为实现 trait 的具体类型；在 `impl Eq for bool` 中将会指向 `bool`。）

然后我们可以定义一个哈希映射，泛化实现 `Hash` 和 `Eq` 的类型 `T`：

```
struct HashMap<Key: Hash + Eq, Value> { ... }

```

泛化的静态编译模型将带来许多好处：

*   每个具体 `Key` 和 `Value` 类型的 `HashMap` 将会生成一个具体的 `HashMap` 类型，这意味着 `HashMap` 可以在桶中内联键值（而不是间接）。这节省了空间和间接访问，并改善了缓存局部性。
*   `HashMap` 中的每个方法同样会生成专门的代码。这意味着没有分发调用 `hash` 和 `eq` 的额外成本。这也意味着优化器可以使用完全具体的代码——也就是说从优化器的角度来看，*没有抽象*。特别是，静态分发允许跨越泛型进行*内联*。

总之，就像 C++ 中的模板一样，泛型的这些方面意味着你可以编写相当高级的抽象，而这些抽象可以*保证*向下编译成“你不可能手写得更好”的完全具体的代码。

**但是与 C++ 模板不同的是，trait 的使用者提前进行了完全的类型检查**。也就是说，当你单独编译 `HashMap` 时，对于抽象 `Hash` 和 `Eq` traits 的类型正确性只检查*一次*，而不是在应用具体类型时反复检查。对于库作者来说这意味着更早更清晰的编译错误，对于使用者来说这意味着更少的类型检查开销（也就是，更快的编译）。

### []()动态分发

我们已经看过了一种编译模型，其中所有的抽象都被静态地编译掉。但是有时候抽象不仅仅是关于重用和模块化——**有时候抽象在运行时扮演着重要角色，这不能被编译掉**

例如，GUI 框架经常涉及响应事件的回调，例如鼠标点击：

```
trait ClickCallback {
    fn on_click(&self, x: i64, y: i64);
}

```

对于允许注册多个回调到单个事件的 GUI 元素也很常见。通过泛型，你可能会这样编写代码：

```
struct Button<T: ClickCallback> {
    listeners: Vec<T>,
    ...
}

```

但是问题很明显：这意味着每个按钮精确指定一个 `ClickCallback` 的实现，也就是说按钮的类型反映了 `ClickCallback` 的类型。这根本不是我们想要的！相反，我们想要单个 `Button` 类型关联一组*异构的*监听器，每个监听器可以是不同的具体类型，但是每个监听器都实现了 `ClickCallback`。

一个直接的困难是，如果我们讨论的是一组异构类型，*每个类型都有不用的大小*——那么我们如何布局内部向量呢？答案通常是：间接。我们将会在向量中存储回调的*指针*：

```
struct Button {
    listeners: Vec<Box<ClickCallback>>,
    ...
}

```

在这里我们就像使用类型那样使用 `ClickCallback` trait。实际上在 Rust 中，[traits *是*类型，但是“不确定大小”](http://smallcultfollowing.com/babysteps/blog/2014/01/05/dst-take-5/)，这意味着它只允许出现在 `Box`（指向堆）或 `&`（可以指向任意地方）这样的指针里面。

在 Rust 中，像 `&ClickCallback` 或 `Box<ClickCallback>` 这样的类型被称为“trait 对象”，它包含一个指向实现了 `ClickCallback` 的 `T` 类型实例的指针，*以及*一个 vtable：指向 trait 中每个方法对应 `T` 实现的指针（在这里就是 `on_click`）。这些信息足以在运行时正确分发方法调用，并且可以支持对所有 `T` 统一表示。所以 `Button` 只需要编译一次，而抽象存在与运行时。

静态分发和动态分发是互补的工具，各自适合不同的场景。**Rust 的 traits 提供了统一简单的接口表示，可以用最小、可预期的成本在两种风格中使用**。trait 对象满足 Stroustrup 的“现用现付”原则：你需要 vtables 的时候就有，但是当你不需要的时候，同一个 trait 也可以被静态编译。

### []()traits 的许多用途

上面已经介绍了一些 traits 的机制和基本用法，但是它在 Rust 中还扮演着许多其它重要角色。如下：

*   **闭包**。就像 `ClickCallback` trait，Rust 中的闭包只是一种特殊的 trait。关于这个主题，你可以从 Huon Wilson [深入的文章](http://huonw.github.io/blog/2015/05/finding-closure-in-rust/)中了解更多。

*   **条件 APIs**。通过范型可以有条件地实现一个 trait：

    ```
    struct Pair<A, B> { first: A, second: B }
    impl<A: Hash, B: Hash> Hash for Pair<A, B> {
        fn hash(&self) -> u64 {
            self.first.hash() ^ self.second.hash()
        }
    }

    ```

    在这里 `Pair` 类型仅在成员为哈希时实现了 `Hash`——允许单个 `Pair` 类型在不同的上下文使用，同时支持最大的 API 对每个上下文可用。这是一种常见的模式，Rust 内建支持自动生成某种类型“机械的”实现：

    ```
    #[derive(Hash)]
    struct Pair<A, B> { .. }

    ```

*   **扩展方法**。trait 可以为存在的类型（在别处定义的）扩展新的方法，类似于 C# 的扩展方法。这直接超出了 traits 的作用域规则：你只需在 trait 中定义新方法，为相关类型提供实现，*瞧*，方法就可用了。

*   **标记**。Rust 有一些区分类型的“标记”：`Send`、`Sync`、`Copy` 和 `Sized`。这些标记仅仅是空的 *traits*，既可以用作泛型，也可以用作 trait 对象。标记可以定义在库中，并且会自动提供 `#[derive]` 风格的实现：例如，如果一个类型的所有组件都是 `Send`，那么该类型也是。就像在[前面文章](http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)看到的，这些标记非常强大：`Send` 标记是 Rust 保证线程安全的方式。

*   **重载**。Rust 不支持传统的重载，也就是同一方法定义多个签名。但是 traits 提供了许多重载的好处：如果某个方法用 trait 泛型定义，那么它可以被任意实现该 trait 的类型调用。相对于传统的重载，这有两个优势：第一，这意味着重载不是[特别的](http://dl.acm.org/citation.cfm?id=75283)：一旦你理解了 trait，你就立即理解了任意 API 的重载模式。第二，它是*可扩展的*：通过提供新的 trait 实现，你可以有效地从方法下游提供新的重载。

*   **操作符**。Rust 允许在自定义类型上重载 `+` 这样的操作符。每个操作符都由对应标准库中的 trait 定义，实现该 trait 的类型就自动提供该运算符。

要点：**尽管 traits 看起来很简单，但是 trait 是一个统一的概念，支持广泛的用例和模式，而不需要额外的语言特性。**

### []()未来

语言进化的的主要方式之一就在于抽象设施中，Rust 也不例外：许多[1.0 后优先事项](http://internals.rust-lang.org/t/priorities-after-1-0/1901) 都是 trait 系统在某个方向上的扩展。以下是一些亮点。

*   **静态分发的输出**。现在，可以对函数的参数使用泛型，但是对返回结果不能使用：你不能声明“这个函数返回某个实现了 `Iterator` trait 的类型”，然后将该抽象编译掉。当你想要返回一个静态分发的闭包时，这尤其成为问题——在当前版本的 Rust 中这是无法做到的。我们想要实现该特性，并且[已经有了一些想法](https://github.com/rust-lang/rfcs/pull/105)。

*   **专门化**。Rust 不允许 trait 实现重叠，所以对执行的代码不会有歧义。另一方面，在某些情况下你可以为广泛使用的类型提供一个“通用”实现，但随后你又想为一些情况提供更专门的实现，通常是出于性能原因。我们希望在不久的将来提供一个设计方案。

*   **更高级的类型**（HKT）。traits 当前只能应用于*类型*，而不能应用于*类型构造器*——也就是，只能应用于 `Vec<u8>` 这样，而不能应用于 `Vec` 本身。该限制使得很难提供一组良好的容器 traits，因此当前的标准库中并没有包含这些。HKT 是一个主要的横切特性，这将代表 Rust 的抽象能力向前迈了一大步。

*   **有效的重用**。最后，虽然 trait 提供了一些代码重用机制（我们在上面没有提到），但是仍然有一些重用模式不适应当前版本的 Rust ——特别是 DOM、GUI框架和许多游戏中发现的面向对象层次结构。在不增加太多重叠或复杂性的情况下适应这些场景是一个非常有趣的设计问题，Niko Matsakis 已经开启了一个单独的[博客系列](http://smallcultfollowing.com/babysteps/blog/2015/05/05/where-rusts-enum-shines/)探讨这个问题。目前还不清楚是否可以通过 traits 实现，或者是否需要一些其它的材料。

当然，我们还处在 1.0 发布的前夕，这还需要一些时间才能尘埃落定，社区需要有足够的经验来开启这些扩展。但这正是参与进来的好时机：从影响早期阶段的设计，到进行实现，一直到在代码中尝试不同的用例——我们希望得到你的帮助！
