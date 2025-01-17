[toc]
## lambda演算-1
__本文主要内容都脱胎于[这里](https://opendsa.cs.vt.edu/ODSA/Books/PL/html/index.html#lambda-calculus)的第三章的前4节。如果有什么错误都是我没有理解到位。__
### lambda演算的语法和语义
lambda演算，也叫$\lambda$-演算(lambda-calculus，$\lambda$-calculus)，是邱奇在研究函数的可计算性时创建的。在可计算理论中，lambda演算是一个简洁而有力的模型，可以认为lambda演算是一个函数式编程语言，它和目前所有大规模使用的图灵完备语言的能力是一样。使用lambda演算的程序叫做lambda表达式(λexp)它的语法的BNF格式如下：
```BNF
<λexp> ::= <var>
        | λ<var> . <λexp>
        |(<λexp> <λexp>)
```
上述BNF语法揭示了lambda演算的三种形式：
1. 一个 _变量(variable)_(第一个产生式定义的)，通常是单个字母或者带下标的字母表示一个变量。
2. 一个 _函数抽象(function abstraction)_(第二个产生式定义的)，这类λ表达式也叫做lambda抽象(lambda abstraction)，对应于函数定义。它包含两部分：函数的形参(由第二个生成式可以看出来，只能有且仅有一个形式参数)和函数体。二者以`.`连接。注意，根据第二个生成式可以看出来，lambda演算定义的函数没有名字，都是匿名函数(注意，这里就引入了一个问题，使用lambda演算如何定义递归函数，具体内容可以看3.9章)。
3. 一个 _应用(application)_(第三个产生式定义的)，这类λ表达式对应一个函数调用(function call,application,invocation)，它也由两部分组成：被调用的函数，跟着传入到这个函数的实参。例如，`(f x)`就是对`f`的应用，传入的实参就是`x`。注意，函数式lambda演算中唯一的值，所以这里的`f`和`x`都是函数(注：我只在这里看到了这种说法，对lambda演算的值做了约束即必须是函数，别的地方没有看到，或者是我没有看仔细)。

注意，lambda演算中，括号的使用是包裹住函数和它的实参的。而且，根据上述的语法，lambda演算中函数调用的括号不仅不是可有可无，而且是有着清晰的规则(也即，只能是在能成功应用第三个生成式的时候括号才是合法的)，并不能随意使用。

尽管lambda演算的语法如此简洁，但是它的语法可以表达出来所有可能的计算，参考[邱奇图灵论题](https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis)。

注意，在lambda演算中所有的程序都是表达式，也就是说，程序会被估值(evaluate)从而得到值。lambda演算不包含任何语句，因此它不含任何语句，也就是没有副作用的命令，比如通过赋值语句修改一块内的内容或者通过`print`语句将一个字符串发送给标准输出流都不存在，因此，它是一个纯函数式的语言(purely functional language)。

1. lambda演算的 _变量_ 是其他lambda表达式的占位符。它用来表示一个已知或者未知的值。在lambda演算中，每个变量在整个程序执行期间只能绑定到一个值上。这与当前大多数语言不同，后者可以通过赋值语句多次修改变量的值。在lambda演算中，变量更像是其他语言中的命名常量，并且，由于在lambda演算中函数是唯一的值，因此，所有变量都是函数值的占位符。
2. lambda演算中的 _lambda抽象_ 是一个函数定义，也即，这个表达式定义了一个函数。在lambda演算中所有函数都是匿名且有且仅有一个参数，因此，定义一个函数所需要做的事情就是确定参数的名字(第二个生成式中，跟在`λ<var>`中`λ`后的`<var>`变量)和函数体(根据第二个生成式可知，也是一个lambda表达式)。
3. 在lambda演算中 _函数应用_ 是一个函数调用，也即是一个用一个实参调用一个函数的表达式。在函数应用中，第一部分要么是一个变量，要么是一个最终可以估值为函数的lambda表达式，它是一个lambda抽象，在定义后被立即执行。
下面是一些例子：

序号|λ表达式|语义|JS实现|备注
-|-|-|-|-
1|x|变量x|x|一个变量
2|λx.x|返回自身的函数(参数名是x)(identity function)|`function (x) { return x; }`|恒等函数
3|λy.y|返回自身的函数(参数名是y)(identity function)|`function (y) { return y; }`|恒等函数
4|λx.y|返回`y`的常量函数(参数名是x)|`function (x) { return y; }`|常量函数
5|λz.y|返回`y`的常量函数(参数名是z)|`function (z) { return y; }`|常量函数
6|λy.x|返回`x`的常量函数(参数名是y)|`function (y) { return x; }`|常量函数
7|λx.λy.y|参数为`x`的函数返回了一个函数，返回的函数参数是`y`，并返回`y`(identity function)|`function (x) { return function (y) { return y }; }`|对一个有两个形参(`x`,`y`)的函数的柯里化
8|(x y)|使用实参`y`调用函数`x`|`x(y)`|函数调用
9|(λx.x y)|应用了实参`y`的恒等函数(identity function)|`(function (x) { return x ; })(y)`|立即执行恒等函数
10|(λz.x y)|应用了实参`y`的常量函数(参数名是`z`，返回了`x`)|`(function (z) {return x;})(y)`|立即执行常量函数
11|λx.(x y)|定义了一个形参名是`x`的函数，函数体是返回的是使用实参`y`调用函数`x`得到的值|`function (x) { return x(y); }`|高阶函数
12|(λx.λy.y z)|对例子7应用了参数z|`(function (x) { return function (y) {return y;} })(z)`|立即执行函数也同时是高阶函数，返回了标识符函数
13|((λx.λy.y u) v)|对例子7应用了参数u和v|`(function (x) {return function (y){return y;} })(u)(v)`|立即执行函数也同时是高阶函数，最终返回了v
### 自由变量和绑定变量
在lambda演算和其他编程语言中，有两类变量出现(variable occurrences)：变量声明(variable declaration)和变量使用(variable use)，以下面的JavaScript代码为例：
```JS
function (x) {
    return x + y;
}
```
这里，变量`x`出现了两次，第一次出现在参数声明，这是整个程序中第一次引入这个变量的地方。`x`第二次出现的地方是对变量`x`的使用。每个变量的声明为这个变量定义了作用域(scope)，它表示了这个变量被定义以及可用的位置。上面的例子中，变量`x`的作用域就是函数体内部。当对变量的使用出现在这个变量的作用域内时，我们就称前者绑定到(is bound to)了后者，而后者就是变量的绑定出现(binding occurrence,注：后续都用英文)。因此，在上面的例子中，括号内`x`的定义是这个变量的binding occurrence，而下一行对`x`的使用绑定到了(is bound to)这个binding occurrence。如果一个变量没有被绑定(bound)就称为自由的(free)。上面的例子中，`y`在函数体内是自由的，因为这个函数不包含任何`y`的binding occurrence。

程序中可能出现多次`x`的声明，因此就不能对每个变量的使用绑定到的哪个变量声明出现任何歧义，因此编程语言定义了绑定模式(binding scheme)，跟大多数编程语言一样，lambda演算使用的是静态绑定(static binding)(也叫做静态作用域(static scoping)或者词法作用域(lexical scoping),注：还有一个叫做动态作用域(dynamic scoping),现代编程语言很少应用动态作用域)，它意味着每个变量使用都绑定到包含这个变量使用的最小的lambda抽象的同名变量声明(注：词法作用域的一个优点是并不需要真正执行代码，只需要阅读代码就可以确定变量的作用域，这对于写parser很方便)。
考虑这个lambda表达式`λy.(λx.x(y x))`，这是一个lambda抽象，变量是`y`，函数体是对恒等函数的应用，参数是`(y x)`。因此第一个λ后的`y`是变量`y`的binding occurrence。这个声明的作用域是`(λx.x (y x))`，意味着这里面的`y`的occurrence绑定到了最外层`y`的binding occurrence。而在第二个λ后出现的`x`，是`x`这个变量的binding occurrence，而它的作用域仅仅在`λx.x`中`.`后的表达式中。因此`(y x)`这个表达式中的`x`是自由的：这个对`x`的使用不属于任何`x`声明的作用域。因此，我们可以看到，在同一个lambda表达式中，变量既可以是自由的又可以是绑定的，因此，询问一个变量在一个lambda表达式中是自由还是绑定的可能会让人困惑，更好的方式是询问对于一个变量的每次特定的occurrence是绑定还是自由的。请记住，一个binding occurrence永远不是自由的，因为它的作用就是定义一个新变量。

对于在一个lambda表达式$E$中我们说一个变量`x`是出现自由(occur free)的，当且仅当：
1. $E$是一个变量且$E$等于`x`，或者
2. $E$形如$(E_1,E_2)$且`x`在$E_1$或$E_2$中是自由的(或者在二者中都是自由的)，或者
3. $E$形如$\lambda y.E'$，这里`y`不同于`x`，且`x`在$E'$中是自由的。
可以发现，规则2和3就是lambda演算中出现的递归的镜像。下表是这些定义的各个case的例子：

$E$|满足case|`x`在$E$中是否出现自由|解释
-|-|-|-
`x`|1|是|`x`在$E$出现并等于它，而且$E$中不包含任何binding occurrence(没有$\lambda$)
`y`|1|否|`x`不在$E$中，因此在$E$中它没法出现自由
`(x y)`|2|是|`x`出现在了函数应用的第一个部分并递归应用case 1
`(y x)`|2|是|`x`出现在了函数应用的第二个部分并递归应用case 1
`(y z)`|2|否|`x`在了函数应用的第一和第二个部分都没出现自由并两次递归应用case 1
`λz.x`|3|是|`x`不同于这个lambda抽象的参数`z`，并且在lambda抽象的函数体内是出现自由的(递归应用case 1)
`λz.z`|3|否|`x`不同于这个lambda抽象的参数`z`，并且在lambda抽象的函数体内没有出现，也就没有所谓的自由了
`λz.λx.x`|3|否|`x`不同于这个lambda抽象的参数`z`，但是在这个lambda抽象中不是出现自由的(递归应用case 3)
`λx.y`或者`λx.x`|3|否|`x`等于lambda抽象$E$的参数，`x`在$E$中不能是自由的，因为`x`在$E$的函数体内的任何free occurrence都将由于开头`x`的binding occurrence而在$E$中变成绑定的
### $\alpha$-变换($\alpha$-conversion)
当`x`的variable occurrence绑定到一个binding occurrence，`x`的意义依赖于这一事实：它出现在函数的内部并且是这个函数形参的名字。因此，`x`的值是完全由在函数被调用的时候传入的实参所决定的，并且等于这个实参。换句话说，当variable occurrence被绑定一个给定的表达式，它的含义在这表达式良好定义的(well defined)。例如有下列JavaScript代码：
```javascript
function f(x){
    return 2+x-y*x; 
}
```
当我们调用这个函数时，`f(12)`，`x`的含义是无歧义的，它的值是12。而`y`在函数`f`内不是良好定义的(not well-defined)，因为`y`的occurrence在`f`的函数体内是自由的。这里，我们可以从语义到角度上看到，自由和绑定变量关键区别。对于函数参数名字的选择是任意的(注；当然一般来说，会选择一个简单且有意义的名字)。比如：
```javascript
function f(t){
    return 2+t-y*t; 
}
//或者
function f(time){
    return 2+time-y*time;
}
```
关键点在于这个三个函数是恒等的。它们有同样的意义。绑定变量仅仅是一个值的占位符，只要在函数体内一致的使用占位符，那么它叫什么就没有关系。这种改写函数参数名字而不影响函数含义的规则就是lambda演算的$\alpha$-转换($\alpha$-conversion):
>$\alpha$-转换是重命名一个函数抽象的形参的过程。

考虑`λx.x`恒等函数，我们可以通过 $\alpha$- 转换使用`y`替换`x`产生一个语义上相同的恒等函数`λy.y`。注意，$\alpha$- 转换在应用过程中必须不能改变函数的意义。具体的说，函数内所有的自由变量在 $\alpha$- 转换之后必须依旧保持自由。这意味着在 $\alpha$- 转换一个lambda抽象时，我们不能仅仅时随便选择任意一个变量做替换。

考虑`(λx.y x)`这个表达式，这是一个函数应用，将`x`(一个自由变量，注意这不是常量函数的形参，常量函数的形参是一个绑定变量)传入到一个常量函数，这个函数返回`y`(一个自由变量)，我们不能对这个表达式做 $\alpha$- 转换，因为它是一个函数应用而不是函数抽象，$\alpha$- 转换是应用于函数抽象的。所以，我们可以对这个函数应用的第一部分进行 $\alpha$- 转换，因为它是一个函数抽象。假设我们用`z`来做 $\alpha$- 转换得到`(λz.y x)`，这没有修改第一部分函数抽象的含义，从而整个lambda表达式(函数应用)的结果没有改变，最终结果是`y`。

但是，我们如果我们用`y`来做 $\alpha$- 转换得到`(λy.y x)`(注：这里按照原文的意思，应该是分成了两部，第一，先对函数应用的第一部分的函数抽象做转换，然后组合在一起，而不是直接对函数应用做转换)，这就改变了函数应用的意义，最终的结果将不是返回`y`而是`x`。这是因为，在原始的应用中，`x`(函数的实参)和`y`(函数体)都是自由的(注：只有在第一部分中函数抽象中，`x`是绑定的，但是它的作用域仅限于函数体中，也就是`y`)，因此在这个应用中它们的意义是没有指定的，可能在更大的上下文中(也许这个应用是位于某一个更大的程序的内部中)，`x`和`y`绑定到了不同的值，因此二者的含义不同并无法互换。这里我们犯的错误就是在对`λx.y`进行 $\alpha$- 转换时，将`y`从自由的变成了绑定的了，我们称`y`经历了变量捕获(variable capture)或者说它被捕获了。由于变量捕获改变了lambda表达式的含义，所以我们在进行 $\alpha$- 转换时要遵循：
>当对一个lambda表达式进行 $\alpha$- 转换时，总是选择一个新的变量，也就是说，一个没有出现在正被 $\alpha$- 转换的函数体内的变量

$\alpha$- 转换用一个完全的新的名字(为了避免变量捕获)简单的替换了函数参数的名字。$\alpha$- 转换在后面替换过程中很有用。