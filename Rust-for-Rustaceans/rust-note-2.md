《Rust for Rustaceans》读书笔记

—— 为什么设计 Rust 接口？



简而言之想要写出可靠的、可用的接口，需要遵循四个准则：*unsuprising*、*flexible*、*obvious* 和 *constrained*。



## 1. Unsuprising

你的接口应该足够直观，如果用户不得不猜，通常可以猜对。

主要可以通过 naming、common traits 和 ergonomic trait tricks 这三个方面实施。



### 1.1 Naming Practices

使用通常的命名，那么用户可以做出合理的假设。例如

1. 命名为 **iter** 的方法很可能接收参数 **&self**，返回一个迭代器；

   ```rust
   impl<T> [T] {
   	/// Returns an iterator over the slice.
       ///
       /// The iterator yields all items from start to end.
       ///
       /// # Examples
       ///
       /// ```
       /// let x = &[1, 2, 4];
       /// let mut iterator = x.iter();
       ///
       /// assert_eq!(iterator.next(), Some(&1));
       /// assert_eq!(iterator.next(), Some(&2));
       /// assert_eq!(iterator.next(), Some(&4));
       /// assert_eq!(iterator.next(), None);
       /// ```
       #[stable(feature = "rust1", since = "1.0.0")]
       #[inline]
       pub fn iter(&self) -> Iter<'_, T> {
           Iter::new(self)
       }
   }
   ```

2. 命名为 **into_inner** 的方法很可能接收参数 **self**，返回被包装的类型；

   ```rust
   impl<T> Cell<T> {
       /// Unwraps the value.
       ///
       /// # Examples
       ///
       /// ```
       /// use std::cell::Cell;
       ///
       /// let c = Cell::new(5);
       /// let five = c.into_inner();
       ///
       /// assert_eq!(five, 5);
       /// ```
       #[stable(feature = "move_cell", since = "1.17.0")]
       #[rustc_const_unstable(feature = "const_cell_into_inner", issue = "78729")]
       pub const fn into_inner(self) -> T {
           self.value.into_inner()
       }
   }
   ```

3. 命名为 **SomethingError** 的类型很可能实现了 **std::error::Error**，并且出现在 **Result** 中。



如果不遵守这些通常的命名规则，那么势必会给用户造成惊吓，让用户不得不浪费时间理解你的特殊命名。



### 1.2 Common Traits for Types

Rust 用户通常会假设所有接口都能工作，例如可以通过 {:?} 打印，可以在线程间传递等，所以我们应该尽可能提前实现大部分标准 traits，即使我们不需要。

这个要求的一大原因是 coherence rule，也就是编译器不允许为 foreign type 实现 foreign trait，当然你可以写一个包装类型，这样很繁琐又很难实现。

通常要实现的 trait 有：

1. **Debug**；
2. **Send** 和 **Sync**（还有 **Unpin**），如果没有实现需要特殊说明；
3. **Clone** 和 **Default**；
4. **PartialEq**、**PartialOrd**、**Hash**、**Eq**、**Ord**；
5. serde crate 的 **Serialize** 和 **Deserialize**。



### 1.3 Ergonomic Trait Implementations

即使某类型实现了某 trait，Rust 也不能自动为该类型的引用实现该 trait，据个例子，即使 `Bar: Trait`，你也不能用 `&Bar` 调用 `fn foo<T: Trait>(t: T)`。其中的原因是 `Trait` 中可能包含接受`&mut self` 和 `self` 的方法，这明显不能用 `&Bar` 调用。

所以按需自己定义，例如 `IntoIterator`：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T> IntoIterator for &'a [T] {
    type Item = &'a T;
    type IntoIter = Iter<'a, T>;

    fn into_iter(self) -> Iter<'a, T> {
        self.iter()
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T> IntoIterator for &'a mut [T] {
    type Item = &'a mut T;
    type IntoIter = IterMut<'a, T>;

    fn into_iter(self) -> IterMut<'a, T> {
        self.iter_mut()
    }
}
```



