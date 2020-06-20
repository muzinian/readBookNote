## 基于数组的List
本章，我们将会学习List和Queue接口的实现，底层数据是存在数组中的，这个数组也叫做 _支撑数组(backing array)_。下表总结了本章出现的数据结构各个操作的运行时间
||get(i)/set(i,x)|add(i,x)/remove(i)|
|-|-|-|
|ArrayStack|O(1)|O(n-i)|
|ArrayDeque|O(1)|O(min{i,n-i})|
|DualArrayDeque|O(1)|O(min{i,n-i})|
|RootishArrayStack|O(1)|O(n-i)|
那些把数据存放在单个数组中的数据结构有很多共通的优势和限制：
* 数组可以在常量时间内任意一个数组值。这就是为什么`get(i)`和`set(i,x)`可以在常量时间运行。
* 数组不是十分动态的。在List的中间范围增加或者删除一个元素意味着需要移动数组中大量的元素，才能给新加入的元素提供空间，或者填充由于删除元素导致的间隙。这就是为什么`add(i,x)`和`remove(i)`操作的运行时间依赖$n$和$i$。
* 数组无法扩展或者收缩。当数据结构中元素的个数操过了支撑数组的大小，我们就需要分配一个新的数组并把旧数组的元素拷贝到新数组中。这是一个很昂贵的操作。
  
第三点很重要。上表中的运行时间没有包括相关支撑数组的增长或者收缩的开销。我们将看到，如果仔细管理，支撑数组的增长和收缩开销不会对平均操作的开销增加太多。更精细的，如果我们从一个空数组开始，并执行任意$m$个`add(i,x)`或者`remove(i)`操作序列(perform any sequence of $m$ `add(i,x)` or `remove(i)` operations,then the total cost of growing and shrinking the backing array,over the entire sequence of $m$ operations is $O(m)$)，那么支撑数组增加或者收缩的全部开销，在整个$m$序列操作中，是$O(m)$。尽管某些独立的操作更加昂贵，但是摊还开销，当他摊还到全部$m$个操作中时，每个操作是$O(1)$的。

### 2.1 ArrayStack:使用数组的快速栈操作
一个ArrayStack使用数组`a`实现了list接口，这个数组叫做 _支撑数组_。下标为`i`的list元素存放在数组`a[i]`中。在大多数时间内，数组`a`的大小比真正需要的要大，所以整数$n$用来跟踪实际存放在`a`中的元素个数。使用这种方式，list的元素被存放在`a[0],...,a[n-1]`中，并且，在整个时间，`a.length>n`。
```java
T[] a;
int n;
int size() {
    return n;
}
```
#### 2.1.1 基础
使用`get(i)`和`set(i,x)`访问和修改`ArrayStack`的元素很简单。在执行必要的边界检查后，我们简单的返回或者设置对应的`a[i]`。
```java
T get(int i){
    return a[i];
}
T set(int i,T x){
    T y = a[i];
    a[i] = x;
    return y;
}
```
图2.1展示了从`ArrayStack`中增加和删除元素的操作。为了实现`add(i,x)`操作，我们首先检查a是不是满的。如果是，我们调用`resize()`方法增加`a`的大小。我们稍后讨论如何实现`resize()`操作。现在，知道在调用`resize()`后我们可以确保`a.length>n`就足够了。使用这个方法，我们现在对元素`a[i],...,a[n-1]`右移一个元素从而给`x`移动空间，设置`a[i]`为x，然后增加`n`。

![figure2.1.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致调用`resize()`的操作标记了一个星号"](#figure2.1.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致调用`resize()`的操作标记了一个星号")

```java
void add(int i,T x){
    if(n+1>a.length) resize();
    for(int j=n;j>i;j--){
        a[j] = a[j-1];
    }
    a[i] = x;
    n++;
}
```
如果我们忽略对`resize()`潜在调用的开销，那么`add(i,x)`操作的开销正比于我们要为`x`移动位置的元素个数。因此这个操作的开销(忽略改变`a`大小的开销)是$O(n-i)$。

