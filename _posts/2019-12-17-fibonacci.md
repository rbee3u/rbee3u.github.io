---
layout: post
title: 斐波那契数列第一千万项怎么求
tags: [技术, 算法]
---

众所周知，斐波那契数列的定义是：f(0) = 0, f(1) = 1, 且当 n > 1 时 f(n) = f(n-1) + f(n-2)。今天讨论个相对简单的问题：斐波那契数列第一千万项怎么求？

<!--more-->

## 暴力算法(V0)
最容易想到的是直接翻译递推式，二话不说撸出代码：

```golang
func fibV0(n int) Number {
	defer logElapsed("fibV0")()
	a, b := Num1, Num0
	for i := 0; i < n; i++ {
		a, b = NumAdd(a, b), a
	}
	return b
}
```

很可惜当输入为 n = 10000000 时这段代码在我的电脑上跑了足足有 15 分钟，风扇呼呼地转。

## 矩阵快速幂优化(V1)
因为从定义我们很容易得到：

$$ R = \begin{pmatrix} f_{n+1} & f_n \\ f_n & f_{n-1} \end{pmatrix} 
= \begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix} ^ n = T^n $$

问题转成了如何快速求出 R，这里对 n 分奇偶讨论：

$$ R = T^n = \begin{cases}
T^k \cdot T^k , & \text{if $n = 2k$} \\
T^k \cdot T^k \cdot T , & \text{if $n = 2k+1$} \\
\end{cases} $$

根据这一版的递推公式来实现的代码：

```golang
func fibV1Recursive(n int) Number {
	defer logElapsed("fibV1Recursive")()
	return fibV1Helper(n)[1][0]
}

func fibV1Helper(n int) Matrix {
	if n <= 0 {
		return Mat0
	}
	r := fibV1Helper(n / 2)
	if n % 2 != 0 {
		r = MatMul(MatMul(r, r), Mat1)
	} else {
		r = MatMul(r, r)
	}
	return r
}
```

当输入为 n = 10000000 时大约 2.435 秒出结果，当然我们也可以把递归改成迭代(速度没有明显差距)：

```golang
func fibV1Iterative(n int) Number {
	defer logElapsed("fibV1Iterative")()
	r := Mat0
	bs := strconv.FormatInt(int64(n), 2)
	bs = strings.TrimLeft(bs, "0")
	for _, b := range bs {
		if b != '0' {
			r = MatMul(MatMul(r, r), Mat1)
		} else {
			r = MatMul(r, r)
		}
	}
	return r[1][0]
}
```

## 矩阵对称性优化(V2)
之所以写这篇文章，主要就是因为网上大部分的资料都止步于矩阵快速幂，其实在这基础上还有可以秀微操的空间。我们再来仔细看看 R 的定义：

$$ R = \begin{pmatrix} f_{n+1} & f_n \\ f_n & f_{n-1} \end{pmatrix} = \begin{pmatrix} x & z \\ y & t \end{pmatrix} $$

你会发现之前的运算其实做了很多无用功：
- z 和 y 必然相等，z 没必要再计算一遍。
- t = x - y，因此 t 也没必要再计算一遍。

最后你会发现，原来只需要计算矩阵第一列的那两个元素即可：

$$
x_n = \begin{cases}
x_k \cdot x_k + y_k \cdot y_k , & \text{if $n = 2k$} \\
(x_k + y_k) \cdot x_k + x_k \cdot y_k , & \text{if $n = 2k+1$} \\
\end{cases}
\\
y_n = \begin{cases}
x_k \cdot y_k + y_k \cdot (x_k - y_k) , & \text{if $n = 2k$} \\
x_k \cdot x_k + y_k \cdot y_k , & \text{if $n = 2k+1$} \\
\end{cases}
$$

根据这一版的公式再次实现代码：

```golang
func fibV2Recursive(n int) Number {
	defer logElapsed("fibV2Recursive")()
	_, y := fibV2Helper(n)
	return y
}

func fibV2Helper(n int) (Number, Number) {
	if n <= 0 {
		return Num1, Num0
	}
	x, y := fibV2Helper(n / 2)
	xx := NumMul(x, x)
	xy := NumMul(x, y)
	yy := NumMul(y, y)
	if n % 2 != 0 {
		x = NumAdd(NumAdd(xx, xy), xy)
		y = NumAdd(xx, yy)
	} else {
		x = NumAdd(xx, yy)
		y = NumAdd(xy, NumSub(xy, yy))
	}
	return x, y
}
```

