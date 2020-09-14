## 哈希表
存放$n$个少量整数(整数本身属于一个$U=\{0,\ldots,2^w-1\}$很大范围中)的一个高效方式是使用哈希表。词汇 _哈希表_ 一批很广泛的数据结构。本章第一部分关注于哈希表中最常见的两个实现：链式哈希和线性探测哈希。

很多哈希表存储的数据类型不是整数。在这种情况下，每个数据元素都有与之关联的整形 _哈希码(hash code)_，这个码，被用于哈希表中。本章第二部分讨论如何生成这样的哈希码。

本章使用的某些方法要求在某个指定的整数范围随机选择。在代码示例中，这些"随机"整数有些是硬编码常量。这些常量是使用大气噪声产生的随机位获取的。

### 5.1 `ChainedHashTable`：链式哈希
`ChainedHashTable`数据结构使用 _链式哈希(hashing with chaining)_ 把数据存放到list数组`t`中。整数`n`记录所有list中全部元素总数(参见图5.1)：
```Java
List<T>[] t;
int n;
```
![figure5.1.png "n=14且t.length=16的ChainedHashTable例子。这个例子中hash(x)=6"](figure5.1.png "n=14且t.length=16的ChainedHashTable例子。这个例子中hash(x)=6")

一个数据元素`x`的 _哈希值(hash value)_，记为`hash(x)`是一个位于范围$\{0,\ldots,t.length-1\}$的值。所有哈希值为`i`的元素都存放在`t[i]`的list中。为了保证这个list不会太长，我们维护了不变量：
$$n \ge t.length$$
这样，这些list中单个包含的元素平均数是$n/t.length \lg 1$。

为了把一个元素`x`加入到哈希表中，我们首先检查`t`的长度是否需要增加，如果需要，我们就增长`t`。完成这个之后，我们通过对`x`执行哈希操作得到一个整数`i`，位于范围$\{0,\ldots,t.length-1\}$中，然后我们将`x`加入到list `t[i]`中：
```Java
boolean add(T x){
    if(find(x) != null) return false;
    if(n+1 > t.length) resize();
    t[hash(x)].add(x);
    n++;
    return true;
}
```
增长表操作(如果有必要)涉及到要倍增`t`的长度并把所有元素重新插入到新的表中。这个策略和曾在`ArrayStack`实现中使用的完全一样，因此也有同样的结果：当增长操作摊还到一系列插入操作时，增长的开销就只是常量时间(参见引理2.1)。

除了增长操作外，当给`ChainedHashTable`增加一个元素`x`时，其它工作就只是将`x`添加到list `t[hash(x)]`中。对于在第2和第3章中描述的其他任何list实现，都是花费常量时间。

为了从哈希表中删除一个元素`x`，我们在list `t[hash(x)]`上迭代查找`x`，然后删除它：
```Java
T remove(T x){
    Iterator<T> it = t[hash(x)].iterator();
    while(it.hasNext()){
        T y = it.next();
        if(y.equals(x)){
            it.remove();
            n--;
            return y;
        }
    }
    return null;
}
```
这会花费$O(n_{hash(x)})$的时间，其中$n_i$表示位于`t[i]`的list长度。

在哈希表中搜索元素`x`类似。我们在`t[hash(x)]`上执行线性搜索：
```Java
T find(Object x){
    for(T y : t[hash(x)]){
        if(y.equals(x)){
            return y;
        }
    }
    return null;
}
```
再一次，这个花费的时间正比于相应list `t[hash(x)]`的长度。

哈希函数是哈希表的性能的关键依赖。一个好的哈希函数会将元素平均地分布到`t.length`个list上，所以单个list t[hash(x)]的期望长度是$O(n/t.length)=O(1)$。另一方面，一个坏的哈希函数可能会把所有的值(包括`x`)哈希到同一个表位置，此时，单个list `t[hash(x)]`的长度就是`n`。下一节我们描述一个好的哈希函数。

#### 5.1.1 乘法哈希(multiplicative hashing)
乘法哈希(multiplicative hashing)是基于模运算(modular arithmetic)(在2.3节讨论过)和整数除法的高效哈希值成方法。他使用$\text{div}$操作符，只计算商的整数部分不考虑余数。形式化的说，对于任意整数$a \ge 0 \text{和} b \ge 1, a \text{div} b = \lfloor a/b \rfloor$。

