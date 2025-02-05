[文章](https://robcasloz.github.io/blog/2022/05/24/a-friendlier-visualization-of-javas-jit-compiler-based-on-control-flow.html)

C2 是 OpenJDK 的主要 JIT 编译器，它使用 程序依赖图（ [program dependence graph PDG](https://dl.acm.org/doi/10.1145/24039.24041) ）表示编译后的程序。在 PDG 中，节点表示操作，边表示操作之间的依赖。 PDG 的显式特点是统一了数据和控制，节点既表示普通的代数/逻辑操作也表示像是条件跳转或者循环这样的控制操作。 C2 中使用的 PDG 是 [sea-of-node](https://dl.acm.org/doi/10.1145/202530.202534) 。 sea-of-node 表示统一的特性使得很多优化容易实现，但是缺点就是会快速膨胀， 即使对于简单的程序来说也是如此。

在编译器中，最常见的用来程序表示的可能就是控制流图（ control-flow graph CFG ）。在 CFG 中，节点对于基本块（一系列总是一起执行的操作序列），边对应这些基本块之间的控制跳转。和 PDG 不同， CFG 显式的给每个操作赋值一个基本块，然后在每个基本块内排序这些操作。这会增加几种编译器优化的复杂度，但是他产出了一种结构化的，顺序的程序视图，使得程序易于理解和调试。而且，相对于 PDG 这个很难理解的高级编译器课程中的概念， CFG 要简单很多。

作者在 [Ideal Graph Visualizer(IGV)](https://github.com/openjdk/jdk/tree/master/src/utils/IdealGraphVisualizer)提供了一个视图，可以将 C2 的 PDG 转换为 CFG 。这个视图功能会计算它自己的“ trail schedule ”，包括将操作分配给基本块（ global schedule ）以及在每个基本块内对操作进行排序（ local schedule ）。这个 trail schedule 会在 C2 的 actual schedule 变得可用（当 C2 最后的编译阶段期间变得可用 ）时被 actual schedule 替换。