---
layout: post
title: Y Combinator (续前一篇Lambda)
tags: [技术, 编程]
---

这篇文章是对前一篇[An Online Lambda Interpreter](/2017-09-07-lambda-qlcoder)的续写，搞清楚 `Y Combinator` 这个概念的来龙去脉。当然这两篇文章里的 lambda 演算语法定义实际上并不是标准的定义，具体的标准定义与实现我在[这个目录](https://github.com/rbee3u/lambda-erlang)做了单独实现。

<!--more-->

# Y combinator
## 问题的提出
如果让你用自己最熟练的编程语言写一个fibonacci(暂不考虑效率问题)，这一定不是问题，下面是一个Python的例子：
```python
def fib(n):
    if n < 2:
        return n
    else:
        return fib(n-1)+fib(n-2)
```
我们现在考虑把它翻译成ELambda的语法：
```lisp
\n.
    (((cond ((less n) 2))
    \_.
        n
    )
    \_.
        ((add
        (fib ((sub n) 1)))
        (fib ((sub n) 2)))
    )
```
当我们在考虑一个问题，如果不用关注其具体细节，可以把它们抽象一下，这样可以在一定程度上减轻思维负担。比如关于上面的整个函数体，本质上就是一个过程中包含了`(fib x)`这种形式的调用，因此我们可以把整个函数简记为：
```lisp
\n.<(fib x)>
```
现在问题一下就清晰起来了：`fib`是啥？在我们的理解中它应该就是这个函数自身，但是lambda演算中并没有绑定函数名这种说法。没关系我们可以通过传参的方式来绑定，这是lambda演算的一个常见技巧：
```lisp
(
\fib.\n.<(fib x)>
\fib.\n.<(fib x)>
)
```
你看之前不是说fib没有定义不能绑定自身吗，现在我把自己当做参数传给自己，这不就实现了绑定？问题好像得到解决了，不过等等，好像有点儿问题。`\fib.\n.<(fib x)>`这已经是一个俩参数的函数了，因此函数体里`<(fib x)>`这种调用方式很显然不合法。这个问题不大，想必你很快就找出了解决方案：
```lisp
(
\fib.\n.<((fib fib) x)>
\fib.\n.<((fib fib) x)>
)
```
OK，问题解决。但是，作为强迫症的你会发现，这样一来`fib`根本就不是真正的递归函数啊，相反`(fib fib)`才是。

![你根本不是司机](/assets/img/20191120-old-driver.jpg)

面对这个问题，有有两派分别采取了不同的方案，最终都殊途同归，形成了完美的递归方案。


## 自顶向下
[参考自Belleve的回答](http://zhihu.com/question/21099081/answer/23893046)

回过头来看前一个方案：
```lisp
(
\fib.\n.<(fib x)>
\fib.\n.<(fib x)>
)
```
我们是想把`\fib.\n.<(fib x)>`传给`\fib.\n.<(fib x)>`，天真地以为这样可以形成递归，怎奈所托非人，最后不得不委曲求全。那么我们换一种思维，如果一开始传给`\fib.\n.<(fib x)>`的如果不是`\fib.\n.<(fib x)>`，而是一个真正的fibonacci函数(`driver_fibonacci`)呢？
```lisp
(
\fib.\n.<(fib x)>
driver_fibonacci
)
```
什么意思？我们把一个`driver_fibonacci`传给`\fib.\n.<(fib x)>`来构成`driver_fibonacci`，也就是：
```lisp
driver_fibonacci = (
    \fib.\n.<(fib x)>
    driver_fibonacci
)
```
这不是有病吗，我们要寻找的就是`driver_fibonacci`，我要知道它是啥才能传进去。如果我都已经知道了它是啥，问题不早就解决了吗。不不不，我不是指已经找到，而是说如果有这么一个`driver_fibonacci`的话，它应该满足这样一个等式：
```lisp
fixed_point = (g fixed_point)
```
仔细观察这个形式，我们梦寐以求的`driver_fibonacci`原来不过是`\fib.\n.<(fib x)>`的一个不动点而已。

那么不动点怎么求，这里就是大名鼎鼎的Y combinator了：
```lisp
Y = \f.(\s.(f (s s)) \s.(f (s s)))
```
这里做个简要证明，很多地方都有：
```lisp
(Y g)
    = (\f.(\s.(f (s s)) \s.(f (s s))) g)
    = (\s.(g (s s)) \s.(g (s s)))
    = (g (\s.(g (s s)) \s.(g (s s))))
    = (g (Y g))
```
你会发现原来`(Y g)`正好就构成了`g`的一个不动点！

因此我们通常看到的带有名称的支持递归的函数定义：
```lisp
def fib: \x.<(fib x)>
```
从lambda演算的角度来看，他和：
```lisp
(Y \fib.\x.<(fib x)>)
```
没有本质上的区别，一个完美的递归函数就这么构成了。

当然现在的你心里可能更多地会想：fuck，这 Y 怎么来的！


## 自底向上
[参考自王垠的PPT](http://www.slideshare.net/yinwang0/reinventing-the-ycombinator)

好好好，接下来我就要讲，其实你也可以发现Y combinator。还记得我们之前那个“你根本不是司机”的fibonacci函数吗？
```lisp
(
\fib.\n.<((fib fib) x)>
\fib.\n.<((fib fib) x)>
)
```
由于这里面的`fib`根本不是真`fib`，我们决定给它随便改个名字：
```lisp
(
\s.\n.<((s s) x)>
\s.\n.<((s s) x)>
)
```
我们首先要解决的是`(s s)`，这一坨直接决定了它是不是老司机函数，用`fib`替换它得到：
```lisp
(
\s.(\fib.\n.<(fib x)> (s s))
\s.(\fib.\n.<(fib x)> (s s))
)
```
好现在我们发现这里面有一个很nice的结构`\fib.\n.<(fib x)>`，也考虑把它提取出来用`f`替换：
```lisp
(
\f.(\s.(f (s s)) \s.(f (s s)))
\fib.\n.<(fib x)>
)
```
看到没，只需要两步，Y combinator就已经出来了，可见它并不是一个什么神秘的不可捉摸的东西。


## 写在最后
最后有一点需要注意的是，在call-by-value的调用过程中，上面这个Y combinator会造成死循环：
```lisp
(
\f.(\s.(f (s s)) \s.(f (s s)))
\fib.\n.<(fib x)>
)
=
(
\s.(\fib.\n.<(fib x)> (s s))
\s.(\fib.\n.<(fib x)> (s s))
)
=
(
\fib.\n.<(fib x)>
    (
    \s.(\fib.\n.<(fib x)> (s s))
    \s.(\fib.\n.<(fib x)> (s s))
    )
)
=
(
\fib.\n.<(fib x)>
    (
    \fib.\n.<(fib x)>
        (
        \s.(\fib.\n.<(fib x)> (s s))
        \s.(\fib.\n.<(fib x)> (s s))
        )
    )
)
(
\fib.\n.<(fib x)>
    (
    \fib.\n.<(fib x)>
        ......
            (
            \s.(\fib.\n.<(fib x)> (s s))
            \s.(\fib.\n.<(fib x)> (s s))
            )
    )
)
```
这个问题在call-by-name的调用过程中是不会出现的，要解决它你需要对Y combinator中的`(s s)`做一次`eta-expansion`。


## 最后的最后
你看，括号完全不是问题嘛！！！