`remove(i)`操作的实现是类似的。对元素`a[i+1],...,a[n-1]`每个我们都左移一个位置(覆写了`a[i]`)然后减少`n`的值。这样做之后，我们通过`a.length>3n`检查`n`是不是比`a.length`小很多。如果是满足了，我们调用`resize()`减少`a`的大小。
```java
T remove(int i){
    T x = a[i];
    for(int j=i;j<n-1;j++){
        a[j] =a[j+1];
    }
    n--;
    if(a.length>=3*n)resize();
    return x;
}
```
如果我们忽略`resize()`方法的开销，`remove(i)`操作的开销正比于我们要移动的元素个数，即$O(n-i)$。

#### 2.1.2 增长和缩小
`resize()`方法相当直接；它分配一个新的数组`b`大小是$2n$，然后拷贝`a`的那个元素到`b`开始的`n`个位置，然后设置`b`为`a`。因此，在调用`resize()`之后，`a.length=2n`。
```java
void resize(){
    T[] b = new Array(max(n*2,1));
    for(int i = 0;i<n;i++){
        b[i] = a[i];
    }
    a = b;
}
```
分析`resize()`操作的实际开销很容易。它分配一个大小为$2n$的的数组`b`然后拷贝`a`的`n`个元素到`b`中。这会花费$O(n)$时间。

前面章节中运行时间的分析忽略了调用`resize()`的开销。本段我们使用一种叫做 _摊还分析(amortized analysis)_ 的技术来分析这个开销。这个技术不会试图确定单个`add(i,x)`和`remove(i)`操作中调整尺寸的开销。相反，它考虑了由`add(i,x)`或者`remove(i)`组成的`m`个调用序列中全部`resize()`调用的开销。特别的，我们会展示：

__引理 2.1__ 如果创建一个空的`ArrayStack`然后执行任意$m \ge 1$个`add(i,x)`和`remove(i)`序列，那么全部`resize()`调用的开销时间是$O(m)$。

$\textit{Proof.}$我们会展示任意时间调用`resize()`，从上一次调用`resize()`后调用`add`或者`remove`的次数至少是$n/2-1$。因此，如果$n_i$代表在第$i$次调用`resize()`时`n`的值而$r$代表调用`resize()`的次数，那么调用`add(i,x)`或者`remove(i)`的全部数量至少是：
$$\sum_{i=1}^r(n_i/2-1)\le m\quad ,$$
它等于
$$\sum_{i=1}^r(n_i)\le 2m+2r$$
另一方面，对`resize()`所有调用的全部时间花费是：
$$\sum_{i=1}^rO(n_i)\le O(m+r) = O(m)\quad,$$
因为$r$不会比$m$大。所有这些显示了在第$(i-1)$次和第$i$次调用`resize()`之间调用`add(i,x)`或者`remove(i)`的次数至少是$n_i/2$。

这里需要考虑两种情况。第一种，`resize()`是因为调用`add(i,x)`时支撑数组`a`满了被调用的，例如$a.length=n=n_i$。考虑上一次对`resize()`的调用：再上一次调用后，`a`的大小是`a.length`，但是存放在`a`中元素数量是$a.length/2=n_i/2$。但是现在存放在`a`中元素个数是$n_i=a.length$，所以从上一次调用`resize()`之后至少是发生了$n_i/2$次对`add(i,x)`的调用。