### 1.4 Wrapper Tyeps

Rust 没有面向对象那种继承，然而 `Deref` 和 `AsRef` 两个 trait 提供了类似的行为。

```rust
#[lang = "deref"]
#[doc(alias = "*")]
#[doc(alias = "&*")]
#[stable(feature = "rust1", since = "1.0.0")]
#[rustc_diagnostic_item = "Deref"]
pub trait Deref {
    /// The resulting type after dereferencing.
    #[stable(feature = "rust1", since = "1.0.0")]
    #[rustc_diagnostic_item = "deref_target"]
    #[lang = "deref_target"]
    type Target: ?Sized;

    /// Dereferences the value.
    #[must_use]
    #[stable(feature = "rust1", since = "1.0.0")]
    #[rustc_diagnostic_item = "deref_method"]
    fn deref(&self) -> &Self::Target;
}
```

```rust
#[stable(feature = "rust1", since = "1.0.0")]
#[cfg_attr(not(test), rustc_diagnostic_item = "AsRef")]
pub trait AsRef<T: ?Sized> {
    /// Converts this type into a shared reference of the (usually inferred) input type.
    #[stable(feature = "rust1", since = "1.0.0")]
    fn as_ref(&self) -> &T;
}
```



如果你提供了一个相对透明的包装类型，那么最好实现一下 `Deref`，这样用户就能用 `.` 操作符之间调用内部类型的方法。如果访问内部类型不需要复杂、缓慢的逻辑，你也应该考虑实现 `AsRef`，这样就能用 `&WrapperType` 作为 `&InnerType`。对于大部分包装类型，你也应该实现 `From<InnerType>` 和 `Into<InnerType>`，这样就很容易添加和移除包装。

以 `Rc` 为例：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for Rc<T> {
    type Target = T;

    #[inline(always)]
    fn deref(&self) -> &T {
        &self.inner().value
    }
}

#[stable(since = "1.5.0", feature = "smart_ptr_as_ref")]
impl<T: ?Sized> AsRef<T> for Rc<T> {
    fn as_ref(&self) -> &T {
        &**self
    }
}

