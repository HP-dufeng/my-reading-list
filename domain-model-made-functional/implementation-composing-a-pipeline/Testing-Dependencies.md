## Testing Dependencies

像这样传递依赖关系的好处是，它使得核心的 function 非常容易测试，因为很容易创建 fake function ，而且也不需要任何特殊的 mock 库。

例如，假设要测试关于 product code 的校验逻辑。这里需要两个 test case，一个 case 校验 checkProductCodeExists 成功的逻辑。另一个 case 校验 checkProductCodeExists 失败的逻辑。接下来就会看到如何编写这些 test 。

开始之前，给你个提示：F# 允许创建包含 空格 和 标点符号 的标识符，只需要将它们包含在双反引号中。对于一般的代码最好不要这样做，但对于 test function 来说，这样做是可以接受的，因为它使测试的输出更具可读性。


下面的例子是校验成功的 case ，它使用了 Arrange/Act/Assert model ：
```
open NUnit.Framework

[<Test>]
let ``If product exists, validation succeeds``() =
    // arrange: set up stub versions of service dependencies
    let checkAddressExists address =
        CheckedAddress address  // succeed
    let checkProductCodeExists productCode =
        true                    // succeed

    // arrange: set up input
    let unvalidatedOrder = ...

    // act: call validateOrder
    let result = validateOrder checkProductCodeExists checkAddressExists ...

    // assert: check that result is a ValidatedOrder, not an error
    ...
```
可以看到 checkAddressExists 和 checkProductCodeExists (它们代表的是两个 service ) 的 stub 版本编写起来很简单，可以在测试中直接定义。

编写校验失败的 case ，只要使 checkProductCodeExists 返回 false 就可以了：
```
let checkProductCodeExists productCode =
    false // fail
```
以下是完整的 test ：
```
[<Test>]
let ``If product doesn't exist, validation fails``() =
    // arrange: set up stub versions of service dependencies
    let checkAddressExists address = ...
    let checkProductCodeExists productCode =
        false // fail

    // arrange: set up input
    let unvalidatedOrder = ...

    // act: call validateOrder
    let result = validateOrder checkProductCodeExists checkAddressExists ...

    // assert: check that result is a failure
    ...
```

当然，在本章中，我们已经说过，service 的错误将通过抛出 exception 来表示。 exception 不是个好东西，所以最好避免抛出 exception 。在下一章中，我们将修复这个问题。

通过这个很小的例子，可以看到使用 functional programming 原则，对于编写 test 来说有很多的好处:
* validateOrder function 是 stateless 的。它不改变任何东西，如果你用相同的输入调用它，你会得到相同的输出。
* 所有依赖项都被显式地传递，所以很容易理解 function 的内在逻辑。
* 所有 side-effect 都封装在参数中，而不是直接封装于 function 本身。同样，这使得 function 易于测试，也易于控制 side-effect 。

测试是个很大的主题，在这里我们没有篇幅来讨论它。下面是一些流行的 F# 友好的测试工具，值得研究:
* FsUnit 用 F# 的语法封装了 NUnit 和 XUnit 等标准测试框架。
* Unquote 显示导致测试失败的所有值 (相当于 “unrolling the stack” )。
* 如果你只是熟悉 “example-based” 的测试方法，比如 NUnit，那么你应该研究基于 “property-based” 的测试方法。FsCheck 是 F# 主要的基于 property-based 的测试库。
* Expecto 是一个轻量级的 F# 测试框架，它使用标准 function 作为 test fixture，而不需要类似 [< test >] 这样专门的 attribute 。