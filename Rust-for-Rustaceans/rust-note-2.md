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





## 3. Obvious



## 4. Constrained









## 扩展阅读

1. Rust API Guidelines  https://rust-lang.github.io/api-guidelines/
2. Rust RFC 1105  https://rust-lang.github.io/rfcs/1105-api-evolution.html
3. *The Cargo book* on SemVer compatibility  https://doc.rust-lang.org/cargo/reference/semver.html
