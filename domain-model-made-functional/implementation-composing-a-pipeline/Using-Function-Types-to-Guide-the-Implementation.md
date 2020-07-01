## Using Function Types to Guide the Implementation

在第2部分关于建模的章节里，定义了一些 function type 用来描述 workflow 的步骤。现在是时候做具体的实现了，但如何确保实现的代码符合这些 function type 的定义呢?

最简单的方法就是以正常的方式定义一个 function ，稍后使用它时，编译器会帮助我们检查类型错误。例如，可以像下面这样定义 validateOrder function ，而不引用我们之前设计的 ValidateOrder type ：
```
let validateOrder
    checkProductCodeExists // dependency
    checkAddressExists     // dependency
    unvalidatedOrder =     // input
    ...
```
大部分 F# 代码，以这样的方式实现，但是如果想要明确正在实现一个特定的 function type ，可以使用另外一种风格。也就是定义一个自己的 function type ，实现时明确指定 type ，像这样：
```
// define a function signature
type MyFunctionSignature = Param1 -> Param2 -> Result

// define a function that implements that signature
let myFunc: MyFunctionSignature =
    fun param1 param2 ->
        ...
```
将此方法应用于 validateOrder function，我们得到：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
     // ^dependency            ^dependency        ^input
        ...
```
这样做的好处是，所有参数和返回值的类型都是由所定义的 function type 决定的，所以在写实现代码的时候，如果使用了错误的签名，你会立即得到错误的提示，而不用等到稍后组合这些 function 时，才发现使用了错误的签名。

这里就有一个示例，其中我们偶然地将一个 int 传递给 checkProductCodeExists function :
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        if checkProductCodeExists 42 then
        //         compiler error ^
        // This expression was expected to have type ProductCode
        // but here has type int
            ...
        ...
```
如果没有适当的 function type 来确定参数的类型，那么编译器将使用类型推断，有可能得出 checkProductCodeExists 处理 int 的结论，从而在其他地方导致(可能令人困惑的)编译器错误。