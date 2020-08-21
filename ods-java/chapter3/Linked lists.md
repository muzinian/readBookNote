## Linked Lists
本章，我们继续学习`List`接口的实现，这一次使用基于指针而不再是数组的数据结构。本章的数据结构是由包含了链表元素的节点组成的。使用引用(指针)，节点被链接到一个序列中。我们首先研究单链表，可以以常量时间实现栈和(FIFO)队列操作，接着我们转向双链链表，可以以常量时间实现双端队列(`Deque`)操作。

基于指针链接的链表相比于基于数组的List接口实现有优点也有缺点。主要的缺点是我们无法使用`get(i)`或者`set(i,x)`在常量时间内访问任意元素。相反，我们必须要遍历整个`list`，一次一个元素，知道我们到达第i个元素。主要的优点是它们具有更高的动态性：有任意一个链表节点`u`的引用，我们可以在常量时间内删除`u`或者在`u`的旁边插入一个元素。无论`u`在`list`中哪个位置。

### 3.1 SLList:单链链表
一个`SLList`(一个单链表`singly-linked list`)由`Nodes`组成的序列。每个节点`u`存储了数据值`u.x`和指向序列下一个节点的引用`u.next`。对于序列中最后一个节点`w`，`w.next = null`。
```Java
class Node{
    T x;
    Node next;
}
```
为了效率，`SLList`使用变量`head`和`tail`跟踪序列中开始和最后的节点，以及整数`n`记录序列的长度。
```Java
Node head;
Node tail;
int n;
```
对一个`SSList`的栈和队列操作序列解释如图3.1。

![figure3.1.png "在`SSList`上执行`Queue`(`add()`和`remove()`)和`Stack`(`push(x)`和`pop()`)操作序列"](figure3.1.png "在`SSList`上执行`Queue`(`add()`和`remove()`)和`Stack`(`push(x)`和`pop()`)操作序列")

`SSlist`通过在序列头部添加和删除元素可以高效的实现`Stack`的`push()`和`pop()`操作。`push`操作就是简单的创建一个节点`u`包含数据`x`，设置`u.next`指向链表旧的头节点然后设置`u`为链表新的头节点。最后，对`n`加一，因为`SSList`的长度多了一个元素：
```Java
T push(T x){
    Node u = new Node();
    u.x = x;
    u.next = head;
    head = u;
    if(n == 0)
      tail = u;
    n++;
    return x;
}
```
`pop()`操作在检查完`SSList`非空后，通过设置`head = head.next`删除头节点并对`n`减一。需要考虑的特殊情况是，最后一个元素被删除了，此时需要设置`tail`为`null`:
```Java
T pop(){
    if(n==0)return null;
    T x = head.x;
    head = head.next;
    if(--n = 0)tail = null;
    return x;
}
```
显然，`push(x)`和`pop()`操作运行时间为$O(1)$。
#### 3.1.1 队列操作
`SSList`还可以在常量时间里实现FIFO队列的`add(x)`和`remove()`操作。`remove()`是从链表的头开始的，它和`pop()`操作一样：
```Java
T remove(){
    if(n==0) return null;
    T x = head.x;
    head = head.next;
    if(--n = 0) tail = null
    return x;
}
```
另一方面，`add(x)`，是在链表尾端完成的。在大多数情况下，是通过设置`tail.next = u`完成，这里`u`是新创建的节点包含值`x`。然而，当`n==0`时，此时`tail==head==null`，在这种情况下，`tail`和`head`都被设置为`u`。
```Java
boolean add(T x){
    Node u = new Node();
    u.x = x;
    if(n == 0){
        head = u;
    }else{
        tail.next = u;
    }
    tail = u;
    n++;
    return true;
}
```
显然，`add(x)`和`remove()`时间开销是常量。

#### 3.1.2 总结
下面的定理总结了`SSList`的性能：
__定理3.1__ `SSList`实现了`Stack`和(FIFO)`Queue`接口。`push(x)`，`pop()`，`add(x)`和`remove()`操作的运行时间都是$O(1)$。

`SSList`基本上实现了全部的`Deque`操作。唯一缺少的操作是从`SSList`尾部移除元素。从`SSList`尾部移除元素很麻烦，因为它需要更新`tail`的值，使得它指向`tail`在`SSList`中的前驱节点`w`；这个节点满足`w.next == tail`。不幸的是，得到`w`的唯一方法是从`head`开始遍历整个`SSList`，这需要`n-2`步。