这次只需要 0.843 秒了，同样我们可以把递归改成迭代(速度没有明显差距)：

```golang
func fibV2Iterative(n int) Number {
	defer logElapsed("fibV2Iterative")()
	x, y := Num1, Num0
	bs := strconv.FormatInt(int64(n), 2)
	bs = strings.TrimLeft(bs, "0")
	for _, b := range bs {
		xx := NumMul(x, x)
		xy := NumMul(x, y)
		yy := NumMul(y, y)
		if b != '0' {
			x = NumAdd(NumAdd(xx, xy), xy)
			y = NumAdd(xx, yy)
		} else {
			x = NumAdd(xx, yy)
			y = NumAdd(xy, NumSub(xy, yy))
		}
	}
	return y
}
```

## 一种更慢的写法(Slow)
其实不管是V1还是V2，除了上面的递归和迭代两种写法，网上还能见到另一种写法：

```golang
func fibV1Slow(n int) Number {
	defer logElapsed("fibV1Slow")()
	r, a := Mat0, Mat1
	for m := n; m > 0; m /= 2 {
		if m != n {
			a = MatMul(a, a)
		}
		if m%2 != 0 {
			r = MatMul(r, a)
		}
	}
	return r[1][0]
}

func fibV2Slow(n int) Number {
	defer logElapsed("fibV2Slow")()
	x, y, a, b := Num1, Num0, Num1, Num1
	for m := n; m > 0; m /= 2 {
		if m != n {
			aa := NumMul(a, a)
			ab := NumMul(a, b)
			bb := NumMul(b, b)
			a = NumAdd(aa, bb)
			b = NumAdd(ab, NumSub(ab, bb))
		}
		if m%2 != 0 {
			xa := NumMul(x, a)
			xb := NumMul(x, b)
			ya := NumMul(y, a)
			yb := NumMul(y, b)
			x = NumAdd(xa, yb)
			y = NumAdd(ya, NumSub(xb, yb))
		}
	}
	return y
}
```

很不幸这种写法虽然更容易实现，但在绝大多数情况下要更慢一些，下表是各个函数在我电脑上的运行时间汇总：

|  n  | V1Rec | V1Iter | V1Slow | V2Rec | V2Iter | V2Slow |
| :-: | :---: | :----: | :----: | :---: | :----: | :----: |
| 10000000 | 2.435s | 2.423s | 3.254s | 0.843s | 0.837s | 1.374s |
| 16777215 | 5.353s | 5.329s | 7.785s | 1.909s | 1.887s | 3.558s |
| 16777216 | 5.314s | 5.315s | 5.460s | 2.023s | 1.969s | 1.875s |