#[cfg(not(no_global_oom_handling))]
#[stable(feature = "from_for_ptrs", since = "1.6.0")]
impl<T> From<T> for Rc<T> {
    /// Converts a generic type `T` into an `Rc<T>`
    ///
    /// The conversion allocates on the heap and moves `t`
    /// from the stack into it.
    ///
    /// # Example
    /// ```rust
    /// # use std::rc::Rc;
    /// let x = 5;
    /// let rc = Rc::new(5);
    ///
    /// assert_eq!(Rc::from(x), rc);
    /// ```
    fn from(t: T) -> Self {
        Rc::new(t)
    }
}
```



还有一个 `Borrow` trait，跟 `Deref` 和 `AsRef` 类似，只是使用场景更窄一些。

```rust
#[stable(feature = "rust1", since = "1.0.0")]
#[rustc_diagnostic_item = "Borrow"]
pub trait Borrow<Borrowed: ?Sized> {
    /// Immutably borrows from an owned value.
    ///
    /// # Examples
    ///
    /// ```
    /// use std::borrow::Borrow;
    ///
    /// fn check<T: Borrow<str>>(s: T) {
    ///     assert_eq!("Hello", s.borrow());
    /// }
    ///
    /// let s = "Hello".to_string();
    ///
    /// check(s);
    ///
    /// let s = "Hello";
    ///
    /// check(s);
    /// ```
    #[stable(feature = "rust1", since = "1.0.0")]
    fn borrow(&self) -> &Borrowed;
}
```

定义跟`AsRef`几乎一模一样，但是`Borrow`有个潜在的语义，目标类型的 `Hash`、`Eq`、`Ord`与实现类型的是等效的。

```rust
impl<K, V, S> HashMap<K, V, S>
where
    K: Eq + Hash,
    S: BuildHasher,
{

    #[stable(feature = "rust1", since = "1.0.0")]
    #[inline]
    pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        self.base.get(k)
    }


}
```

例如 `HashMap`这里只能用 `Borrow`。



## 2. Flexible

写一个接口实质上是定一个契约，这个契约包含两个方面：

1. 要求：关于如何使用代码的限制；
2. 承诺：关于如何使用代码的保证。

**一个经验之谈就是避免强加不必要的限制，只许下你能兑现的承诺。**

在 Rust 中限制以 trait bound 和 argument type 的形式给出；承诺以 trait implementation 和 return type 的形式给出。

例如这三个函数，功能是一样的，但是定义的契约是不同的：

```rust
fn frobnicate1(s: String) -> String
fn frobnicate2(s: &str) -> Cow<'_, str>
fn frobnicate3(s: impl AsRef<str>) -> impl AsRef<str>
```

第一个函数要求调用者传递 `String` 类型参数的所有权，并且返回 owned ` String`，所以以这个契约我们之后无法以向后兼容的方式使得该函数 allocatoin-free。

第二个函数放松了这个契约，调用者只需提供一个字符串引用，所以使用者不需要再分配或者放弃一个 `String` 的所有权，该函数承诺返回一个 `Cow`，这意味着可以返回一个字符串引用或者一个 owend `String`。

第三个函数就没有这些限制，该函数只要求参数类型和返回类型只要能生成字符串引用即可。

这三个定义没有好坏之分，主要看使用场景和需求。



### 2.1 Generic Arguments

使用泛型，而不具体的参数，会更灵活，但是也会更难阅读和理解。

**一个经验之谈就是如果你能想到用户可能合理地经常地使用其它类型，而不是开始设定的具体类型，那么就将该参数泛型化。**

Rust 的泛型是编译期静态分发的，会增大编译后二进制文件的大小，这个问题可以通过 trait object 动态分发解决，也就将 `impl AsRef<str>` 替换为 `&dyn AsRef<str>`。但是动态分发也是有问题的：1. 有一些性能损失；2. trait object 不能作复杂的限制，例如 `&dyn Hash + Eq` 这样是不合法的，只能用泛型参数。

所以开始我们可以用具体的类型定义接口，然后按照需要泛型化，但是要注意这可能会破坏向后兼容。

据个例子，我们将函数定义由 `fn foo(v: &Vec<usize>)` 改为 `fn foo(v: impl AsRef<[usize]>)`，是放松了参数限制，但是在类型推导的时候可能会有问题。



### 2.2 Object Safety

当你定义一个 trait 的时候，最好定义为 object-safe 的。

> To be object-safe, none of a trait's methods can be generic or use the Self type.

如果你确实需要定义一个泛型方法，那么有两个办法：

1. 考虑一下泛型参数是否可以放在 trait 自身中？或者是否可以使用动态分发？
1. 在方法上添加 `where Self: Sized` 这个 trait bound。

`Iterator` 是个很好的例子，它是 object-safe 的，但是又有“泛型方法”：

```rust
pub trait Iterator {
    /// The type of the elements being iterated over.
    #[stable(feature = "rust1", since = "1.0.0")]
    type Item;
    
    #[lang = "next"]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn next(&mut self) -> Option<Self::Item>;
    
    #[inline]
    #[unstable(feature = "iter_next_chunk", reason = "recently added", issue = "98326")]
    fn next_chunk<const N: usize>(
        &mut self,
    ) -> Result<[Self::Item; N], array::IntoIter<Self::Item, N>>
    where
        Self: Sized,
    {
        array::iter_next_chunk(self)
    }

