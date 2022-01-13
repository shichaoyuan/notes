



## 0.



最近给 Netty 提了个 [issue](https://github.com/netty/netty/issues/11963)，了解到一个字符串查找算法 —— [Two-Way String-Matching](http://monge.univ-mlv.fr/~mac/Articles-PDF/CP-1991-jacm.pdf)，学习了一下，简单做个笔记。



## 1. 

在摘要中作者将 Two-Way 称之为 KMP 与 BM 之间的“中间体”，笔者感觉关联不大，实际上这个“对比”主要体现在扫描方向上，KMP 是从左向右扫描模式字符串，BM 是从右向左扫描模式字符串，而 Two-Way 既有从左向右扫描又有从右向左扫描。

理解 Two-Way 的关键不在于理解 KMP 和 BM，而是理解 *Critical Factorization Theorem*。



## 2. Critical Factorization Theorem

这里用 *x* 表示一个字符串。

### 2.1 *period* 是什么？

如果字符串 *x* 中所有 *x[i] = x[i+p]*，那么 *p* 就是 *x* 的一个 *period*，在这里 *p* 是一个严格的正整数。

直观理解 *period* 就是字符串中重复模式的周期，举几个例子：

- "abaabaa" 的 *period* 可以是 3 或者 6；
- "abcdefg" 的 period 可以是 7。

### 2.2 *the period* 是什么？

字符串 *x* 最小的 *period* 称之为 *the period*，表示为 *p(x)*，满足不等式  *0 < p(x) <= |x|*。

上面例子中，"abaabaa" 的 *p(x)* = 3。

### 2.3 *local period* 是什么？

既然是 *local* 的那肯定得先设定个位置 *l* （*0 <= l <= |x|*，字符串起始索引为 1），如果对于所有索引 *i*（*l-r+1 <= i <= l*）都满足 *x[i] = x[i+r]* 那么 *r* （*r >= 1*）称为 *x* 在位置 *l* 的一个 *local period*。

字符串 *x* 在位置 *l* 的 *the local period* 也就是在 位置 *l* 最小的 *local period* 表示为 *r(x, l)*，满足 *1 <= r(x, l) <= p(x)*。

上面例子，"abaabaa" 在位置 *l=3* 处的 *local period* 可以 1 或者 3，那么 *r(x, l) = 1*。

### 2.4 Critical Factorization Theorem

> *THEOREM (CRITICAL FACTORIZATION THEOREM).*
>
> *For each word x, there exists at least one position l with 0 <= l < p(x) such that*
>                      *r(x, l) = p(x).*
> A position *l* such that *r( x, l) = p(x)* is called a *critical position* of *x*. 



上面的例子，"abaabaa" 的 *p(x)* = 3，有三个 critical position，也就是 2、4、5.

```
ab|aabaa
abaa|baa
abaab|aa
```

存在一个 *0 <= l < p(x)* 的位置，也就是 2。



## 3. 初识 Two-Way



Two-Way 算法的整体流程就是以 pattern 字符串的 critical position 分界，右侧（不包含 critical position）从左至右扫描匹配，左侧（包含 critical position）从右至左扫描匹配。如果扫描到不匹配的字符，那么就向右移动 pattern 字符串，再按照上述逻辑扫描匹配。



![](./assets/two-way-1.png)



假设已知 pattern 字符串的 the period (*p*) 和 critical position (*l*)，并且 *l < p*，那么可以这样描述 Two-Way 算法。



![](./assets/two-way-2.png)



变量 `pos` 记录 text 中每一轮 pattern 匹配的起始位置，第 3 行是从左向右扫描的逻辑，变量 `i` 记录字符匹配的位置，第 6 行是从右向左扫描的逻辑，变量 `j` 记录字符匹配的位置。

在这里多了一个变量 `s`，记录的是从当前 `pos` 开始，text 与 pattern 相同的字符长度。为什么在扫描之前可以预先知道相同的前缀呢？这个场景出现在上一轮扫描 critical positon 右侧全部匹配的情况。

这里还比较好理解，先看个例子：

![](./assets/two-way-3.png)

直观理解一下，在第一轮结束时，就已知了当前 text 与 pattern 的 critical position 右侧完全一致，将 pattern 向右移动 `p`，也就是使索引加`p`，根据 *the period* 的性质，也就是把 pattern 的左侧子串与 text 中已知的 pattern 右侧子串根据 critical position 对齐，所以肯定是相同的。

形式化的证明为：

> Consider a step of the loop that ends with the statement at line 9. Let q<sub>k+1</sub> be the value of *pos* at the end of the step. In this situation, the loop at line 3 must false, which means that *"i <= |x|"* false, which means that
>
> ​           *x[i'] = t[q<sub>k</sub>+i'],    l < i' <= |x|.*
>
> Since *q<sub>k+1</sub> = q<sub>k</sub> + p* and *l < p* we get
>
> ​          *x[i'] = t[q<sub>k+1</sub> + i'],     0 < i' <= |x|-p.*
>
> The property is then true at the end of the step, because its last statement at line 9 gives the value *|x|-p* to *s*.











