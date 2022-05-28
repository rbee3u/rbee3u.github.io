---
layout: post
title: 记一道有趣的CodeReview题目
tags: [技术, 编程]
---


![问题代码](/assets/img/20171216-146228364162795.jpg)

这道题目是在[千里码](http://www.qlcoder.com/task/7692)(不知道这野鸡网站现在还在不在)，大意就是这段代码会因为某个特殊的随机种子而抛出异常，需要你找到这个随机种子。


## 如果只是想要过题
这也是我写这篇学习资料的初衷，如果只是想要过题的话这再简单不过了。根据题目的描述，是由于`seed`变量取了某个特殊值导致了异常。那么最直接的方法：遍历`seed`的所有可能取值不断`try..catch`，接着坐下来喝杯咖啡估计结果就出来了。我相信稍微有点编程基础的人都能够顺利解决，但是仅止步于此的话未免太可惜了。

<!--more-->

## 是哪里抛出的异常
好好好现在我们提高一下自身追求，去找找异常究竟是哪里抛出的。从`main`函数入口一步一步往下看：第一个可能抛异常的是 `new Gen()`，但立即可以发现这是不可能的，因为`Gen`并没有自己实现构造函数；那么第二个可能抛异常的就应该是`rand.srand(seed)`了，因此需要去检查`srand`函数，检查后发现这也是不可能的；排除了前两种可能，异常的抛出点只可能是在`arr[rand.next()%100]`这里了，因此可以确定很大可能是数组下标越界。

由于数组下标越界，其实我们完全可以很武断地认为`rand.next()%100`是个负值，然后就判定是`rand.next()`返回了负值。关于这道题这种判断确实是对的，但未必所有的语言都是这样其实，举一些例子（当然你还可以去尝试更多）：

```javascript
// JavaScript => -1
console.log((-11) % 5);
```

```lua
-- Lua => 4
print((-11) % 5)
```

```haskell
-- Haskell => 4
show $ mod (-11) 5
```

```python
# Python => 4
print((-11) % 5)
```

```erlang
% Erlang => -1
io:format("~w~n", [(-11) rem 5]).
```

```mysql
-- MySQL => -1
SELECT (-11) % 5;
```

```perl
# Perl => 4
print((-11) % 5)
```

```c++
// C++ => -1
#include <iostream>
int main() {
    std::cout << (-11) % 5 << std::endl;
    return 0;
}
```

```java
// Java => -1
public class Fuck{
    public static void main(String[] args) {
        System.out.println((-11) % 5);
    }
}
```

因此对于不同的编程语言而言，对负数取模可能有不同的实现方式，而这题因为是`Java`对负数取模恰好确实可能为负。

好了我们继续回来研究`rand.next()`，发现在第一次调用它时实际上是执行了一次`next_state()`然后返回`Math.abs(state[0])`，而这个返回的值(根据之前的推断)恰好就是一个引起数组下标越界的罪魁祸首。什么？`Math.abs`竟然会返回负数，不是求绝对值吗？理论上是这样的，但如果你了解整数是以补码方式存放在计算机里的话，不难知道有一个值`Math.abs`是不能hold住的--`INT_MIN`。`Math.abs(INT_MIN) == INT_MIN`，更准确地说是恰好`INT_MIN`不是`100`的倍数，所以才会导致异常的抛出。

## 如何寻找罪魁祸首
问题开始变得清晰许多了，原来是因为第一次执行完`next_state()`以后`state[0]`的值变成了`0x80000000`，那么接下来显然是应该分析`next_state()`。这个函数虽然看起来挺复杂，但由于我们只关注`state[0]`本身，所以好多东西是可以抛开不管的。比如说当执行第一个循环的第一次时，`state[0]=state[397]^twist(state[0],state[1])^2074608327;` 这个是有用的。但是这个执行以后，由于`p`一直在增加，后面的所有操作并不会影响到`state[0]`，所以都可以抛开不管了。

到目前为止似乎还和题目中说的`seed`没有半毛钱联系，但马上就有了：因为`state[0],state[1],state[397]`的初值就是由`seed`确定的。我们进一步把原问题简化：
```
(398*seed) ^ twist(seed, 2*seed) ^ 2074608327  ≡  0x80000000
==>      (398*seed) ^ twist(seed, 2*seed) ^ 0xfba802c7  ≡  0
```
现在需要做的就是找出符合条件的`seed`了，有了这个判断条件，一个循环就可以搞定。

## 强迫症晚期的做法
虽然上面的方法对于解出这道题而言已经是秒出结果了，但如果你是强迫症晚期，肯定不会满足于一个原本优美的等式里竟然还夹杂着个`twist`函数。因此我们考虑把`seed`代换进去试试，看看究竟能搞出个什么幺蛾子。
```
twist(seed, 2*seed)
= (((seed & 0x80000000) | (2*seed & 0x7fffffff)) >> 1) ^ ((2*seed & 1) == 1 ? 0x9908b0df : 0)
= (((seed & 0x80000000) | (2*seed & 0x7fffffff)) >> 1)
= (((seed >> 1) & 0x40000000) | (seed & 0x3fffffff))
```
这个结果就很显然了，其实`twist`最后返回的结果就是把`seed`的最高位右移到次高位，最高位用零填充其余位不变。我们不妨也把这个函数叫做`twist`，亦即：
```c++
int32_t twist(int32_t seed) {
    return (((seed >> 1) & 0x40000000) | (seed & 0x3fffffff));
}
```
因此我们最终要解决的问题变成了：
```
(398*seed) ^ 0xfba802c7  ≡  twist(seed)
```
事情看起来要简单些了，但事实上还是没有得到有效解决，`twist`的阴影始终笼罩在强迫症晚期患者头顶。注意到`twist`的返回值除了最高两位可能不同外其它位数是一样的，所以如果我们先在`mod 2**30`系统下考虑这个问题（原问题是`mod 2**32`），应该就会简化许多：
```
(398*seed) ^ 0xfba802c7  ≡  seed   (mod 2**30)
```
关于这个问题，可以参考我之前在 [StackOverflow](http://stackoverflow.com/questions/37019612/how-to-find-the-fixed-point-of-a-simple-mod-function-elegantly) 的提问。答案的思路大致是：
> 对高位的运算结果并不会影响到低位，因此可以从低位开始慢慢往高位推。

这个思路也同样适用于我们目前还在抛弃考虑的最高两位，因此在`mod 2**30`系统下先找到了合适的解以后再来枚举最高两位就可以得到我们的最终答案了。

如果还有时间精力深入去研究我猜测还会有一些有趣的东西，总之很好玩就是了。顺手贴个简单的`C++`参考代码，食汉三应该不会删吧：
```c++
#include <iostream>
#include <vector>
#include <cstdint>
using namespace std;

class Functor {
public:
     int32_t operator()(int32_t x) {
          return x*mulnum ^ xornum ^ twist(x);
     }
     Functor(int32_t mularg, int32_t xorarg)
          : mulnum(mularg), xornum(xorarg) {}
private:
     int32_t mulnum, xornum;
     int32_t twist(int32_t x) {
        return (((x >> 1) & 0x40000000)
                    | (x & 0x3fffffff));
    }
};

int main() {
    Functor f(398, 0xfba802c7);
    vector<int32_t> solve32(Functor &);
    vector<int32_t> solns = solve32(f);
    for (int32_t ans: solns) {
         cout << ans << endl;
    }
    return 0;
}

vector<int32_t> solve32(Functor &f) {
    vector<int32_t> solns, tmp;
    void solve30(Functor &, int32_t, int
            , vector<int32_t> &solns);
    solve30(f, 0, 0, solns);
    solns.swap(tmp); int32_t y;
    for (auto x : tmp) {
        y = (x & 0x3fffffff) | 0x00000000;
        if (!f(y)) solns.push_back(y);
        y = (x & 0x3fffffff) | 0x40000000;
        if (!f(y)) solns.push_back(y);
        y = (x & 0x3fffffff) | 0x80000000;
        if (!f(y)) solns.push_back(y);
        y = (x & 0x3fffffff) & 0xc0000000;
        if (!f(y)) solns.push_back(y);
    }
    return solns;
}

void solve30(Functor &f, int32_t x, int i
            , vector<int32_t> &solns) {
    // if (false == solns.empty()) { return; }
        // you can determine when to return
        // if there are too many solutions
    if (i>=30) {solns.push_back(x);return;}
    int32_t mask = (1<<(i+1)) - 1;
    if ((f(x) & mask) == 0) {
        solve30(f, x, i + 1, solns);
    }
    x = x | (1 << i);
    if ((f(x) & mask) == 0) {
        solve30(f, x, i + 1, solns);
    }
}
```


## 写在最后的总结
> 古人之观于天地、山川、草木、虫鱼、鸟兽，往往有得，以其求思之深而无不在也。夫夷以近，则游者众；险以远，则至者少。而世之奇伟、瑰怪、非常之观，常在于险远，而人之所罕至焉，故非有志者不能至也。

有时候题目虽小，如果认真挖掘，里面的知识点其实也挺有趣的，收获不小。

