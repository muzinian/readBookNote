在查找 Java 虚拟线程是 stackless 还是 stackful 的时候读到这[文章](hhttps://mail.openjdk.org/pipermail/loom-dev/2019-November/000876.html)，没想到还有其他额外的知识内容。原邮件是讨论 Loom 项目中 Java Virtual Thread 命名的邮件，有意思的是第一封邮件作者（ Ron Pressler ） 讨论 Java Virtual Thread 时关于 Java Virtual Thread ， continuation 和 coroutine 的区别。以及记录如下：

首先 Ron 解释了 fiber 这个概念相对于 virtual thread 来说对于用户可能增加了学习成本，后者命名表示 virtual thread 就是 thread ，而 fiber 似乎看起来是一个新概念。并且否定了 lightweight thread 和 user-mode thread ，毕竟还有可能有 lighter-weight thread ，以及这些不一定是在 user-mode 实现的。而 virtual thread 之于 thread 类比于 virtual memory 之于 physical memory 。

然后 Cay Horstmann 认为 virtual 这个词可能暗含着增加开销，比如 virtual memory 导致的换页和缓存 miss 。 virtual function 相对于普通函数增加的调用开销。但是 Brian Goetz 说到 virtual thread 的开销正好可以类比于 virtual memory 或者 virtual function 的开销。因为 mount/unmonut 一个虚拟线程到一个物理线程上/从一个物理线程上也会要求栈帧的拷贝，这与构建和拆毁页表精确相似。 Cay 认为提醒程序员 fiber/virtual thread 并不是没有开销很有用。因为大多数程序员认为“ fiber=better thread ”，而“ virtual thread ”也不能解决这个问题。当大量的 OS 线程都是阻塞的，没有人愿意为此付出内存开销，但是“ fiber ”在执行阻塞调用的时候会“ parked ”，所以付出复制调用栈帧的代价可以接受。 Ron 认为在那个时间点（虚拟线程还在开发），涉及到的算法还在开发，所以讨论开下为时过早。而且即使在某些对性能超级敏感的场景存在可观察到开销，也不会是复制栈帧导致的。而且他也认为使用偶然的实现特性作为名字不合适，因为它很可能被改变。 virtual thread 通常建议在经常阻塞的任务中使用，无论是 IO 还是线程间通信/同步导致的阻塞。而这个名字突出的是线程是由运行时管理而不是直接映射到内核实现。 

Volkan Yazıcı 提出来为什么不直接叫这个功能 corountine 。涉及到这个[邮件](https://mail.openjdk.org/pipermail/loom-dev/2018-September/000141.html)。
Ron 在 volkan 提到的邮件中说到：
   >"more recently [coroutine] has gained the connotation (not in academic literature but in language implementations) of being a syntactic construct, rather than a purely dynamic one"
    
他提出了两个问题：
1. volkan 希望 Ron 解释下什么是 purely dynamic one 。因为 volkan 注意到 C++20 和其他提供了 coroutine 的语言中要求显式的语法糖标记可挂起和可恢复的子例程（ suspendable-and-resumeable subroutines ）。 volkan 认为， Loom 项目除了没有显式要求，提供了几乎相同的东西。而且， Loom 还有自己的显式要求（比如，子例程必须要包裹在“ virtual thread ”中）， volkan 认为这和 coroutine 是类似的。
2. 他还认为，从 Melvin Conway 对 coroutine 的定义来看， coroutine 基本上就是包含支持挂起和恢复的子例程。所以，即使在 coroutine 被现代修改了，但是 volkan 认为可以用它。

重点来了， Ron 接着表明了，不管 coroutine 的常见定义有哪些， Java 的 virtual thread 都不是 coroutine 。 Ron 说明了 coroutine 是 one-shot （ non-reentrant ）， delimited continuations ，并且（至少曾经是）它们有时候被用来指代 symmetric one-shot delimited coroutine  ，不过，现在也用来指代 assymmetric one-shot delimited continuations 。而 Loom virtual thread 底层使用的 continuation 是 multi-prompt ， one-shot （ 但是可能可以 cloneable ）， stackful ， assymmetric ， delimited continuations （ continuation 有很多组合特点（ in many flavors ））。可以称这些 continuations 为 coroutine ，尽管现在 coroutine 又了一个新的含义，即将它等同于 continuation 的特殊实现，作为语言中的特殊语法构造（ 例如 C# 的 async 或者 Kotlin 的 suspend ）。但是，不论怎么说， coroutines 总指的是 continuations （ Go 的 goroutines 到指的是线程）。 Ron 认为，线程是 continuation 加调度器，或者说是调度的 continuations （ scheduled continuations ），因此， Loom 的 user-mode thread 是 thread 。因此，不管怎么说，对于 Loom 的调度实体（ scheduled entities ）来说，不适合叫 coroutine （虽然可以叫 Loom 的continuation 为 coroutine ）。

接着， Ron 回答了 volkan 的两个问题。
1. Ron 认为显式的语法标识不是小问题。它是有那种类型的 continuation 无法统一 suspend 和 blocking 这两个概念的原因，即使二者之间的唯一的区别是 suspension 是由操作系统还是语言实现的。例如， 像 JavaScript 这样的就需要两套 IO APIs ，一个 blocking 的一个 suspending （或者说异步的），即使这两个在语言中可以有同样的标识。而像 C# ， Rust 和 Kotlin 这样的语言，甚至在类型系统中区分出了两个。在 Java 中， suspending 一个 continuation 不要求任何不同的类型或者任何语法表示这个操作是由内核还是语言执行是本质功能。在语法上分开一个概念的语言是基于她的实现，但是 Java （还有 Erlang ， Scheme 和 Go ）不是。 Loom 的 virtual thread __是__ 线程；因此在没有在语法上区分这二者的时候，更应该关注抽象而不是实现。
2. Loom 的 threads 不是 coroutines ，而是 coroutines + scheduler ，而多年来，这种结构的常见名字是 thread 。

volkan 希望 Ron 解释一下： one-shot ， non-reentrant ， delimited ， symmetric/assymmetric ， multi-prompt ， cloneable ， stackful 。 Ron 在下一个邮件解释了（ Ron 认为知道 continuations 的各种特点对于理解 Loom 的 continuations 不必须，更不用说理解线程了 ）：
* assymmetric ：当 continuation suspend 或者 yield ，执行返回到了 caller （调用了 Continuation.run() 的方法 ）。 Symmetric 的 cotinuations 没有 caller 的标记。因此，当它们 yield 的时候，需要指定 execution 到其他 continuation 。但是 symmetric 和 asymmetric 之间不存在谁比谁强，并且它们可以互相模拟。
* stackful ：这样的 continuation 可以在任意栈深度被 suspend 。而对于 stackless 的 continuation （比如 C# ），摇在同一个子例程处（在 delimited context 开始的地方）被 suspend 。
* delimited ：这样的 continuation 会捕获从某些特定调用开始的执行上下文（ execution context ）（对于 Java 来说，就是某个 runnable 的 body ），而不是整个执行状态向上直到 main() 方法。 delimited 线对于 undelimited 强大很多，后者被认为没有实际使用价值。
* multi-prompt ：这样的 continuation 可以嵌套，在调用栈的任何位置，任何外围的 continuation 都可以被挂起。这类似`try/catch`块嵌套，并且在抛出某一个类型的异常时候，会 unwind 这个栈一直到能够处理这个异常的嵌套最深的`catch`而不仅仅是嵌套最深的`catch`。嵌套 continuation 的一个例子是在 virtual thread 内使用类 Python 的生成器。这个生成器代码可以执行一个阻塞 IO 调用，而这会 suspend 外围线程 continuation ，而不仅仅是生成器。
* one-shot/non-reentrant ：每次继续一个 suspended continuation 时它的状态都是可变的，并且我么你无法从同一个 suspension 状态继续它多次（例如，我们不能回到过去）。这和可重入（ reentrant ） continuation 不同，每次我们 suspend 它们，都会返回一个新的不可变 continuation 对象，代表了一个特定的 suspension 点。例如， continuation 是时间上的单个点，并且每次我们继续它都回到那个状态。 reentrant continuation 比 non-reentrant continuation 更强大，它们可以完成 one-shot continuation 不能完成的事情。
* cloneable ：如果可以克隆一个 one-shot continuation ，就可以提供和 reentrant continuation 同样能力。即使每次我们继续这个 continuation ，它都是可变的，我们可以在继续它之前通过复制创建一个那个时间点的快照，然后后面就可以返回到这里。