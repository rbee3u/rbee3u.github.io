---
layout: post
title: 聊聊一看就会一写就跪的二分查找
tags: [技术, 算法]
---

> 图书馆自习的时候，一个女生背着一堆书走进阅览室，结果警报响了。阿姨让女生看是哪本书把警报弄响了，女生把书倒出来，一本一本地测。阿姨见状急了，把书分成两份，第一份过了一下，响了。又把这一份分成两份接着测，三回就找到了。阿姨用鄙视的眼神看着女生，仿佛在说 `O(n)` 和 `O(log n)` 都分不清。

<!--more-->

要说哪个算法的知名度较高，二分查找一定排得上号。以至于在面试的时候，如果应聘者简历写了擅长算法或者参加过ACM竞赛之类，我都会让他现场写一道二分查找的题目：

```golang
// 已知array数组的元素为单调非递减，给定一个target，返回array数组里面
// 第一个大于或等于target的元素下标，如果没有合法结果则返回len(array)
func FirstGreaterOrEqual(array []int, target int) int {
	// TODO: write your code here
	return len(array)
}
```

一开始我觉得如果对方真的擅长算法，这题应该很基础并不算为难应聘者(纸上写只需要伪代码即可/电脑上写则需要通过编译)，直到几十轮面试之后几乎90%的人都在这道题上败北。

后来我认真反思了这个事情，发现从思路上来讲，绝大部分人都是没有问题的：用`l`和`r`来表示一个待查找区间，将区间中点`m`对应的元素同`target`相比较，根据比较结果来决定继续查找哪一半。也就是说，大家几乎都能写出如下的“骨架”：

```golang
func FirstGreaterOrEqual(array []int, target int) int {
	// 初始化区间左端点： -1  ||  0  ||  1  ？
	l := 0
	// 初始化区间右端点： len(array) - 1  ||  len(array)  ||  len(array) + 1  ?
	r := len(array)
	// 当区间不为空时循环： l + 1 < r  ||  l < r  ||  l <= r  ||  l <= r + 1  ?
	for l < r {
		// 计算区间中点： l + (r - l) / 2  ||  l + (r - l + 1) / 2  ?
		m := l + (r - l) / 2
		// 将中点对应的元素同target比较： >  ||  >=  ||  <  || <=  ?
		if array[m] < target {
			// 继续查找右侧这一半： m - 1  ||  m  ||  m + 1  ?
			l = m + 1
		} else {
			// 继续查找左侧这一半： m - 1  ||  m  ||  m + 1  ?
			r = m
		}
	}
	// 这里应该是 l - 1  ||  l  ||  l + 1  ?
	// 这里应该是 r - 1  ||  r  ||  r + 1  ?
	return l
}
```

