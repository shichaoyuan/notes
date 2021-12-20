
# 0.

对于一个常年写 Java 的程序员来说，Rust 最难入门的概念的应该是 ownership。

在 Java 中不需要关注对象的释放，内存的回收，而谈到这些概念想到的估计都是 C 语言的 malloc 和 free，Rust 特殊的地方在于自己搞了一套管理系统 —— ownership，刚开始学的时候会比较难理解。

# 1. 容易理解的部分

## 1.1. Copy 语义与 Move 语义

直观理解何谓所有权（ownership）呢？就是让每个值有明确的权属界限。

抽象概念比较模糊，先来两个例子：

```
fn main() {
    let x = 42;
    let y = x;
    println!("{}", x);
    println!("{}", y);
}
//42
//42
```
```
fn main() {
    let x = Box::new(42);
    let y = x;
    println!("{}", x);
    println!("{}", y);
}
//error[E0382]: borrow of moved value: `x`
// --> src/main.rs:4:20
//  |
//2 |     let x = Box::new(42);
//  |         - move occurs because `x` has type `std::boxed::Box<i32>`, which does not implement the `Copy` trait
//3 |     let y = x;
//  |             - value moved here
//4 |     println!("{}", x);
//  |                    ^ value borrowed here after move
```

可以看出在第一个例子中，虽然 `x` 和 `y` 的值在字面上都是 42，但是是两个不同的值，分别属于两个不同的变量。也就是说在 `let y = x;` 这一步进行了复制，这就是 Rust 中的 **Copy 语义**，只要类型实现了 **Copy trait** 就会进行复制。

而在第二个例子中，根据报错信息可见在 `let y = x;` 这一步，值 `Box::new(42)` 由 `x` 转移给了 `y`，那么 `x` 就不应该再使用了，这就是 Rust 中的 **Move 语义**，智能指针 Box 类型默认就会进行 Move。

## 1.2. 变量的生命周期

理解了 Copy 和 Move 语义之后，再来捋一下变量的生命周期。

首先是继承 C/C++ 的花括号词法作用域：
```
fn main() {
    let outer_val = 1;
    let outer_sp = "hello".to_string();
    {
        let inner_val = 2;
        outer_val;
        outer_sp;
    }
    println!("{:?}", outer_val);
    // error[E0425]: cannot find value `inner_val` in this scope
    // println!("{:?}", inner_val);
    // error[E0382]: borrow of moved value: `outer_sp`
    println!("{:?}", outer_sp);
}
```

`inner_val` 的生命周期就在花括号内，所以在外部就会获取不到。`outer_val` 是 Copy 语义，所以在花括号内访问只是复制一下，并没有所有权转移。

`outer_sp` 就比较微妙了，在花括号内访问会触发 Move 语义，所以在花括号结束后就会析构 `outer_sp` 转移前绑定的值。

除了独立的花括号之外，match 匹配、流程控制、函数和闭包等所有用到花括号的地方都会产生词法作用域。

# 2. 难理解的部分

## 2.1. 所有权借用

以上的复制和转移限制太大，很多功能无法实现，比如说写一个函数修改数组的第一个元素。按照 Move 语义，所有权转移给函数内，然后在函数结束时析构，传进来的数组就没有了。

怎么办？这就需要所有权借用了。

还是先看例子：
```
fn foo(v : &mut [i32; 3]) {
    v[0] = 3;
}

fn main() {
    let mut v = [1, 2, 3];
    foo(&mut v);
    assert_eq!([3, 2, 3], v);
}
```

注意看函数参数签名 `&mut [i32; 3]`，通过 & 符号就可以实现所有权的借用。

相对于“转移”，“借用”是一种更灵活的策略，灵活就会有隐患，所以为了保证内存安全，“借用”需要遵循以下三个规则：
1. 借用的生命周期不能长于出借方的生命周期。
2. 可变借用不能有别名，因为可变借用具有独占性。
3. 不可变借用不能再次出借为可变借用。

这三个规则的意义比较难掌握，暂时就记住了。

## 2.2. 生命周期参数

引入借用这个概念之后，基于词法的生命周期就不够用了，所以就引入了生命周期参数。

还是先看例子：

```
fn the_longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}

fn main() {
    let s1 = String::from("Rust");
    let s1_r = &s1;
    {
        let s2 = String::from("C");
        let res = the_longest(s1_r, &s2);
        println!("{} is the longest", res);
    }
}
```

生命周期参数的作用是帮忙借用检查器验证非法借用。函数间传入和返回的借用必须相关联，并且返回的借用生命周期不能比出借方的生命周期长。

另外在结构体和方法定义上也可以加生命周期参数。

在某些情况下，生命周期参数也是可以省略的，因为编译器可以自动补齐。

*以笔者当前的理解水平，基本上都是不主动写生命周期参数，等到编译器报错再回来写。*

## 2.3 智能指针

智能指针还有一些比较特殊的操作。

其中 Box<T> 支持“解引用移动”，来个例子：

```
fn main() {
    let a = Box::new("hello");
    let b = Box::new("Rust".to_string());
    let c = *a;
    let d = *b;
    println!("{:?}", a);
    //error[E0382]: borrow of moved value: `b`
    println!("{:?}", b);
}
```

这里的 `*a` 和 `*b` 操作相当于 `*(a.deref)` 和 `*(b.deref)` 操作。对于 Box<T> 类型来说，如果包含的类型 T 属于复制语义，则执行按位复制；如果属于移动语义，则移动所有权。

`*b` 这种解引用移动只有 Box<T> 支持，Rc<T> 或 Arc<T> 不支持。

接下来分析一个稍复杂的例子，为链表实现枚举：

```rust
pub struct List<T> {
    head: Link<T>,
}

struct Node<T> {
    elem: T,
    next: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// '_ 是一种显式省略生命周期参数的写法，
// 等价于 fn iter<'a>(&'a self) -> Iter<'a, T>
impl <T> List<T> {
    //List 不可变借用给 iter 方法
    pub fn iter(&self) -> Iter<'_, T> {
        // Option 的 as_ref() 方法将 &Option<Box<Node<T>>> 转为 Option<&Box<Node<T>>>，
        // &**node 操作的含义就是 &*(Box<Node<T>>.deref)，
        // 解引用移动，最后借用了 Node<T> 的所有权。
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    // Iter 可变借用给 next 方法
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            // 修改 next 指向下一个 Node
            self.next = node.next.as_ref().map(|node| &**node);
            // 返回从当前 Node 借用的 T
            &node.elem 
        })
    }
}
```

---
学习资料：
1. 《Rust 编程之道》 张汉东
2. [《Learn Rust With Entirely Too Many Linked Lists》](https://rust-unofficial.github.io/too-many-lists/index.html)

