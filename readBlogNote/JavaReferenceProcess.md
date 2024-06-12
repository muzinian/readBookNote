## JavaGC对引用的处理
内容来自于[这里](https://developers.redhat.com/articles/2021/05/20/shenandoah-garbage-collection-openjdk-16-concurrent-reference-processing#)。文章介绍了 JavaVM/GC 对 Java 规范中各种引用处理的流程，并介绍了 ShenandoahGC 在 OpenJDK 的 Java16 中改进的处理。

### 概述
JVM 在 GC 处理过程中会出现 STW 的情况，而对 GC 对引用(reference 根据文章内容可知，这里指的是 类型是 Reference 类以及子类的对象，下文都统一用 reference)的处理是暂停时间的一个主要贡献者。暂停时间和 reference 处理的关系几乎是线性的：应用程序使用的 reference 越多，对 GC 暂停时间和延迟的影响就越高。关键点在于“使用”，或者说，在每个 GC 周期中，需要处理多少个 reference。如果 reference 对应的被引用者(referent，下文统一用 referent)没有死亡或者跟随 reference 一起死亡，那就不会构成问题。过去，一般会建议如果应用关心延迟，就最好不要使用 soft，weak 和 phantom references 或者 finalizees。而 ShenandoahGC 在 Java16 中通过并发处理 reference，解决了这个问题。

### 引用处理回顾
Java 语言有`java.lang.ref.Reference`类型及其子类`SoftReference`，`WeakReference`，`PhantomReference`和 `FinalReference`类型。对象之间的普通 reference 也叫做 _强引用(strong references)_。每个 reference 指向一个 referent。各种 reference 类型的目的能够引用一个对象但是不会因为由于有这个 reference 而阻止这个对象被回收(这句有点拗口，就是说 reference 指向的 referent 自己还是可能会被回收，而 reference 有自己的处理逻辑)。回收遵循可达性规则，按照可达性的强度下降的顺序，规则大致如下：

* `SoftReference`：一旦它的 referent 没有任何 strong reference 可达，referent 就可以被回收了。回收还收到其他启发式策略影响，比如，soft reference 通常只会在内存压力大的时候才会被回收。
* `WeakReference`：一旦它的 referent 没有任何 strong reference 或者 soft reference 可达，referent 就可以被回收了。只要还没有被回收，这个 referent 仍然可能被访问，此时就会再次变得强可达。
* `FinalReference`：这是 JDK 内部的引用类型。用来实现`Object.finalize()`。任何实现了`finalize()`方法的对象都会成为`FinalReference`的 referent，并且这个`FinalReference`被注册到对应的`ReferenceQueue`中。只要那个对象不再是没有任何 strong reference，soft reference 和 weak references 可达，它会被处理，并且它的`finalize()`会被调用。
* `PhantomReference`：一旦它的 referent 没有任何 strong，soft，weak 或者 final reference 可达，也就是说它就是不可达了，referent 就可以被回收了。这个 referent 永远不能再被访问。这么做是为了避免 referent 在被判定为不可达后被意外的复活。一旦 referent 的可达性被确定，不可达的 referent 被清理（设置为`null`），它对应的 reference 对象进入到对应的`ReferenceQueue`中供应用后续处理。phantom reference 最典型的用法是管理原生内存或者操作系统文件句柄这些资源，GC 无法处理这些资源。

### 传统的引用处理方式
在 JDK16 之前，ShenandoahGC 基本是按照可达性规则处理 reference 的。

* 首先，在并发标记阶段确定所有强可达对象的可达性。只要标记波面(marking wavefront，这是一个 GC 上的习惯用语。考虑 GC 使用三色抽象，在标记阶段，从根开始，遍历整个对象图，将白色对象标记为灰色和黑色。整个过程就好像是波在这个对象图上传播，波面 wavefront 首先到达白色的未标记对象，下文如果遇到这个术语，统一用 wavefront)。到达一个 reference 对象，而这个对象没有一个被标记过的 referent（例如，没有从其他地方强可达的一个对象），它会停在这里，入队这个 reference 到四个 _发现队列(discovery queues)_（对于每个 reference 类型都有一个对应的发现队列）中的一个供后续处理。软引用还有特殊逻辑，GC 可能会基于启发式信息（例如内存压力和/或对象年龄）确定是否将软引用看作强引用。
* 接着，一旦并发（强）标记完成，JVM 暂停应用并启动处理发现队列：
  1. 检查所有`SoftReference`对象，如果 referent 没有强可达（例如，没有被并发标记标记到。这里，如果在第一步软引用被视作强应用，那么这个 referent 就会被认为是强可达的吧，从而不会清理这个 referent），就会清理它，然后这个 `SoftReference` 被放到处理队列。
  2. 检查所有`WeakReference`对象，如果 referent 没有强可达，就会清理它，然后这个 `WeakReference` 被放到处理队列。
  3. 检查所有`FinalReference`对象，如果 referent 没有强可达，这个 `FinalReference` 被放到处理队列。由于后续 GC 可能会调用 referent 的 `finalize()` 方法，暂时不会清理 referent。同时，会从这个已经是不可达的 referent 开始，恢复标记，并且从这个 referent 找到的子图对象也被标记。这是为了避免下一步回收从 finalizees 可达的`PhantomReference`。然而这一步由于发生在 JVM 的 STW 过程中，而理论上这一步的执行时间仅仅根存活对象数据大小有关系，因此，可能会花费很多时间用于标记从 finalizees 开始的子图。
  4. 所有的`PhantomReference`对象被检查。如果 referent 不可达（既没有强可达也没有来自 finalizees 的指向），referent 被清理，`PhantomReference` 被放到处理队列。
*  最后处理队列被加入到 Java 链式表（linked list，不是 array list）供`ReferenceQueue`处理。

由上可知，引用处理对 GC 暂停时间的贡献基本正比于需要处理的 _references 数量 + 新标记的子图大小_。所以，解决 reference 处理的暂停时间问题的方式是 GC 可以并发执行。而 reference 处理的任务可以分成并发标记 referent 及其对应子图和并发处理和清理 reference 两个子任务。

### 并发 reference 标记
ShenandoahGC 根据上述内容简化了可达性模型。它认为不需要五个级别的可达性（strong，soft，weak，final 和 phantom），只需要两个（strongly 可达和 finalizably 可达）。先考虑`WeakReference`，当它的 referent 不是强可达的，就会被清理并入队。当`SoftReference`对象满足启发式规则并且它的 referent 不是强可达了才会被清理。可以看到，对于`SoftReference`来说它要么就按照 strong reference 处理，要么按照 weak reference 处理，而这可以在并发标记前或者期间决定出来。当`FinalReference`对象的 referent 不是强可达的，就会被入队。当`PhantomReference`的 referent 不是强可达且无法从任何`FinalReference`可达时就会被清理并入队。因此，可以看出来，涉及到的可达性只有强可达和 finalizably 可达。

传统实现方法按照下面步骤进行标记以确定可达性：
1. 并发标记期间，构造一个包含所有强可达对象的集合
2. 处理所有只需要这一可达性级别的软/弱/ final 引用
3. 从 finalizees 开始继续标记
4. 处理剩下的需要上一步信息的 Phantom 引用
ShenandoahGC 通过对标记位图（marking bitmap）增加一个 bit 的方式并发判断强可达和 finalizable 可达。而不再是仅用一个 bit 表示有没有标记，通过两个 bit 可以表达所有可能的状态：强可达，finalizably 可达，不可达以及一个内部使用的状态。这样就可以并发标记所有存活对象并确定 strong 和 finalizable 可达性：
1. 使用普通的强波面（strong wavefront）从根开始标记
2. 一旦一个 strong wavefront 遇到一个`SoftReference`根据启发式规则决定应该当作 strong reference 还是 weak reference。如果决定当作强引用，正常标记 referent（标记它的子图为 strong），否则在这里停下 wavefront 并将 soft reference 加入到发现队列。
3. 当一个 strong wavefront 遇到一个`WeakReference`或`PhantomReference`，在这里停下 wavefront 并将 reference 加入到发现队列。
4. 只要 strong wavefront 遇到一个`FinalReference`，标记`FinalReference`为 strong，然后切换到 finalizable wavefront 标记 referent。所有从这里可达的对象都会标记为 finalizable。

至此，我们就并发的标记了所有可达对象并确定了它们是强可达还是 finalizably 可达，入队了所有 reference 对象。接下来就是并发处理 reference。
### 并发处理 reference
ShenandoahGC 是在 final-mark 阶段标记过程中并发建立起可达性并在这个暂停期间为驱逐（evacuation）进行设置。要做两件事：
1. 扫描发现队列并清理所有不可达的 referents。
2. 把“不可达的” reference 对象入队到处理队列供后续 Java 侧处理。
ShenandoahGC 会并发扫描发现队列，检查每个 reference 对象并查看 referent 是否可达。如果一个 reference 是不可达的（可达性规则依赖于上面的描述，对于 soft，weak 和 final reference 必须是强可达的，而 phantom reference 必须是强可达或者 finalizably 可达），清理 referent 并将 reference 入队处理队列。如果 Java 程序在此过程中尝试在我们清理 referent 前访问它，Java 应用还是可以看到本来我们确定为不可达的 referent 的情况，并恢复它。而这会破坏规范并最终可能导致 JVM 崩溃，因为 GC 随后会回收或者覆写那个对象，就导致 Java 程序使用悬空指针（dangling pointer）。ShenandoahGC 在`Reference.get()`方法中插入了一个特殊的 barrier 解决这个问题。如果对象不可达，就返回`null`。
```java
// Pseudocode of Reference.get() intrinsic
T Reference_get() {
  T ref = this.referent;
  return lrb(ref);
}
// Let's return null when the referent is unreachable:

// Pseudocode of Reference.get() for concurrent reference processing
T Reference_get() {
  T ref = this.referent;
  if (isUnreachable(ref) {
    return null;
  }
  return lrb(ref);
}
```
代码中的`lrb`表示的是 ShenandoahGC 的 Load Reference Barriers。具体内容可以参阅[这里](https://developers.redhat.com/blog/2019/06/27/shenandoah-gc-in-jdk-13-part-1-load-reference-barriers)

### 总结
改进后，ShenandoahGC 在对比 Java8 和 Java16 的测试过程中，从 final-mark 阶段中完全移除了 reference 处理，暂停时间从3.8ms 降低到了 0.9ms。整个引用处理用时 9.5ms，但是，由于它不会 STW，就完全不需要关心它。