1. 数组添加类型 `Array[Type](total)`
2. 使用下标访问数组的方式是`greetStrings(0)`，跟通常使用中括号不同，scala的访问方式类似方法调用
3. scala没有操作符重载，他没有通常意义上的操作符。而`+,-,*,/`这类符号可以作为方法名称。因此对于`1+2`,你实际上是在`Int`对象`1`上调用它的方法`+`并传入了参数`2`。所以，你可以把`1+2`替换为方法调用语法`(1).+(2)`。
4. 由上，可以知道scala使用小括号访问数组了。scala中特例比java少。数组在scala中就是简单的类。当对变量使用圆括号围绕一到多个参数时，Scala会把这个代码转换为使用变量调用一个名字叫`apply`的方法。所以`greetStrings(i)`等价于`greetStrings.apply(i)`。因此，访问数组的元素对于scala来说就是一个方法调用。这个原则并不仅仅限制于数组：对一个对象应用圆括号内加一些参数(any application of an object to some arguments in parentheses)会被翻译为一个apply方法调用。当然，这只有在这个对象的类型真正定义了一个apply方法。这不是一个特例，这是一个通用规则。
5. 因此，当赋值给一个变量，而这个变量应用了圆括号和一到多个参数时，编译器会被翻译为对一个update方法调用，其中参数是圆括号内的参数和等号右边的对象。例如：`greetStrings(0) = "Hello"`被翻译为`greetStrings.update(0,"Hello")`