想知道为什么可以查一查秦九韶算法(Horner's method)，我们原始的写法实际上是利用了这个方法进行优化的。

## 补充版本：Cassini's formula(V3)
V2版本看起来似乎已经做到极致了，因为对于一个 n 它只需要执行 3*log(n) 次乘法运算(数据大到一定程度时乘法运算是主要开销)。

还能再优化吗？可以的。

再来分析V2版本里面的 3 次核心乘法，其本质上就是：

$$
\begin{cases}
xx = f_{n+1} \cdot f_{n+1} \\
xy = f_{n+1} \cdot f_{n} \\
yy = f_{n} \cdot f_{n} \\
\end{cases}
$$

这时候我们需要用到一个叫做 Cassini's formula 的公式了：

$$ f_{n+1} \cdot f_{n-1} - f_{n} \cdot f_{n} = (-1)^{n} $$

证明很简单，考虑 R 矩阵的行列式就可以了，这里不多赘述。利用这个公式，我们可以进行以下推导：

$$ f_{n+1} \cdot f_{n} = f_{n+1} \cdot (f_{n+1} - f_{n-1}) = f_{n+1} \cdot f_{n+1} - f_{n} \cdot f_{n} - (-1)^{n} $$

也就是说，原来 xy 还可以由 xx 和 yy 得到，免去了一次乘法，基于此优化的代码如下：

```golang
func fibV3Recursive(n int) Number {
	defer logElapsed("fibV3Recursive")()
	_, y := fibV3Helper(n)
	return y
}

func fibV3Helper(n int) (Number, Number) {
	if n <= 0 {
		return Num1, Num0
	}
	x, y := fibV3Helper(n / 2)
	xx := NumMul(x, x)
	yy := NumMul(y, y)
	xy := NumSub(xx, yy)
	if n / 2 % 2 != 0 {
		xy = NumAdd(xy, Num1)
	} else {
		xy = NumSub(xy, Num1)
	}
	if n % 2 != 0 {
		x = NumAdd(NumAdd(xx, xy), xy)
		y = NumAdd(xx, yy)
	} else {
		x = NumAdd(xx, yy)
		y = NumAdd(xy, NumSub(xy, yy))
	}
	return x, y
}
```

这段代码在 n = 10000000 时运行了 0.522 秒，比起V2版本的 0.843 秒还要好不少，当然我们也可以把递归改成迭代(速度没有明显差距)：

```golang
func fibV3Iterative(n int) Number {
	defer logElapsed("fibV3Iterative")()
	x, y := Num1, Num0
	bs := strconv.FormatInt(int64(n), 2)
	bs = strings.TrimLeft(bs, "0")
	prev := '0'
	for _, curr := range bs {
		xx := NumMul(x, x)
		yy := NumMul(y, y)
		xy := NumSub(xx, yy)
		if prev != '0' {
			xy = NumAdd(xy, Num1)
		} else {
			xy = NumSub(xy, Num1)
		}
		if curr != '0' {
			x = NumAdd(NumAdd(xx, xy), xy)
			y = NumAdd(xx, yy)
		} else {
			x = NumAdd(xx, yy)
			y = NumAdd(xy, NumSub(xy, yy))
		}
		prev = curr
	}
	return y
}
```


最后附上相关的辅助代码：

```golang
type Number = *big.Int

var (
	Num0 = big.NewInt(0)
	Num1 = big.NewInt(1)
)

func NumAdd(x Number, y Number) Number {
	return big.NewInt(0).Add(x, y)
}

func NumSub(x Number, y Number) Number {
	return big.NewInt(0).Sub(x, y)
}

func NumMul(x Number, y Number) Number {
	return big.NewInt(0).Mul(x, y)
}

type Matrix = [2][2]Number

var (
	Mat0 = Matrix{ {Num1, Num0}, {Num0, Num1} }
	Mat1 = Matrix{ {Num1, Num1}, {Num1, Num0} }
)

func MatAdd(x Matrix, y Matrix) (z Matrix) {
	z[0][0] = NumAdd(x[0][0], y[0][0])
	z[0][1] = NumAdd(x[0][1], y[0][1])
	z[1][0] = NumAdd(x[1][0], y[1][0])
	z[1][1] = NumAdd(x[1][1], y[1][1])
	return
}

func MatSub(x Matrix, y Matrix) (z Matrix) {
	z[0][0] = NumSub(x[0][0], y[0][0])
	z[0][1] = NumSub(x[0][1], y[0][1])
	z[1][0] = NumSub(x[1][0], y[1][0])
	z[1][1] = NumSub(x[1][1], y[1][1])
	return
}

func MatMul(x Matrix, y Matrix) (z Matrix) {
	z[0][0] = NumAdd(NumMul(x[0][0], y[0][0]), NumMul(x[0][1], y[1][0]))
	z[0][1] = NumAdd(NumMul(x[0][0], y[0][1]), NumMul(x[0][1], y[1][1]))
	z[1][0] = NumAdd(NumMul(x[1][0], y[0][0]), NumMul(x[1][1], y[1][0]))
	z[1][1] = NumAdd(NumMul(x[1][0], y[0][1]), NumMul(x[1][1], y[1][1]))
	return
}

func logElapsed(name string) func() {
	start := time.Now()
	return func() {
		log.Printf("%s: %v", name, time.Since(start).Seconds())
	}
}
```
