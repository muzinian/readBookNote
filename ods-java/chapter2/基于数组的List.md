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

[<sup id="1">1</sup>](#content1)公式中的$-1$是为了防止$n=0$且$a.length=1$这个特殊例子。

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

[<sup id="2">2</sup>](#content2)有时候也叫做 _脑死亡_ mod操作，因为当第一个参数是负数的时候，它没有被正确的实现数学上的mod操作。

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

![figure2.3.png "对`ArrayDeque`的`add(x)`和`remove()`操作。箭头表示元素被复制了。"](#figure2.2.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。")

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

### 2.5 `DualArrayDeque`：使用两个栈构建一个双端队列
下一步，我们展示一个数据结构：`DualArrayDeque`，它使用两个`ArrayStack`达到了和`ArrayDeque`有同样的性能限制。尽管`DualArrayDeque`的渐进性能不比`ArrayDeque`好，它依旧值得我们学习，因为对于如何使用两个简单的数据结构结合成一个更复杂的数据结构，它提供了很好的范例。

一个`DualArrayDeque`表示一个使用了两个`ArrayStack`的链表。回忆一下，当一个操作是修改的元素靠近`ArrayStack`端点时这个操作就很快。`DualArrayDeque`背靠背放置两个`ArrayStack`，叫做`front`和`back`，这样操作在哪一端都很快。
```Java
List<T> front;
List<T> back;
```
一个`DualArrayDeque`不会显式存储它存放的元素个数`n`。它不需要，因为他存放了$n=front.size()+back.size()$个元素。然而，在分析`DualArrayDeque`时，我们依旧使用`n`表示它包含的元素个数。
```Java
int size(){
    return front.size()+back.size();
}
```
`front` `ArrayStack`存放的链表元素索引从`0,...,front.size()-1`，但是是按照逆序存放的。`back` `ArrayStack`包含的链表元素按照正常顺序存放，索引范围是`front.size(),...,size()-1`。按照这种方式，`get(i)`和`set(i,x)`会被对应的翻译为对`front`或者`back`的`get(i)`和`set(i,x)`的调用，每个操作时间开销是$O(1)$。
```Java
T get(int i){
    if(i<front.size()){
        return front.get(front.size()-i-1);
    }else{
        return back.get(i-front.size());
    }
}
T set(int i,T x){
    if(i<front.size()){
        return front.set(front.size()-i-1,x);
    }else{
        return back.set(i-front.size(),x);
    }
}
```
注意如果索引`i`满足`i<front.size()`，那么，它对应到的`front`元素位置是在`front.size()-i-1`，因为`front`的元素是按照逆序方式存放的。

从`DualArrayDeque`中添加或者删除一个元素在图2.4中展示了。`add(i,x)`操作的是对应的`front`或者`back`：
```Java
void add(int i,T x){
    if(i<front.size()){
        front.add(front.size()-i,x);
    }else{
        back.add(i-front.size(),x);
    }
    balance();
}
```
![figure2.4.png "对`DualArrayDeque`的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致使用`balance()`函数重平衡的操作用星号标记了。"](#figure2.2.png "对`ArrayStack`的的`add(x)`和`remove()`操作。箭头表示元素被复制了。导致使用`balance()`函数重平衡的操作用星号标记了")

`add(i,x)`方法通过调用`balance()`方法对`front`和`back`这两个`ArrayStack`执行了再平衡操作。`balance()`的实现在后面描述，现在，知道`balance()`保证了除非`size()<2`，`front.size()`和`back.size()`不会相差超过三倍(`front.size()` and `back.size()` do not differ by more than a factor of 3)。具体地说，$3 \cdot front.size() \ge back.size()$且$3 \cdot back.size() \ge front.size()$。

接下来我们在忽略`balance()`调用开销的情况下分析`add(i,x)`的开销。如果`i<front.size()`，那么`add(i,x)`是通过调用`front.add(front.size()-i-1,x)`实现的。因为`front`是一个`ArrayStack`，这个开销就是：
$$O(front.size()-(front.size()-i-1)+1) = O(i+1).(2.1)$$
另一方面，如果`i>front.size()`，那么`add(i,x)`是通过`back.add(i-front.size(),x)`实现的。开销是：
$$O(back.size()-(i-front.size())+1) = O(n-i+1).(2.2)$$
注意，第一个情况(2.1)发生在$i<n/4$。第二种情况出现在$i\ge 3n/4$。当$n<4\le i <3n/4$，我们不能确定影响的是`front`或者`back`的哪个，但是在两种情况下，操作都会花费$O(n)=O(i)=O(n-i)$的时间，因为$i\ge n/4$且$n-i>n/4$。总结一下情况，我们有：
$$add(i,x)\text{运行时间}\le \begin{Bmatrix}
    \begin{aligned}
        &O(1+i) \qquad 如果\; i< n/4\\
        &O(n) \qquad 如果\;n/4 \le <3n/4\\
        &O(1+n-i) \qquad 如果\;i \ge 3n/4 
    \end{aligned}
\end{Bmatrix}$$
因此，在忽略调用`balance`的开销，`add(i,x)`的运行时间是$O(1+min\{i,n-i\})$。
```Java
T remove(int i){
    T x;
    if(i < front.size()){
        x = front.remove(front.size()-i-1);
    }else{
        x = back.remove(i-front.size());
    }
    balance();
    return x;
}
```
#### 2.5.1 平衡操作
最后，我们转向`add(i,x)`和`remove(i)`执行的`balance()`操作。这个操作确保了无论`front`还是`back`都不会变得太大(或者太小)。它保证了，除非链表中元素小于两个，那么`front`和`back`至少包含$n/4$个元素。如果不是这种情况，那么他就会在两者之间移动元素这样`front`和`back`就分别准确的包含了$\lfloor n/2\rfloor$个元素和$\lceil n/2\rceil$元素。
```Java
void balance(){
    int n = size();
    if(3*front.size()<back.size()){
        int s = n/2 - front.size();
        List<T> l1 = newStack();
        List<T> l2 = newStack();
        l1.addAll(back.subList(0,s));
        Collections.reverse(l1);
        l1.addAll(front);
        l2.addAll(back.subList(s,back.size()));
        front = l1;
        back = l2;
    } else if(3*back.size()< front.size()){
        int s = front.siez() - n/2;
        List<T> l1 = newStack();
        Lsit<T> l2 = newStack();
        l1.addAll(front.subList(s,front.size()));
        l2.addAll(front.subList(0,s));
        Collections.reverse(l2);
        front = l1;
        back = l2;
    }
}
```
这里我们稍微分析下。如果`balance()`操作进行了再平衡操作，那么它移动了`O(n)`个元素并花费了`O(n)`的时间。这很糟糕，因为`balance()`在每次调用`add(i,x)`和`remove(i)`时都会被调用。然而，下面的引理显示了，平均情况下，对于每次操作，`balance()`只花费常量的时间。

__引理2.2__ 如果创建了一个空`DualArrayDeque`并且以任意顺序调用了`add(i,x)`和`remove(i)`$m \ge 1$次，那么对`balance()`所有调用花费的全部时间是$O(m)$。

$\textit{证明.}$我们将展示，如果`balance()`需要移动元素，那么从上一次有元素被`balance()`移动到这一次，期间`add(i,x)`和`remove(i)`操作执行次数至少是$n/2-1$。在引理2.1的证明中，这就足够证明`balance()`操作花费的时间是$O(m)$。

我们将使用一个叫做 _势能方法(potential method)_ 的技术进行分析。定义`DualArrayDeque`的 _势能(potential)_，$\Phi$，为`front`和`back`的差值：
$$\Phi = |front.size()-back.size()|$$
关于这个势能有意思的是对于不执行任何平衡操作的`add(i,x)`和`remove(i)`最多会对势能加一。

观察到，在调用`balance()`操作移动了元素后，此时势能$\Phi_{0}$，最多等于1，因为：
$$\Phi_{0} = |\lfloor n/2\rfloor - \lceil n/2 \rceil| \le 1.$$

考虑马上就要调用会移动元素的`balance()`操作前，并假设，不失一般性的，`balance()`操作是因为$3front.size() \lt back.size()$。注意到，在这种情况下：
$$\begin{aligned}
  n = &front.size() + back.size()\\
  & back.size()/3 + back.size() \\
  &=\frac{4}{3}back.size()
\end{aligned}$$
进一步的，在此刻势能就是：
$$\begin{aligned}
    \Phi_{1} &= back.size() - front.size()\\
    &\gt back.size() -back.size()/3\\
    &= \frac{2}{3}back.size()\\
    &> \frac{2}{3}\times\frac{3}{4}n\\
    &= n/2
\end{aligned}$$
因此，从上一次移动元素的`balance()`从左后调用`add(i,x)`和`remove(i)`的次数至少是$\Phi_{1}-\Phi_{0} \gt n/2-1.$这就完成了证明。$\square$
#### 25.2 总结
下面的定理总结了`DualArrayDeque`的特性：
__定理2.4__ 一个`DualArrayDeque`实现了`List`接口。忽略掉调用`resize()`和`balance()`的开销，`DualArrayDeque`支持
* `get(i)`和`set(i,x)`的时间开销是$O(1)$的，并且
* `add(i,x)`和`remove(i)`的时间开销是$O(1+\min\{1,n-i\})$。
进一步的，从一个空的`DualArrayDeque`开始，$m$次任意的调用`add(i,x)`和`remove(i)`操作最终会导致全部对`resize()`和`balance()`的调用一共花费$O(m)$时间。

### 2.6 `RootishArrayStack`:一个空间高效的数组栈
本章前面所有数据结构的缺点之一就是：由于它们把数据存到一到两个数组中并且避免太平凡的调整数组大小，这些数组频繁的处于不是很满的状况。例如，对于一个刚刚进行了进行`resize()`操作的`ArrayStack`，后端数组只有填充了一半。更糟糕的是，有时候只有三分之一包含数据。

在本段，我们将讨论了`RootishArrayStack`数据结构，它解决了空间浪费的问题。`RootishArrayStack`使用$O(\sqrt{n})$个数组存放$n$个元素。在这些数组中，任意时刻最多有$O(\sqrt{n})$个数组位置没有被使用。其他所有剩下的数组位置都用来存放数据了。因此，这些数据结构在存放$n$个元素时，最多浪费了$O(\sqrt{n})$个空间。

`RootishArrayStack`把它的元素存放在一个由$r$个数组组成的列表中，这些数组又叫做 _块(block)_，编号`0,1,...,r-1`。如图2.5。

![figure2.5.png "对RootishArrayStack执行一系列add(i,x)和remove(i)操作。箭头表示元素被复制"](figure2.5.png "对RootishArrayStack执行一系列add(i,x)和remove(i)操作。箭头表示元素被复制")

块`b`包含`b+1`个元素。因此，所有`r`个块一共包含:
$$1+2+3+\cdot \cdot \cdot +r=r(r+1)/2$$
个元素。上面的等式可以根据图2.6获取到。
![figure2.6.png "白色方块个数是$1+2+3+...+r$。灰色方块个数是一样的。灰色和白色方块加在一起得到矩形是由$r(r+1)个方块组成的$"](figure2.6.png "白色方块个数是1+2+3++r。灰色方块个数是一样的。灰色和白色方块加在一起得到矩形是由r(r+1)个方块组成的")
```Java
List<T[]>blocks;
int n;
```
正如我们所期望的，list的元素在blocks内是有序排列的。list中下标为0的元素存在块(block)`0`中。下标是1和2的元素存放在块(block)`1`中，下标为3，4和5的存放在块(block)`2`中，以此类推。我们要解决的主要问题是给定一个下标`i`，怎么确定哪个block包含下标`i`的元素，以及block内`i`对应的下标。

事实上，确定`i`在包含它的block中的索引很容易。如果索引`i`位于块b中，那么位于块`0,...,b-1`的元素个数是$b(b+1)/2$。因此，`i`存放在块`b`内位置
$$j=i-b(b+1)/2$$
处。根据挑战性的问题是确定`b`的值。下标小于等于`i`的元素个数是`i+1`。另一方面，在块`0,...,b`中的元素个数是$(b+1)(b+2)/2$。因此，`b`是满足不等式
$$(b+1)(b+2)/2 \ge i+1$$
的最小整数。我们可以重写这个不等式为：
$$b^2+3b-2i \ge 0.$$
对应的二次方程式为$b^2+3b-2i = 0$有两个解：$b=(-3+\sqrt{9+8i})/2$和$b=(-3-\sqrt{9+8i})/2$。第二个解在我们的应用中没有意义，因为他总是返回一个负数。因此i，我们取的解是$b=(-3+\sqrt{9+8i})/2$。一般的，这个解不是一个整数，但是回到我们的不等式，我们想要的是的最小整数$b$因此$b\ge (-3+\sqrt{9+8i})/2$。这可以简化为：
$$b=\lceil(-3+\sqrt{9+8i})/2\rceil$$

```Java
int i2b(int i){
    double db = (-3.0 + Math.sqrt(9+8*i))/2.0;
    int b = (int)Math.ceil(db);
    return b;
}
```
使用这种方法，`get(i)`和`set(i,x)`方法的实现就很直接了。我们首先计算对应的块`b`和这个块内对应的索引`j`然后执行相应的操作:
```Java
T get(int i){
    int b = i2b(i);
    int j = i - b*(b+1)/2;
    return blocks.get(b)[j];
}
T set(int i,T x){
    int b = i2b(i);
    int j = i - b*(b+1)/2;
    T y = blocks.get(b)[j];
    blocks.get(b)[j] = x;
    return y;
}
```
如果我们使用本章中任意数据结构表示`blocks`链表，那么`get(i)`和`set(i,x)`运行时间都是常量时间。

`add(i,x)`方法现在看起来就很眼熟了。我们首先通过检查块的数量$r$是否满足$r(r+1)/2=n$判断链表是否已满。如果满了，我们调用`grow()`增加一个block。完成后，我们分别向右移动下标为`i,...,n-1`的元素一个位置，为下标为`i`的新元素腾位置：
```Java
void add(int i,T x){
    int r = blocks.size();
    if(r*(r+1)/2 < n+1) grow();
    n++;
    for(int j = n-1; j > i;j--)
        set(j,get(j-1));
    set(i,x);
}
```
`grow()`方法做了我们希望的事情。它加入一个新块：
```Java
void grow(){
    blocks.add(new Array(blocks.size()+1));
}
```
忽略`grow()`操作的开销，一次`add(i,x)`操作的开销由移动元素的开销决定，因此结果是$O(1+n-i)$，就像`ArrayStack`。

`remove(i)`操作类似于`add(i,x)`。它分别向左移动下标为`i+1,...,n`的元素一个位置然后，如果有超过一个空的块，就调用`shrink()`方法只保留一个未使用块，其它都删除了：
```Java
T remove(int i){
    T x = get(i);
    for(int j = i; j < n-1; j++)
       set(j,get(j+1));
    n--;
    int r = blocks.size();
    if((r-2)*(r-1)/2 >= n) shrink();
    return x;
}
```
```Java
void shrink(){
    int r = blocks.size();
    while(r > 0 && (r-2)*(r-1)/2 >= n){
        blocks.remove(blocks.size()-1);
        r--;
    }
}
```
再一次，忽略`shrink()`操作的开销，一个`remove(i)`操作的开销是由移动元素决定的，所以结果是$O(n-i)$。

#### 2.6.1 Growing和Shrinking分析
上面对`add(i,x)`和`remove(i)`的分析没有计算`grow()`和`shrink()`的开销。注意，和`ArrayStack.resize()`操作不同，`grow()`和`shrink()`操作不复制任何数据。它们仅仅是分配或者释放大小为`r`的数组。在某些环境中，这只花费常量时间，然而在另外一些，他可能会需要正比于$r$的时间。

我们注意到，在调用`grow()`或者`shrink()`刚结束时，情况是很清晰的。最后一个块是完全空的，其它所有块是满的。知道至少删除或者增加$r-1$个元素后才会发生另外一次对`grow()`或者`shrink()`调用。因此，即使`grow()`和`shrink()`时间开销是$O(r)$，这个开销可以摊还到至少`r-1`次`add(i,x)`或者`remove(i)`操作上，所以`grow()`和`shrink()`的摊还开销是每个操作$O(1)$。

#### 2.6.2 空间使用
接下来，我们分析`RootishArrayStack`的额外空间使用。具体地说，我们想要统计`RootishArrayStack`中没有被当前某个数组元素用来存放列表元素的全部空间。我们称这些空间为 _浪费空间(waste space)_。

`remove()`操作保证了`RootishArrayStack`永远不会超过两个块是全满的。`RootishArrayStack`用来存放$n$个元素的块个数$r$满足等式：
$$(r-2)(r-1)\le n.$$
再一次，使用二次方程式，我们得到：
$$r \le (3+\sqrt{1+4n}/2)=O(\sqrt{n}).$$
最后两个块的大小是$r$和$r-1$，所以这两个块浪费的空间最多不会超过$2r-1=O(\sqrt{n})$。如果我们把这些块存放到`ArrayStack`中(举个例子)，那么这个链表用来存放那些$r$个块浪费的空间总数也是$O(r) = O(\sqrt{n})$。用来存放$n$和其他统计信息的空间是$O(1)$。因此，`RootishArrayStack`浪费的全部空间总量是就是$O(\sqrt{n})$。

接下来，我们证明对于任意一个数据结构，如果它是从空开始，并且支持一次增加一个元素，那么上述的空间使用是最优的。更精确的说，我们将会展示，在增加$n$个元素期间的某一点，这种数据结构正在浪费的空间总量至少是$\sqrt{n}$(尽管他可能只会在很短的时间内浪费这些空间)。

假设我们从一个空的数据结构开始，我们往这个结构一次加入一个元素共加入$n$个元素。在过程结束后，所有$n$个元素都存入了结构中，并且分布在$r$个内存块中。如果$r\ge \sqrt{n}$，那么这个数据结构必须要使用$r$个指针(或者引用)开追踪这个$r$个块，这些指针就是浪费空间。另一方面，如果$r<\sqrt{n}$，那么，通过鸽笼原理，某些块必须的大小必须至少是$n/r>\sqrt{n}$。考虑这个块第一次被分配时的情况。在他被分配后那一时刻，这个块是空的，因此浪费了$\sqrt{n}$个空间。因此，再插入$n$个元素期间的某一时刻，这个数据结构浪费了$\sqrt{n}$的空间。
### 2.6.3 总结
下面的定理总结了我们对于`RootishArrayStack`这个数据结构的的讨论。
__定理2.5__ `RootishArrayStack`实现了List接口。忽略调用`grow()`和`shrink()`的调用开销，一个`RootishArrayStack`支持这些操作：
* 操作是$O(1)$开销的`get(i)`和`set(i,x)`；以及
* 操作为$O(1+n-i)$开销的`add(i,x)`和`remove(i)`。
进一步的，从一个空`RootishArrayStack`开始，以任意顺序调用`add(i,x)`和`remove(i)`操作$m$次所产生的对`grow()`和`shrink()`调用一共会花费$O(m)$的时间开销。

`RootishArrayStack`用来存放$n$个元素的空间(按照字来计算[<sup id= "3">3</sup>](#3))是$n+O(\sqrt{n})$。
[<sup id="content3">3</sup>](#content3) 回忆下在1.4节讨论过的内存如何被测量。
### 2.6.4 计算平方
对计算模型(models of computation)有所了解的读者可能会注意到上面讨论到的`RootishArrayStack`并没不满足通常的word-RAM计算模型(第1.4节)，因为它要求计算平方数。平方操作通常不认为是基本操作因此经常不是word-RAM模型的一部分。

在这一节，我们会展示开平方操作有更高效的实现。特别的，我们会展示，对于任意一个整数$x$满足$x\in \{0,...,n\}$，那么在经过$O(\sqrt{n})$时间开销的预处理操作并产生两个长度为$O(\sqrt{n})$的数组后，$\lfloor \sqrt{x}\rfloor$可以在常量时间中计算出来。下面的引理向展示了我们可以把计算$x$的平方根这一问题化简为计算相关值$x'$的平方根。

__引理2.3__ 设$x\ge 1$且$x'=x-a$，其中$0\le a \le \sqrt{x}$。那么$\sqrt{x'}\ge \sqrt{x}-1$。
$\text{证明}$ 有如下不等式：
$$\sqrt{x-\sqrt{x}}\ge \sqrt{x}-1.$$
两边都平方得到不等式如下：
$$x-\sqrt{x}\ge x-2\sqrt{x}+1$$
合并计算后：
$$\sqrt{x}\ge 1$$
这对任意$x\ge 1$显然是真的。
让我们先稍微限定一下问题开始，假设$2^r\le x \le 2^{r+1}$，那么$\lfloor \log x\rfloor = r$，例如，$x$是一个整数，它的二进制表示包含了$r+1$个位。我们可以设置$x'=x-(x \bmod 2^{\lfloor r/2 \rfloor})$(译注：$a \bmod b$的结果肯定是小于$b$的，所以，要考虑的就是$b$的取值是否满足引理2.3)。这样$x'$满足引理2.3的条件，所以$\sqrt{x}-\sqrt{x'}\lg 1$。进一步的，$x'$所有低于$\lfloor r/2 \rfloor$的bit位都等于0，所以$x'$只有
$$2^{r+1-\lfloor r/2 \rfloor} \le 4 \cdot 2^{r/2} \le 4\sqrt{x}$$
个可能值。这意味着我们可以使用一个数组`sqrttab`，对于每个可能的$x'$存放了$\lfloor \sqrt{x'}\rfloor$的值。更精确点，我们有：
$$sqrttab[i]=\lfloor \sqrt{i2^{\lfloor r/2 \rfloor}} \rfloor$$
使用这种方式，对于任意$x \in \{i2^{\lfloor r/2 \rfloor},...,(i+1)2^{\lfloor r/2 \rfloor}-1\}$，`sarttab[i]`与$\sqrt{x}$的差是2。换句话说，这个数组的条目$s=sqrttab[x>>\lfloor r/2\rfloor]$可能会等于$\lfloor \sqrt{x} \rfloor,\lfloor \sqrt{x} \rfloor-1,\lfloor \sqrt{x} \rfloor-2$其中之一。通过逐渐递增`s`值到$(s+1)^2\gt x$，我们就可以确认$\lfloor \sqrt{x} \rfloor$的值。
```Java
int sqrt(int x,int r){
    int s = sqrttab[x>>r/2];
    while ((s+1)*(s+1)<=x)s++;//最多两次
    return s;
}
```
注意，这只针对$x\in \{2^r,...,2^{r+1}-1\}$有效果，并且`sqrttab`是特定的表隔只针对于特定的值$r=\lfloor \log x \rfloor$有效。为了克服这个，我们可以计算$\lfloor \log n\rfloor$个不同的`sqrttab`数组，每一个都针对于$\lfloor \log x \rfloor$每个可能值。这些表的大小组成了一个指数序列最大值最多是$4\sqrt{n}$，所以所有表的全部大小是$O(\sqrt{n})$。

然而，事实证明超过一个`sqrttab`数组是没必要的；我们只需要一个针对值$r=\lfloor \log n \rfloor$的`sqrttab`。任何值$x$满足$\log x = r'\lt r$都可以通过对$x$乘一个$2^{r-r'}$升级，并使用等式：
$$\sqrt{2^{r-r'}x}=2^{(r-r')/2}\sqrt{x}$$
$2^{r-r'}x$的值范围位于$\{2^r,...,2^{r+1}-1\}$，这样我们可以在`sqrttab`中查找它的平方根。如下代码按照这个方式使用大小为$2^{16}$的`sqrttab`实现了为所有范围在$\{0,...,2^{30}-1\}$的非负整数$x$计算$\lfloor \sqrt{x}\rfloor$。
```Java
int sqrt(int x){
    int rp = log(x);
    int upgrade = ((r-rp)/2)*2;
    int xp = x << upgrade;//xp的位数为r或者r-1
    int s = sqrtab[xp >> (r/2)]>>(upgrade/2);
    while((s+1)*(s+1) <= x)s++;//最多执行两次
    return s;
}
```
还有一个我们想当然的是如何计算$r'=\lfloor \log x\rfloor$。再一次，我们可以使用一个大小为$2^{r/2}$的数组`logtab`解决这个问题。在这种情况下，代码就很简单了，因为$\lfloor \log x \rfloor$就是$x$二进制表示中为1的最高位下标。这意味着，对于$x \ge 2^{r/2}$，我们可以把$x$向右移动$r/2$个位置作为`logtab`的下标。如下代码使用了大小为$2^{16}$的`logtab`来计算范围位于$\{1,...,2^{32}-1\}$所有$x$的$\lfloor \log x \rfloor$。
```Java
int log(int x){
    if( x >= halfint)
        return 16+logtab[x>>>16];
    return logtab[x];
}
```
最后，为了完整性，我们展示了如下初始化`logtab`和`sqrttab`的代码：
```Java
void inittabs(){
    sqrttab = new int[1 << (r/2)];
    logtab = new int[1 << (r/2)];
    for(int d = 0;d < r/2; d++)
        Arrays.fill(logtab,1<<d,2<<d,d);
    int s = 1<<(r/4);
    for(int i = 0;i < 1<<(r/2);i++){
        if((s+1)*(s+2) <= i << (r/2)) s++;
        sqrttab[i] = s;
    }
}
```
总结一下，`i2b(i)`方法的计算操作可以在word-RAM上面通过使用$O(\sqrt{n})$额外的内存存储`sqrttab`和`logtab`数组来在常量时间实现。这些数组可以在$n$增加或减少2倍时重建，这中重建的开销可以按照在分析`ArrayStack`中`resize()`的开销那样，摊销到导致$n$改变的`add(i,x)`和`remove(i)`数上。

### 2.7 讨论和练习
本章描述的大部分数据结构都是很古老的。在三十年前就有了相关实现。例如，由Knuth描述的`stack`，`queues`和`deques`，很容易泛化为本章描述的`ArrayStack`，`ArrayQueue`和`ArrayDeque`。

Brodnik及其合作者似乎是第一个描述`RootishArrayStack`并证明$\sqrt{n}$下界。它们还提供了另一个数据结构，这个结构对块大小的选择更加复杂，为了避免在`i2b(i)`方法中计算平方根。在他们的模式里，包含`i`的块`b`是$\lfloor \log(i+1) \rfloor$，这个值就是`i+1`的二进制表示中出现1的最高位下标(the index of leading 1 bit in the binary representation of i+1)。某些计算机架构提供了计算整数这一下标的指令。在Java中，`Integer`类提供了一个方法`numberOfLeadingZeros(i)`，我们使用这个方法可以很容易计算$\lfloor \log(i+1) \rfloor$。

与`RootishArrayStack`相关的结构是由Goodrich和Kloss介绍的两级tiered-vector。这个结构支持常量时间的`get(i,x)`和`set(i,x)`以及$O(\sqrt{n})$时间的`add(i,x)`和`remove(i)`。练习2.11中讨论的`RootishArrayStack`经过精心的实现也可以有类似的运行时间。