    ...
}
```



trait 是否必须定义为 object-safe 的？这很难回答，主要还是看情况。但是如果因为添加方法造成不兼容，那么记得升主版本号。



### 2.3 Borrowed vs. Owned

借用还是所有，你在定义函数、trait、类型的时候都需要做一个决定。

1. 如果你写的代码需要数据的所有权，比如说调用的方法需要`self`，或者需要将数据转移到另一个线程，那么它必须保存所有的数据。这种情况应该让调用者提供所有的数据，而不是通过引用克隆数据。这样调用者就能控制数据分配，并且知晓使用该接口的成本。
2. 如果你写的代码不需要数据的所有权，那么应该使用引用。当然对于 `i32`、`bool`、`f64`这类小类型除外，copy 自身和引用的成本是一样的。
3. 如果你不确定的时候，那么使用 `Cow`，在返回类型时很常用，但是方法参数用的很少。

还有一种情况，如果你定义的接口让 lifetime 搞得太复杂，以至于很难使用，那么你应该考虑一下是否持有所有权了。



### 2.4 Fallible and Blocking Destructors

有一些 I/O 相关的类型会在 `Drop` 中进行一些清理操作，比如刷写到磁盘，关闭文件或者关闭连接等，但是此时发生异常外部是获取不到的，所以只能尽最大的努力将 `Drop` 实现完善。

当然还有一种更可控的方式，那就是提供一个显式的 destructor，方法的形式为接收 `self` 所有权，返回错误 `-> Result<_, _>`。这也有两个问题：1. 因为你的类型实现了 `Drop`，那么不能在 destructor 中移动类型中的属性；2. drop 接收 `&mut self`，而不是 `self`，所以在 `Drop` 中不能不能简单的调用 destructor。



## 3. Obvious

虽然某些用户可能了解你的接口的基本实现，但是他们不可能理解所有的规则和限制，只有遇到问题时他们才会找文档阅读类型签名，所以我们需要尽可能让用户容易理解你的接口，并且很难不正确的使用。

文档和类型系统是两项主要的技术，用于解决这个问题。



### 3.1 Documentation

写好文档首先有四个建议：

1. 写情况各种异常的场景，以及除了类型签名，用户还需要考虑的事情。例如 panic 的情况，返回的 error 类型，或者 unsafe 函数的额外要求。
2. 在 crate 和 module 等级的代码上添加完整的使用示例。
3. 用好 module 组织好文档，不要把所有的代码都放在一起。
4. 尽可能丰富你的文档。



### 3.2 Type System Guidance

类型系统是一个很棒的工具，可以使你的接口 obvious、self-documenting、misuse-resistant。

首先一个是**semantic typing**，也就是添加表征值含意的类型，而不是仅仅使用基础类型。一个典型的例子是 Boolean，如果函数接受多个 bool 参数，那么很容易用错，此时就可以定义多个 enum 类型。

再一个是使用 zero-sized types 指示类型类型满足特定条件。举个例子：

```rust
struct Grounded;
struct Launched;
// and so on
struct Rocket<Stage = Grounded> {
    stage: std::marker::PhantomData<Stage>,
}

impl Default for Rocket<Grounded> {}
impl Rocket<Grounded> {
    pub fn launch(self) -> Rocket<Launched> { }
}
impl Rocket<Launched> {
    pub fn accelerate(&mut self) { }
    pub fn decelerate(&mut slef) { }
}

impl<Stage> Rocket<Stage> {
    pub fn color(&self) -> Color { }
    pub fn weight(&self) -> Killograms { }
}
```

还有一个小技巧是 `#[must_use]`注解，典型的使用场景是 Result。



## 4. Constrained

当你发布一个库以后，它就失去控制了，用户的代码逻辑既可能基于你想提供的特性，也可能基于你当初每考虑到的 bug，所以当你要做出用户有感知的修改时一定得仔细考虑。当你你可以不断的升级主版本，表示这是个不兼容的修改，但是过于频繁势必让用户感到不爽。