第二种情况发生在`resize()`是由于在执行`remove(i)`时$a.length \ge 3n=3n_i$。再一次， 在上一次调用`resize()`之后存放在数组`a`中的元素个数至少是$a.length/2-1$[<sup id="content1">1</sup>](#1)。现在有
$n_i \le a.length/3$个元素存放在`a`中。因此，从上一次调用`resize()`后`remove(i)`操作的数量至少是：
$$\begin{aligned}
    R &\ge a.length/2-1-a.length/3 \\
    &= a.length/6-1 \\
    &=(a.length/3)/2-1 \\
    &\ge n_i/2-1 \quad.
\end{aligned}$$
在这两种情况下，在第$(i-1)$次和第$i$次调用`resize()`之间`调用`add(i,x)`或者`remove(i)`的数量至少是$n_i/2-1$，证明完成。$\square$ 

[<sup id='1'>1</sup>](#content1)公式中的$-1$是为了防止$n=0$且$a.length=1$这个特殊例子。

#### 2.1.3 总结
下面的定理总结了`ArrayStack`的性能：
__定理2.1__ `ArrayStack`实现了`List`接口。忽略`resize()`调用的开销，`ArrayStack`支持的操作：
* `get(i)`和`set(i,x)`每个操作开销是$O(1)$；并且
* `add(i,x)`和`remove(i)`每个操作的开销是$O(1+n-i)$
进一步，从一个空的`ArrayStack`开始，执行任意$m$个`add(i,x)`和`remove(i)`操作序列中全部`resize()`调用一共花费了$O(m)$时间。

`ArrayStack`是`Stack`接口的一个高效实现。特别的，我们可以使用`add(n,x)`实现`push(x)`，使用`remove(n-1)`实现`pop()`，这些操作的运行时间是$O(1)$摊还时间。

### 2.2 `FastArrayStack`：优化的`ArrayStack`
`ArrayStack`大部分工作的完成涉及移动(通过`add(i,x)`和`remove(i)`)和复制(通过`resize()`)数据。上面的实现中，是通过`foo`循环完成的。事实上很多编程环境有特殊的函数可以非常高效的复制和移动数据块。在C语言中，`memcpy(d,s,n)`和`memmove(d,s,n)`函数可以完成这项工作。在C++中，存在一个`std::copy(a0,a1,b)`算法。Java中存在`System.arraycopy(s,i,d,j,n)`方法。
```java
void resize() {
    T[] b = newArray(max(2*n,1));
    System.arraycopy(a, 0, b, 0, n);
    a = b;
}
void add(int i, T x) {
    if (n + 1 > a.length) resize();
    System.arraycopy(a, i, a, i+1, n-i);
    a[i] = x;
    n++;
}
T remove(int i) {
    T x = a[i];
    System.arraycopy(a, i+1, a, i, n-i-1);
    n--;
    if (a.length >= 3*n) resize();
    return x;
}
```
这些函数通常是高度优化的甚至是使用了特殊的机器指令使得复制操作比我们使用`for`循环快很多。尽管使用这些函数不会渐进的降低运行时间，它依旧是一个值得的优化。

这里的Java实现，对于原生`System.arraycopy(s,i,d,j,n)`的使用会有2到3之间因子的提升，这一来执行的操作类型。你的结果可能很不同。

### 2.3 `ArrayQueue`：基于数组的队列
本节中，我们展示一个实现了FIFO(先进先出)队列的数据结构`ArrayQueue`；元素从队列中删除的顺序(使用`remove()`操作)和它们加入队列的顺序是一样的(使用`add(x)`操作)。

注意使用`ArrayStack`实现一个FIFO队列是一个糟糕的选择。原因是我们必须要选择列表的一端添加元素然后从另一端删除元素。这两个操作中的一个要在链表头工作，通过调用`add(i,x)`或者`remove(i)`实现，这里$i=0$。这个操作的运行时间正比与$n$。

为了得到一个基于数组的高效队列实现，我们首先注意到如果我们有一个无限数组`a`就很容易实现。我们可以维护一个索引`j`跟踪下一个要删除的元素和一个整数`n`统计了队列中元素的个数。队列中的元素总时存放在
$$a[j],a[j+1],...,a[j+n-1]$$
开始时，`j`和`n`都被设置为`0`。为了增加一个元素，我们会把元素放到`a[j+n]`并对`n`加一。为了删除一个元素，我们从`a[j]`中删除它并对`j`加一，对`n`减一。

当然，这个方案的问题要求一个无限数组。`ArrayQueue`通过一个有限数组`a`和 _模运算(modular arithmetic)_ 模拟了这个。当我们谈论一天的时间就使用的是这种运算。例如10:00加5个小时是3:00。形式化的，我们说：
$$10+5 = 15\equiv 3\pmod {12}.$$
等式后面一部分读作"15对12取模等于3"。我们也把$\bmod$看作一个二元操作符，所以：
$$15 \bmod 12 = 3$$
更一般的是，对于一个整数$a$和正整数$m$，$a\bmod m$的结果是唯一的整数$r\in \{0,...,m-1\}$对某个整数$k$使得$a=r+km$。不太形式化的说，值$r$是我们用$m$除$a$后得到的余数。在很多计算机语言中，包括Java，mod操作是使用%表示[<sup id="content2">2</sup>](#2)。

模运算对于模拟无限数组很有帮助，因为$i \bmod a.length$结果总是位于$0,...,a.length-1$中。使用模运算我们可以在数组中存放队列元素。
`a[j%a.length],a[(j+1)%a.length],...,a[(j+n-1)%a.length]`

它把数组看一个 _循环数组(circular array)_，如果大于`a.length-1`的数组索引会绕回到数组开始。

唯一需要担心的是`ArrayQueue`元素数量不要超过`a`的大小。
```Java
T[] a;
int j;
int n;
```
对`ArrayQueue`的`add(x)`和`remove()`操作序列展示如图2.2。为了实现`add(x)`，我们先检查`a`是否满了，并且，如果需要的话，调用`resize()`来增加`a`的大小。下一步，我们存放`x`到`a[(j+n)%a.length]`并对`n`

![figure2.2.png "对`ArraQueue`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致调用`resize()`的操作标记了一个星号"](#figure2.2.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致调用`resize()`的操作标记了一个星号")

```java
boolean add(T x) {
    if (n + 1 > a.length) resize();
    a[(j+n) % a.length] = x;
    n++;
    return true;
}
```
为了实现`remove()`，我们先存储a[j]这样在后面可以返回它。然后，我们对`n`减一并通过设置$j = (j+1)\bmod a.length$增加$j\pmod {a.length}$。最后，我们返回存放的$a[j]$值。如果需要，我们可以调用`resize()`来减少`a`的尺寸。
```Java
T remove() {
    if (n == 0) throw new NoSuchElementException();
    T x = a[j];
    j = (j + 1) % a.length;
    n--;
    if (a.length >= 3*n) resize();
    return x;
}
```
最后，`resize()`操作和`ArrayStack`的`resize()`操作非常类似。它分配一个新的数组，`b`，大小是`2n`并拷贝
`a[j%a.length],a[(j+1)%a.length],...,a[(j+n-1)%a.length]`
到
`b[0],b[1],...,b[n-1]`
并设置`j=0`。
```Java
void resize() {
    T[] b = newArray(max(1,n*2));
    for (int k = 0; k < n; k++)
      b[k] = a[(j+k) % a.length];
    a = b;
    j = 0;
}
```
#### 2.3.1 总结
下面定理总结了`ArrayQueue`数据结构的性能：
__定理2.2.__ 一个`ArrayQueue`实现了(FIFO)`Queue`接口。忽略调用`resize()`的开销，`ArrayQueue`支持$O(1)$的`add(x)`和`remove()`操作。进一步的，从一个空`ArrayQueue`开始，任意$m$个`add(i,x)`和`remove(i)`操作序列会导致对`resize()`全部调用一共会花费$O(m)$的时间。

### 2.4 `ArrayDeque`：使用数组的快速双端队列操作
上一节的`ArrayQueue`表示的是一个允许我们高效的在一端添加在另一端删除的序列。`ArrayDeque`数据结构允许我们高效的在双端天机和删除。这个结构同样使用了用于表示`ArrayQueue`的循环数组技术实现了`List`接口
```Java
T[] a;
int j;
int n;
```
`ArrayDeque`的`get(i)`和`set(i,x)`操作很直接。它们获取或者设置数组元素$a[(j+1) \bmod a.length]$。
```java
T get(int i){
    return a[(j+i)%a.length]
}
T set(int i,T x){
    T y = a[(j+i)%a.length];
    a[(j+i)%a.length] = x;
    return y;
}
```
`add(i,x)`的实现更有意思点。想通常那样，我们先检查`a`是不是满的，如果需要，调用`resize()`来调整`a`。记住我们想要这个操作很快当`i`很小(接近0)或者当`i`很大(接近$n$)。因此，我们检查是否$i \lt n/2$。如果是，我们对元素`a[0],...,a[i-1]`左移一个位置。否则($i \ge n/2$)，我们对元素`a[i],...,a[n-1]`右移一个位置。图2.3解释了`ArrayDeque`的`add(i,x)`和`remove(x)`操作。

![figure2.3.png "对`ArrayDeque`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。"](#figure2.2.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。")

```Java
void add(int i,T x){
    if(n+1>a.length) resize();
    if(i<n/2){//左移`a[0],...,a[i-1]`一个位置
        j = (j == 0)?a.length - 1:j-1;//(j-1)mod a.length
        for(int k = 0; k<=i-1;k+=){
            a[(j+k)%a.length] = a[(j+k+1)%a.length];
        }
    }else{
        for (int k = n; k > i;k--){// 右移 a[i],..,a[n-1] 一个位置
            a[(j+k)%a.length] = a[(j+k-1)%a.length];
        }
    }
}
```
按照这种方式移动元素，`add(i,x)`永远不会移动超过$\min\{i,n-1\}$个元素。因此，`add(i,x)`操作运行时间(忽略`resiz()`操作的开销)是$O(1+\min \{i,n-i\})$

`remove(i)`操作的实现蕾西。通过判断`i<n/2`,他要么对元素`a[0],...,a[i-1]`右移一个位置，要么对元素`a[i+1],...,a[n-1]`右移一个位置。再一次，这意味着`remove(i)`永远不会花费超过$O(1+\min \{i,n-i\})$的时间移动元素。
```Java
void add(int i,T x){
    T x = a[(j+i)%a.length];
    if(i<n/2){//右移`a[0],...,a[i-1]`一个位置
        for (int k = i;k > 0;k--){
            a[(j+k)%a.length] = a[(j+k-1)%a.length];
        }
        j = (j+1)%a.length;
    }else{
        for (int k = i ;k<n-1;k++){ //左移` a[i+1],..,a[n-1] `一个位置
            a[(j+k)%a.length] = a[(j+k+1)%a.length];
        }
    }
    n--;
    if(3*n<a.length) resize();
    return x;
}
```
#### 2.4.1 总结
下面的定理总结了`ArrayDeque`数据结构的性能：
__定理2.3.__ 一个`ArrayDeque`实现了`List`接口。忽略`resize()`调用的开销，一个`ArrayDeque`支持操作：
* 运行时间为$O(1)$的`get(i)`和`set(i,x)`；并且
* `add(i,x)`和`remove(i)`操作的运行开销是$O(1+\min \{i,n-i\})$

进一步的，从一个空`ArrayDeque`开始，执行任意$m$个`add(i,x)`和`remove(i)`操作序列会导致所有对`resize()`的调用一共花费$O(m)$的时间。



[<sup id="2">2</sup>](#content2)有时候也叫做 _脑死亡_ mod操作，因为当第一个参数是负数的时候，它没有被正确的实现数学上的mod操作。