### 3.2 DLList:双向链表
一个DLList(双向链表)和SLList类似，除了在DLList的节点`u`有指向在它之后`u.next`的引用和在它之前的节点`u.prev`的引用。
```Java
class Node{
    Tx;
    Node prev,next;
}
```
在实现SLList时，我们看到总会有几种特殊情况需要考虑。例如，从SLList删除最后一个元素，或者对空SLList加入一个元素都需要仔细确保`head`和`tail`被正确的更新。在DLList中，这些特殊情况的数量增加了相当多。在DLList中考虑这些特殊情况最干净的可能方式是引入`dummy`节点。这个节点不包含任何数据，只是扮演一个占位符，这样就没有特殊节点了；每个节点都会有`next`和`prev`，这里`dummy`扮演了链表最后一个节点的后继以及第一个节点的前驱。使用这种方式，列表的节点以双向的方式链接成一个圆，如图3.2。

![figure3.2.png "包含了a,b,c,d,e的DLList"](figure3.2.png "包含了a,b,c,d,e的DLList")

```Java
int n;
Node dummy;
DLList(){
    dummy = new Node();
    dummy.next = dummy;
    dummy.prev = dummy;
    n = 0;
}
```
在DLList中找到特定下标的节点很容易；我们要么从链表的头开始(`dummy.next`)向前查找，或者从列表尾部开始(`dummy.prev`)向后查找。这使得我们可以在$O(1+\min\{i,n-i\})$的时间内找到第i个元素。
```Java
Node getNode(int i){
    Node p  = null;
    if(i < n/2){
        p = dummy.next;
        for(int j = 0;j < i; j++){
            p = p.next;
        }
    }else{
        p = dummy;
        for(int j = n; j > i; j--){
            p  = p.prev;
        }
    }
    return p;
}
```
`get(i)`和`set(i,x)`操作现在也很容易了。我们先找到第i个节点，然后获取或者设置它的`x`值：
```Java
T get(int i){
    return getNode(i).x;
}

T set(int i,T x){
    Node u = getNode(i);
    T y = u.x;
    u.x = x;
    return y;
}
```
这些操作的运行时间由它用来找第i个元素的时间所决定，因此是$O(1+\min\{i,n-i\})$。

### 3.2.1 新增和删除
如果我们有在DLList内一个节点`w`的引用并且我们想要在`w`前插入节点`u`，那么就只需要设置`u.next=w,u.prev=w.prev`，然后调整`u.prev.next`和`u.next.prev`。(参见图3.3)由于有`dummy`节点，我们就不需要担心`w.prev`或`w.next`不存在。
```Java
Node addBefore(Node w,T x){
    Node u = new Node();
    u.x = x;
    u.prev = w.prev;
    u.next = w;
    u.next.prev = u;
    u.prev.next = u;
    n++;
    return u;
}
```
![figure3.3.png "在DLList中的w节点前增加一个节点u"](figure3.3.png "在DLList中的w节点前增加一个节点u")

现在，链表操作`add(i,x)`就很容易实现了。我们找到DLList内的第$i$个节点，然后就在他前面插入包含`x`的新节点`u`。
```Java
void add(int i,T x){
    addBefore(getNode(i),x);
}
```
`add(i,x)`运行时间中唯一不是常量的部分就是用在找第$i$个节点(`getNode(i)`)所花费的时间。因此，`add(i,x)`运行时间$O(1+\min\{i,n-i\})$。

从DLList中删除一个节点`w`是很容易的。我们只需要调整`w.next`和`w.prev`这两个指针，使得可以跳过`w`。再一次，使用`dummy`节点消除了对特殊场景的考虑：
```Java
void remove(Node w){
    w.prev.next =w.next;
    w.next.prev = w.prev;
    n--;
}
```
现在，`remove(i)`操作就很简单了。我们可以找到索引为`i`的节点，然后删除它：
```Java
T remove(int i){
    Node w = getNode(i);
    remove(w);
    return w.x;
}
```
再一次，这个操作唯一耗时的部分是使用`getNode(i)`找到第$i$个节点。，所以`remove(i)`的运行时间是$O(1+\min\{i,n-i\})$。

### 3.2.2 总结
下面的定理总结了DLList的性能：
__定理3.2__ `DLList`实现了`List`接口。其中，`get(i)`，`set(i,x)`，`add(i,x)`和`remove(i)`操作的运行时间是$O(1+\min\{i,n-i\})$。

值得注意的是，如果我们忽略`getNode(i)`操作的开销，对DLList的所有操作都只花费常量时间。因此，对DDList的所有操作有开销的就是找到相关节点。

一旦我们有了相关节点，增加，删除和访问这个节点的数据都只用花费常量时间。

这和第二章中基于数组的链表实现形成了鲜明的对比；在那些实现中，相关的数组元素可以在常量时间找到。但是，增加和删除要求在数组中移动元素，并且，通常情况下，这会花费非常量时间。

由于这个原因，链式`list`结构非常适合那些可以通过外部方式获取链表节点引用的应用。一个例子就是Java集合框架中的`LinkedHashSet`数据结构，一系列元素存放在一个双端链表中，双端链表的节点存放在一个哈希表中(将在第5章介绍)。当从`LinkedHashSet`删除元素时，哈希表用来在常量时间查找相关链表节点，然后删除链表节点(所以也在常量时间内)。