所以在有些方面需要与 flexible 取得平衡，有时候需要作出取舍。



### 4.1 Type Modifications

移除或者重命名公开的类型几乎都会破坏用户的代码，这里需要用好 Rust 的可见性描述符，例如 `pub(crate)`和`pub(in path)`，你暴露的公开类型越少，你修改代码的自由度越高。

除了类型的命名，类型中的字段修改也可能破坏向后兼容，举个例子：

```rust
// in your interface
pub struct Unit;
// in user code
let u = lib::Unit;
```

如果添加一个字段，constructor 就不匹配了。

```rust
// in your interface
pub struct Unit { pub field: bool };
// in user code
fn is_true(u: lib::Unit) -> bool {
    matches!(u, Unit { field: true })
}
```

还有一个是 match 的场景，Rust 默认是 exhaustive pattern match，如果存在没有覆盖的模式那么会在编译期报错。

Rust 提供了一个 `#[non_exhaustive]` 属性解决这个问题，添加这个属性后编译器将不允许使用 implicit constructor，以及 non-exhaustive pattern match。当然如果你觉得类型已经稳定了，以后肯定不会再添加字段，那么不需要加这个属性。



### 4.2 Trait Implementations

Rust 的 coherence rule 已经能限制很多异常场景，但是也有些特殊场景，例如为一个存在的类型实现任意的 trait 都需要特别小心，举个例子：

```rust
//crate1 1.0
pub struct Unit;
pub trait Foo1 { fn foo(&self) }
// not that Foo1 is not implemented for Unit

//crate2; depends on crate1 1.0
use crate1::{Unit, Foo1};
trait Foo2 { fn foo(&self) }
impl Foo2 for Unit { .. }
fn main() {
    Unit.foo();
}
```

如果在 crate1 中增加 `impl Foo1 for Unit`，而 crate2 升级到新版本那么就会报错。

```rust
pub trait CanUseCannotImplement: sealed::Sealed { .. }
mod sealed {
    pub trait Sealed {}
    impl<T> Sealed for T where T: TraitBounds {}
}
impl<T> CanUseCannotImplement for T where T: TraitBounds {}
```

Rust 还提供了一种 *sealed trait*，这种 trait 只能被外部使用，不能被实现。



### 4.3 Hidden Contracts

上面提到写一个接口就是定一个契约，除了上面提到的，其实还有两个微妙的细节：re-exports 和 auto-traits。



#### 4.3.1 Re-Exports

如果你的接口暴露了其他的外部类型，那么一旦你更新了外部类性的版本，很可能就破坏了用户代码，举个例子：

```rust
// your crate: bestiter
pub fn iter<T>() -> itercrate::Empty<T> { .. }

//their crate
struct EmptyIterator { it: itercrate::Empty<()> }
EmptyIterator { it: bestiter::iter{} }
```

如果从 itercrate 1.0 升级到 2.0，那么必然会有问题。

解决这个问题的方法也很简单，使用 newtype pattern 包装外部类型，或者使用 impl Trait 提供一个最小化的契约。



#### 4.3.2 Auto-Traits

Rust 有很多自动实现的 trait，例如 `Send` 和 `Sync`。

举个例子，你有一个公开的类型 B，其中包含一个类型 A，如果你改变了A使其不支持 `Send`，那么 A 也就自动不支持 `Send` 了，这可能会破坏用户代码。

很难追踪和发现这类问题，所以最好通过单元测试检测这种异常情况：

```rust
fn is_normal<T: Sized + Send + Sync + Unpin>() {}
#[test]
fn normal_types() {
    is_normal::<MyType>();
}
```



## 扩展阅读

1. Rust API Guidelines  https://rust-lang.github.io/api-guidelines/
2. Rust RFC 1105  https://rust-lang.github.io/rfcs/1105-api-evolution.html
3. *The Cargo book* on SemVer compatibility  https://doc.rust-lang.org/cargo/reference/semver.html