在乘法哈希中，对于某个整数`d`(叫做 _维度(dimension)_)。哈希一个整数$x\in \{0,\ldots,2^w-1\}$公式是：
$$hash(x)=((z\cdot x)\bmod 2^w)\text{div}2^{w-d}$$
这里，$z$是从$\{1,\ldots,2^w-1\}$中选择的 _奇数_。默认情况，对于整数的操作已经完成了模$2^w$，这里$w$是一个整数的位数[<sup id="content1">1</sup>](#1)(参见图5.2)，所以这个操作可以被实现的非常高效。进一步的， 整数除以$2^{w-d}$等于丢掉这个整数二进制表示的最右边$w-d$位(通过使用$>>>$操作向右移动$w-d$位)。使用这种方式，代码可以以更简单的方式实现上述公式：

![figure5.2.png "w=32,d=8时乘法哈希函数操作"](figure5.2.png "w=32,d=8时乘法哈希函数操作")

```Java
int hash(Object x){
    return (z*x.hashcode())>>>(w-d);
}
```
如下引理，本节稍后证明，显示了乘法哈希对于避免冲突非常有效：
__引理5.1.__ 设`x`和`y`为范围$\{0,\ldots,2^w-1\}$中任意两个值，且$x\not ={y}$。那么$\text{Pr}\{hash(x)=hash(y)\}\le 2/2^d$。

使用引理5.1，`remove(x)`和`find(x)`的性能就很容易分析了：
__引理5.2.__ 对于任意数据值`x`，list `t[hash(x)]`的期望长度最多$n_x+2$，这里$n_x$是`x`在哈希表中出现的次数。
$\text{Proof}$ 设$S$为哈希表中不等于`x`的元素的集合。对于一个元素$y\in S$，定义指示器变量：
$$I_y=\begin{cases}
    1\quad\text{如果$hash(x) = hash(y)$}\\
    0\quad\text{其他情况}
\end{cases}$$
并且，注意到，根据引理5.1，$\mathrm{E}[I_y]\le 2/2^d = 2/t.length$。list `t[hash(x)]`的期望长度由如下等式给出：
$$\begin{aligned}
    \mathrm{E}[t[hash(x)].size()] &= \mathrm{E}\left[n_x+\sum_{y\in S}I_y\right]\\
    &= n_x+\sum_{y \in S}\mathrm{E}[I_y]\\
    &\le n_x+\sum_{y \in S}2/t.length\\
    &\le n_x+\sum_{y \in S}2/n\\
    &\le n_x+(n-n_x)2/n\\
    &\le n_x+2\\
\end{aligned}$$
这就证明了引理。

现在，我们要证明引理5.1，但首先我们需要一个数论的结论。在后面的证明中，我们使用符号$(b_r,\ldots,b_0)_2$表示$\sum_{i=0}^rb_i2^i$，这里每个$b_i$是一个bit，要么0，要么1。换句话说，$(b_r,\ldots,b_0)_2$是一个整数的二进制表示。我们使用$\star$标记一个bit的值未知。

__引理5.3.__ 设$S$为$\{1,\ldots,2^w-1\}$中奇数集合；设`q`和`i`为$S$中任意两个值。那么就唯一存在一个值$z\in S$满足$zq \bmod 2^w=i$。

$\text{Proof}$ 对于$z$和$i$的选择是相同的，就足够证明最多有一个值$z\in S$满足$zq \bmod 2^w=i$。

假设，不失矛盾的，有两个值$z$和$z'$，有$z \gt z'$。那么：
$$zq \bmod 2^w = z'q\bmod 2^w = i$$
所以，
$$(z-z')q \bmod 2^w = 0$$
单这意味着
$$\tag{5.1} (z-z')q = k2^w$$
其中，$k$是某个整数。按照二进制数考虑，我们有：
$$(z-z')q = k\cdot(1,\underbrace{0,\ldots,0}_{m})_2$$
所以$(z-z')q$的$w$个末尾bit位二级制表示都是0。

进一步的，$k\neq 0$，因为$q \neq 0$且$z-z' \neq 0$。由于$q$是奇数，它的二进制表示是没有拖尾的0：
$$q=(\star,\cdots,\star,1)_2$$
由于$\vert z-z'\vert \lt 2^w$，$z-z'$的二进制表示拖尾0要小于$w$：
$$z-z' = (\star,\cdots,\star,1,\underbrace{0,\cdots,0}_{\lt w})_2$$
因此，$(z-z')q$积的二进制表示拖尾0就少于$w$个：
$$(z-z')q=(\star,\cdots,\star,1,\underbrace{0,\cdots,0}_{\lt w})_2$$
因此，$(z-z')q$无法满足等式(5.1)，从而产生了矛盾完成了证明。

对引理5.3的利用来自于如下的观察：如果$z$按照随机的方式均匀地从$S$选择，那么$zt$就会均匀地分布在$S$上。在接下来的证明中，使用$z$的二进制表示(由$w-1$个随机bit后跟着一个1组成)来思考问题会很有帮助。

_引理5.1的证明_ 首先我们注意到条件$hash(x) = hash(y)$和语句"$zx \bmod 2^w$最高顺序的$d$个位(bit)和$zy \bmod 2^w$最高顺序的$d$个位(bit)是一样的"是等价的。这个语句的一个必要条件是$z(x-y)\bmod 2^w$的二进制表示最高顺序的$d$个位(bit)。也就是说：
$$\tag{5.2} z(x-y)\bmod 2^w = (\underbrace{0,\ldots,0}_{d},\underbrace{\star,\ldots,\star}_{w-d})_2$$

当$zx\bmod 2^w \gt zy \bmod 2^w$或者
$$\tag{5.3}  z(x-y)\bmod 2^w = (\underbrace{1,\ldots,1}_{d},\underbrace{\star,\ldots,\star}_{w-d})_2$$
当$zx \bmod 2^w \lt zy \bmod 2^w$。因此，我们只需要界定$z(x-y)\bmod 2^w$形如$(5.2)$或者$(5.3)$的概率。

对于某个整数$r \ge 0$，设$q$是满足$(x-y)\bmod 2^w = q2^r$的唯一奇数。根据引理5.3，$zq\bmod 2^w$的二进制表示是$w-1$个随机bit后跟着一个1：
$$zq\bmod 2^w = (\underbrace{b_{w-1},\ldots,b_1}_{w-1},1)_2$$
因此，$z(x-y)\bmod 2^w = zq2^r\bmod 2^w$的二进制表示有$w-r-1$各个随机bit后跟一个1，然后后跟$r$个0：
$$z(x-y)\bmod 2^w = zq2^r\bmod 2^w = (\underbrace{b_{w-r-1},\ldots,b_1}_{w-r-1},1,\underbrace{0,0,\ldots,0}_{r})_2$$
现在我们可以完成证明：如果$r\gt w-d$，那么$z(x=y)\bmod 2^2$的$d$个较高bit位包含的都是0和1，所以$z(x-y)\bmod 2^w$形如$(5.2)$或者$(5.3)$的概率是0。如果$r=w-d$，$z(x-y)\bmod 2^w$形如$(5.2)$的概率是0，但是形如$(5.3)$的概率是$1/2^{d-1}=2/2^d$(因为我们必须有$b_1,\ldots,b_{d-1} = 1,\ldots,1$)。如果$r\lt w-d$，那么我们必须有$b_{w-r-1},\ldots,b_{w-r-d} = 0,\ldots,0$或者$b_{w-r-1},\ldots,b_{w-r-d} = 0,\ldots,0$。这两种情况的概率都是$1/2^d$，而且它们之间都是互斥的(mutually exclusive)，所以每种情况的概率都是$2/2^d$。这就完成了证明。

#### 5.1.2 总结
如下定理总结了`ChainedHashTable`数据结构的性能：
__定理5.1__ `ChainedHashTable`实现了`USet`接口。忽略调用`grow()`的开销，`ChainedHashTable`支持每个操作期望运行时间是$O(1)$的`add(x)`，`remove(x)`和`find(x)`。

进一步的，从一个空的`ChainedHashTable`开始，以任意顺序对`add(x)`和`remove(x)`$m$次调用操作会花费一共$O(m)$的时间用来调用全部的`grow()`。

[<sup id="content1">1</sup>](#content1)对于大多数编程语言来说，包括`C`，`C#`，`C++`和`Java`。明显的例外是`Python`和`Ruby`，在这两门语言中，如果一个固定长度是`w`位的整数操作溢出了，会升级到可变长度的表示。