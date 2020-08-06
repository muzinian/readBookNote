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
### 3.1.1 队列操作
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

### 3.1.2 总结
下面的定理总结了`SSList`的性能：
__定理3.1__ `SSList`实现了`Stack`和(FIFO)`Queue`接口。`push(x)`，`pop()`，`add(x)`和`remove()`操作的运行时间都是$O(1)$。

`SSList`基本上实现了全部的`Deque`操作。唯一缺少的操作是从`SSList`尾部移除元素。从`SSList`尾部移除元素很麻烦，因为它需要更新`tail`的值，使得它指向`tail`在`SSList`中的前驱节点`w`；这个节点满足`w.next == tail`。不幸的是，得到`w`的唯一方法是从`head`开始遍历整个`SSList`，这需要`n-2`步。