1. 数组添加类型 `Array[Type](total)`
2. 使用下标访问数组的方式是`greetStrings(0)`，跟通常使用中括号不同，scala的访问方式类似方法调用
3. scala没有操作符重载，他没有通常意义上的操作符。而`+,-,*,/`这类符号可以作为方法名称。因此对于`1+2`,你实际上是在`Int`对象`1`上调用它的方法`+`并传入了参数`2`。所以，你可以把`1+2`替换为方法调用语法`(1).+(2)`。
4. 由上，可以知道scala使用小括号访问数组了。scala中特例比java少。数组在scala中就是简单的类。当对变量使用圆括号围绕一到多个参数时，Scala会把这个代码转换为使用变量调用一个名字叫`apply`的方法。所以`greetStrings(i)`等价于`greetStrings.apply(i)`。因此，访问数组的元素对于scala来说就是一个方法调用。这个原则并不仅仅限制于数组：对一个对象应用圆括号内加一些参数(any application of an object to some arguments in parentheses)会被翻译为一个apply方法调用。当然，这只有在这个对象的类型真正定义了一个apply方法。这不是一个特例，这是一个通用规则。
5. 因此，当赋值给一个变量，而这个变量应用了圆括号和一到多个参数时，编译器会被翻译为对一个update方法调用，其中参数是圆括号内的参数和等号右边的对象。例如：`greetStrings(0) = "Hello"`被翻译为`greetStrings.update(0,"Hello")`
6. Scala完成了概念的简化，通过对所有事情，从数组到表达式，都作为对象和方法。这里不会引入显著的性能开销，Scala在编译的代码中会尽可能使用Java的原始类型
7. Scala对数组初始化提供了简化方法初始化数组。类似于`val numName = Array("zero","two","three")`。这里Scala编译器可以通过你传入的参数推断出数组类型是`Array[String]`。
8. 上面实际上是调用了一个名字为`apply`的工厂方法，会创建并返回一个新的数组。这个`apply`定义在数组的伴生对象(companion object)，接收可变数量的参数。可以看作是Array类的静态方法。
9. Scala数组是一个可变的对象序列。对于同类型的不可变对象序列，Scala提供了List类。不同于Java的List，Scala.List总是不可变的。更一般的，Scala的List是为了满足函数式编程而设计的。
10. 声明一个List：`val list1 = List(1,2,3)`。这样就声明了一个`List[Int]`。
11. 所有的List的方法，如果结果看起来是修改List，都是创建并返回了一个包含新值的新List。
12. 例如List的`:::`方法，这个方法用来联合list。
    ```scala
    val oneTwo = List(1,2)
    val threeFour = List(3,4)
    val oneTwoThreeFour = oneTwo:::threeFour
    ```
    你会发现，oneTwo和threeFour没有改变。
13. List还有一个方法`::`，叫做`cons.`。这个操作会在list前添加一个元素然后返回结果list。例如：`1::threeFour`
14. `::`方法右操作数是一个list。对于方法的结合性有一个简便的记忆规则：如果方法名是操作符，例如`a*b`，这个方法是左操作数调用的，就像`a.*(b)`，除非方法名以一个冒号结尾。如果方法以一个冒号结尾，方法就是右操作数调用的。因此`1::threeFour`，`::`方法是`threeFour`调用的，传入的参数是`1`。就像`twoThree.::(1)`。
15. 创建一个空list可以用Nil，一个创建列表的方式是用`::`连接各个字符串并以Nil作为最后一个元素。例如`val oneTwoThree=1::2::3::Nil`。
16. Scala的List类提供了追加操作:`+`。但是这个操作很少使用，因为这个操作的时间复杂度是随着list尺寸线性增加。而`::`方法是常量时间。如果要高效追加元素，可以加在头部然后逆向调用。或者使用ListBuffer，它是Scala提供的可变列表，他提供了追加操作，然后再调用`toList`。