但是你可能已经注意到了我加的这些注释，几乎每一行代码都有很多的“抉择”，这使得许多应聘者最终“栽”在了这短短十来行代码上。后来我又在[维基百科](https://en.wikipedia.org/wiki/Binary_search_algorithm)上发现了这样一段话：

> 尽管二分查找的基本思想相对简单，但细节可以令人难以招架 —— 高德纳

> 当乔恩·本特利将二分查找问题布置给专业编程课的学生时，90%的学生在花费数小时后还是无法给出正确的解答，主要因为这些错误程序在面对边界值的时候无法运行，或返回错误结果。1988年开展的一项研究显示，20本教科书里只有5本正确实现了二分查找。不仅如此，本特利自己1986年出版的《编程珠玑》一书中的二分查找算法存在整数溢出的问题，二十多年来无人发现。Java语言的库所实现的二分查找算法中同样的溢出问题存在了九年多才被修复。

所以我写下了这篇文章，希望能够彻彻底底地讲明白我心目中的“二分查找”到底是什么。在弄懂它的本质以后不管遇到什么类型的二分查找问题，都能够准确无误地写出正确的代码。

### 问题重新定义

我们要解决的第一个问题，就是重新定义“二分查找”。为什么要重新定义，因为它的变种问题实在是太多了：在`严格单调递增`/`非严格单调递增`/`严格单调递减`/`非严格单调递减`的数组中查找`大于`/`大于等于`/`小于`
/`小于等于``target`的`最前一个`/`最后一个`的元素下标。其实这还只是基本变种，一些更为复杂的变种就不在这里一一列举了。如果我们针对特定问题去设计特定算法，那你必然会迷失在茫茫多的变种问题里面。

但是这些问题的解决其实都可以依赖于一个基本问题的解决，可以不严格地陈述为：

> 在一个左边全是 false 右边全是 true 的数组中，有且只有一个从 false 突变为 true 的点，二分查找其实就是要找到这个突变点。

![target](/assets/img/20210108-target.png)

我认为比起严格的陈述，每当说到二分查找，脑海中自然而然地浮现出上面这幅图更为重要。在这幅图所表示的数组中，红色的格子表示`false`，蓝色的格子表示`true`。其中格子为深红色和深蓝色表示颜色已经确定，它们是向两侧无限延伸没有尽头的。整个数组只有中间浅色的一段还未确定，二分查找其实就是要确定出红色到蓝色的突变点(
这里不用着急解决各种变种问题，后面会讲到如何将它们转化为这个基本问题)。

### 区间如何表示

问题看起来像是定义清楚了，其实并没有。阻碍我们的第一个绊脚石就是：待查找区间应该如何表示？当我们说使用`l`和`r`来表示这个区间的时候，其实有四种可能

- 前开后开`(l, r)`
- 前开后闭`(l, r]`
- 前闭后闭`[l, r]`
- 前闭后开`[l, r)`

到底应该使用哪种来表示其实并没有关系，理论上使用任何一种都行，只要做到前后一致即可。例如你使用了前闭后闭`[l, r]`，判断区间非空就应该写成`r - l + 1 > 0`，相反如果你使用了前闭后开`[l, r)`，判断区间非空则应该写成`r - l > 0`
。就个人喜好而言，我更偏向最后一种：前闭后开`[l, r)`。

首先说说前开有什么缺点。例如一个数组`array`，我们想要表示`array[0], array[1], array[2]`这段区间，用前开就意味着`l = -1`
。对于Golang这种语言来说也许算不上什么问题，但是如果遇到数组下标严格要求为无符号整型的语言，你可能将会面临无穷无尽的类型转换。相反，选择前闭则不存在这个问题。

再来说说后闭有什么缺点。严格来说后闭其实没什么缺点，只不过当我们计算区间长度的时候，用前闭后闭是`d = r - l + 1`，用前闭后开则是`d = r - l`。或者从另一个角度来看，当我们固定`l = 0`
时，如果想要表示空区间，前闭后闭的情况下`r = -1`，依然会面临着无符号整型问题。

因此我偏向使用前闭后开`[l, r)`。这种情况下，用`l < r`判断区间非空，用`r - l`计算区间长度，用`m = l + (r - l) / 2`计算区间中点。再次强调，这仅仅代表了个人偏好。

### 突变点如何表示

![target](/assets/img/20210108-target.png)

区间的问题我们算是解决(或者说达成共识)了，下一个问题是突变点如何表示。从之前的图可以看得出来，我们可以有两种选择

- 用最后一个红色(`F`)表示突变点。
- 用最前一个蓝色(`T`)表示突变点。

同样的，这两种选择谁也没有错，但选最后一个红色作为突变点依然会面临着`F = -1`的无符号整型问题，因此我更倾向于选择最前一个蓝色表示突变点。而且由于`F`和`T`之间有着明显的关系`F + 1 = T`，所以真正需要`F`
的时候我们也可以通过`T`将它轻易算出来。

### 算法核心逻辑

当我们把`问题重新定义`、`区间如何表示`、`突变点如何表示`解决以后，整个算法其实已经呼之欲出了。在这之前我们再来严格地陈述下这个问题：

1. $f(x)$ 是一个从整型到布尔型的映射，定义域 $x \in (-\infty, +\infty)$，值域 $f(x) \in \left\\{ false, true \right\\}$
2. 对于任意的 $x_1 \leq x_2$，都有 $f(x_1) \leq (x_2)$，即非严格单调性(这里我们约定 $false \lt true$)
3. 给定一个待查找区间 $[ l, r )$，其中 $0 \leq l \leq r$，且已知 $f(l - 1) = false$ 和 $f(r) = true$
4. 查找出 $\min\left\\{x \mid x \in [l, r], f(x) = true\right\\}$，即最小的使得 $f(x) = true$ 的 $x$

这个陈述其实无关紧要，脑袋里装下这幅图，并且理解它，你一定会自己"定义"出这个问题以及"发现"对应的算法实现。

![core](/assets/img/20210108-core.png)

一图胜千言，我们直接把这幅图“翻译”成代码。尽管只有短短几行，请务必对照上图仔细体会。

```golang
func BSearch(l, r int, f func(int) bool) int {
	for l < r {
	    // 注意这里是为了避免溢出
		m := l + (r - l) / 2
		if !f(m) {
			l = m + 1
		} else {
			r = m
		}
	}
	return l
}
```

### 正确性和复杂度

关于正确性和复杂度分析，在这里说"显然"其实一点都不为过。不过我还是用数学归纳法来证明一下正确性，至于复杂度分析就留给读者了。

当`d = r - l = 0`时，代码不会进入`for`循环，直接返回`l`。由问题严格陈述的第3点我们知道`f(l - 1) = false`和`f(r) = true`，又因为此时`l = r`，所以直接返回`l`就是正确的答案。

假设对于所有的`d = 0, 1, 2, ..., k - 1`，该算法都能返回正确的答案`l`。那么当`d = k`时，代码将会进入`for`循环，此时中点`m ∈ [l, r)`。这时候分两种情况：
- 如果`f(m) = false`，我们令`l = m + 1`，此时依然满足问题严格陈述的第3点，但是区间长度`d`变小了。 
- 如果`f(m) = true`，我们令`r = m`，此时同样满足问题严格陈述的第3点，但是区间长度`d`也变小了。

综上，在这两种情况下，我们都可以在满足问题严格陈述的第3点前提下，将区间长度`d`变小，转化为一个已经被证明可解决的问题。

### 解决变种问题

回到最开始的面试问题

```golang
// 已知array数组的元素为单调非递减，给定一个target，返回array数组里面
// 第一个大于或等于target的元素下标，如果没有合法结果则返回len(array)
func FirstGreaterOrEqual(array []int, target int) int {
	// TODO: write your code here
	return len(array)
}
```

这再简单不过了吧，我们只需要将`f(x)`定义为`array[x] >= target`是不是就可以了，因此它的实现代码可以是这样

```golang
func FirstGreaterOrEqual(array []int, target int) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] >= target
	})
}
```

至于其他变种问题呢，如法炮制，核心就是想办法把问题转化成基本问题中的布尔数组。掌握了这一点，就可以说是掌握了二分查找的精髓。

```golang
// 问题: array 非严格递增，返回第一个大于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 len(array) 即可。
func FirstGtInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] > target
	})
}

// 问题: array 非严格递增，返回第一个大于等于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 len(array) 即可。
func FirstGteInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] >= target
	})
}

// 问题: array 非严格递增，返回最后一个小于等于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 -1 即可。
func LastLtInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] > target
	}) - 1
}

// 问题: array 非严格递增，返回最后一个小于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 -1 即可。
func LastLteInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] >= target
	}) - 1
}

// 问题: array 非严格递减，返回第一个小于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 len(array) 即可。
func FirstLtInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] < target
	})
}

// 问题: array 非严格递减，返回第一个小于等于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 len(array) 即可。
func FirstLteInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] <= target
	})
}

// 问题: array 非严格递减，返回最后一个大于等于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 -1 即可。
func LastGtInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] < target
	}) - 1
}

// 问题: array 非严格递减，返回最后一个大于 target 的元素下标。
// 补充: 如果查找不到合法结果，则返回 -1 即可。
func LastGteInAsc(array []string, target string) int {
	return BSearch(0, len(array), func(x int) bool {
		return array[x] <= target
	}) - 1
}
```

当然除了基本变种，还有很多其他类型的变种，未必都能完全套到这个基础问题上来。但是大差不差，做一些轻微的调整大多数都是能轻易解决的。如果实在遇到解决不了的问题，希望这篇文章中的某些点可以为你提供启发性的思路。

### 写在最后

我真的不是标题党，二分查找确确实实是个一看就会一写就跪的问题。包括几乎90%的面试者都在这道题上败北，也同样是真实数据。写这篇文章主要参考了`Golang`的[sort.Search](https://golang.org/src/sort/search.go)和`C++`的[std::lower_bound](https://en.cppreference.com/w/cpp/algorithm/lower_bound)，后者的实现考虑得更为全面一些，只需要迭代器是ForwardIterator就行了，不依赖随机寻址甚至不依赖向前寻址，相当优雅。
