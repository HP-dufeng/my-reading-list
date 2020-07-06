## Making Life Easier with Computation Expressions

到目前为止，处理的都是简单的 error handling 的逻辑。能够使用 bind 串联 Result-generating function ，能够使用 “adapter” function 改造那些不是 two-track 的 function 使其匹配 two-track 模型。

不过，有时候 workflow 的逻辑很复杂。可能要编写这样的代码： conditional branches ，loops ，深度嵌套的 Result-generating function 。类似这样的代码，F# 使用 “computation expressions” 这样的形式，来隐藏 bind 时的复杂性。computation expressions 只是一种特殊的表达式块。

创建 computation expressions 很简单。比如要为 Result 创建一个叫 result (小写) 的，只需要两个 function 就行了：
* bind ，前面已经见识过了，将它与 Result 一起使用。
* return ，它的作用仅仅只是构造一个值。就 Result 来说，这个 return function 就是 Ok contructor 。

这里不会呈现 result computation expression 的实现细节——可以到 code repository 查看 Result.fs 文件。相反，让我们看看如何在实践中使用 computation expression 来简化具有大量 Result 的代码。

placeOrder 的较早的版本，使用了 bind 将 Result-returning validateOrder 的输出 连接到 priceOrder，如下：
```
let placeOrder unvalidatedOrder =
    unvalidatedOrder
    |> validateOrderAdapted
    |> Result.bind priceOrderAdapted
    |> Result.map acknowledgeOrder
    |> Result.map createEvents
```

通过使用 computation expression ，可以直接操作 validateOrder 和 priceOrder 的输出，就像它们没有被包装在 Result 中一样。

下面是相同的代码，但是使用了 computation expression ：
```
let placeOrder unvalidatedOrder =
    result {
        let! validatedOrder =
            validateOrder unvalidatedOrder
            |> Result.mapError PlaceOrderError.Validation
        let! pricedOrder =
            priceOrder validatedOrder
            |> Result.mapError PlaceOrderError.Pricing
        let acknowledgementOption =
            acknowledgeOrder pricedOrder
        let events =
            createEvents pricedOrder acknowledgementOption
        return events
    }
```

来看看这段代码是如何工作的：
* result computation expression 以单词 “result” 开头，然后将逻辑包装在花括号内。
* let! 看起来像是 let ，但是它的作用是 “unwrap” ，以便获取 Result 的内部值。在 “let! validatedOrder = ...” 语句中的 validatedOrder 就是那个内部值，可以直接传递给 priceOrder function 。
* 贯穿代码块的 error type 必须是相同的，因此 Result.mapError 将 error type 转换成公共的 type 。error 并没有显示的在 result expression 呈现，但是这些 error type 任然要相匹配。
* 最后一行使用了 return 关键字，它代表整个代码块的返回值。

> *！我的提示：*  
> unwrap ，类似 Result 这样的类型都是包装类型，它们将一个普通值包装以下，变成一个带有 *effect* 的值，unwrap 就是获取被这些包装类型所包装的那个普通值，这个普通值被称作包装类型的内部值。

在实践中，在使用 bind 的任何地方都可以使用 let! 。对于那些不需要 bind 的 function 比如，acknowledgeOrder ，只使用普通的语法就行了——不需要使用 Result.map 。

如您所见，computation expressions 使代码看起来好像我们根本没有使用 Result 。它很好地隐藏了复杂性。

这里不会很深入的讲解怎样去定义 computation expression ，但这是很简单的。例如，这里的例子是对上面所使用的 result computation expression 的定义：
```
type ResultBuilder() =
    member this.Return(x) = Ok x
    member this.Bind(x,f) = Result.bind f x

let result = ResultBuilder()
```

本书的后面还会看到更多的 computation expressions ，尤其是 async computation expression ，它以同样优雅的方式来管理 asynchronous callback 。

### Composing Computation Expressions

computation expression 的一个吸引人的地方是它们是可组合的，这是我们一直所追求的品质。

例如，假如使用 result computation expression 来定义 validateOrder 和 priceOrder ：
```
let validateOrder input = result {
    let! validatedOrder = ...
    return validatedOrder
}

let priceOrder input = result {
    let! pricedOrder = ...
    return pricedOrder
}
```
然后像普通的 function 一样，可以将这二者组合成一个更大的 resut expression ：
```
let placeOrder unvalidatedOrder = result {
    let! validatedOrder = validateOrder unvalidatedOrder
    let! pricedOrder = priceOrder validatedOrder
    ...
    return ...
}
```
然后 placeOrder ，可以继续再组合，以此类推。

### Validating an Order With Results

现在可以重构一下 validateOrder function 的实现，这一次将使用 result computation expression 来隐藏 error handling 的逻辑。

提醒一下，下面是不带 Result 的实现：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        let orderId =
            unvalidatedOrder.OrderId
            |> OrderId.create
        let customerInfo =
            unvalidatedOrder.CustomerInfo
            |> toCustomerInfo
        let shippingAddress =
            unvalidatedOrder.ShippingAddress
            |> toAddress checkAddressExists
        let billingAddress = ...
        let lines = ...

        let validatedOrder : ValidatedOrder = {
            OrderId = orderId
            CustomerInfo = customerInfo
            ShippingAddress = shippingAddress
            BillingAddress = billingAddress
            Lines = lines
        }
        
        validatedOrder
```
但是，当我们将所有 helper function 更改为返回 Result 时，代码将不再工作。比如，OrderId.create function 将返回 Result<OrderId,string> ，这不是个纯的 OrderId 。相似的，toCustomerInfo ， toAddress ，都一样。

然而，如果使用 result computation expression ，并且用 let! 代替 let ，就可以直接访问 OrderId ， CustomerInfo ...。以下是实现的代码：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        result {
            let! orderId =
                unvalidatedOrder.OrderId
                |> OrderId.create
                |> Result.mapError ValidationError
            let! customerInfo =
                unvalidatedOrder.CustomerInfo
                |> toCustomerInfo

            let! shippingAddress = ...
            let! billingAddress = ...
            let! lines = ...

            let validatedOrder : ValidatedOrder = {
                OrderId = orderId
                CustomerInfo = customerInfo
                ShippingAddress = shippingAddress
                BillingAddress = billingAddress
                Lines = lines
            }

            return validatedOrder
        }
```
但是，像以前一样，需要使用 Result.mapError 确保所有的 error type 都匹配。OrderId.create 的 error 返回的是一个 string ，因此必须使用 mapError 将它的 error type 提升 ( lift ) 为 ValidationError 。其他 helper function 在处理 simple type 时也需要做同样的事情。这里假设输出的 toCustomerInfo 和 toAddress function 已经是 ValidationError，所以不需要对它们使用 mapError 。

### Working with Lists of Results

当我们起初校验 order line 时，如果不使用 Result type ，则可以使用 List.map 转换每一项 ：
```
let validateOrder unvalidatedOrder =
    ...

    // convert each line into an OrderLine domain type
    let lines =
        unvalidatedOrder.Lines
        |> List.map (toValidatedOrderLine checkProductCodeExists)

    // create and return a ValidatedOrder
    let validatedOrder : ValidatedOrder = {
        ...
        Lines = lines
        // etc
    }

    validatedOrder
```

当 toValidatedOrderLine 返回一个 Result 时，这种方式就不能工作了。使用 map 后，最终得到的结果的类型是 Result<ValidatedOrderLine,...> list ，而不是类型 ValidatedOrderLine list 。

这对我们一点帮助都没有：当设置 ValidatedOrder.Lines 的值是，需要的类型是 Result<item list> 而不是 Result<item> list 。
```
let validateOrder unvalidatedOrder =
    ...

    let lines = // lines is a "list of Result"
        unvalidatedOrder.Lines
        |> List.map (toValidatedOrderLine checkProductCodeExists)

    let validatedOrder : ValidatedOrder = {
        ...
        Lines = lines // compiler error
        //      ^ expecting a "Result of list" here
    }
```

在这里使用 result expression 没有任何帮助——根本的问题是类型不匹配。因此现在的问题是：怎样把类型 Result<item> list 转换成类型 Result<item list> ？

创建一个 hlper function 来完成这个任务：它将遍历 Result<item> list ，如果其中任何一个 item 是 error ，那么总体结果将是 error ，否则，如果它们都是 success ，那么总体结果将是 success 。

在 F# 中，standard list type 是 linked list ——通过将每个元素 prepend (前置) 到一个的 list 中来构建。为了解决我们的问题，需要有一个新版本的 prepend 方法 (在 FP 术语被称作 cons 操作符)，用来将 Result<item> 中的 item ，prepend 到 Result<item list> 中的 list 中去。prepend 实现很简单：
* 如果两个参数都 Ok ，则将第一个参数的 item 前置到 第二个参数的 list 中去，然后使用 Result 包装 list ，最后返回这个 Result 。
* 如果其中任何一个参数是 error ，则返回 error 。  

代码如下：
```
/// Prepend a Result<item> to a Result<list>
let prepend firstR restR =
    match firstR, restR with
    | Ok first, Ok rest -> Ok (first::rest)
    | Error err1, Ok _ -> Error err1
    | Ok _, Error err2 -> Error err2
    | Error err1, Error _ -> Error err1
```
查看 prepend function 的 signature ，能看出它是 generic 的：它接受 Result<'a> 和 Result<'a list> 作为参数，将它们组合进一个新的 Result<'a list> 。

有了 prepend function ，就可以将 Result<'a> list 转换成 Result<'a list> ，从 Result<'a> list 中的最后一项开始向前遍历 (使用 foldBack )，将遍历的每个 Result<'a> 中的 ‘a  都 prepend 到 Result<'a list> 中的 list 中去。把这些逻辑封装到一个 function 中并放入到 Result module 中，命名为 sequence ：
```
let sequence aListOfResults =
    let initialValue = Ok [] // empty list inside Result

    // loop through the list in reverse order,
    // prepending each element to the initial value
    List.foldBack prepend aListOfResults initialValue
```
不要担心这些代码是如何工作的。一旦将它加入到你的 library 中，你只需要知道如何使用它就行了！

写个简单的例子来测试一下 sequence ：
```
type IntOrError = Result<int,string>

let listOfSuccesses : IntOrError list = [Ok 1; Ok 2]
let successResult =
    Result.sequence listOfSuccesses     // Ok [1; 2]
```
可以看到一个包含 Result 的 list (这里的 list 是 [Ok 1; Ok 2] ) 被转换成了一个包含 list 的 Result (这里的 Result 是 Ok[1; 2] )。

再看一下带有 error 的 list 的测试：
```
let listOfErrors : IntOrError list = [ Error "bad"; Error "terrible" ]

let errorResult =
    Result.sequence listOfErrors    // Error "bad"
```
这里得到了另一个 Result ，不过这是它包含的是 error (这里的 error 是 Error “bad” )。

> **! 提示：**  
> 上面的例子只有第一个 error 被返回了。然而，很多情况需要返回所有的 error ，特别是做 校验 的时候。functional programming 解决这种问题的技术是 ***Applicative*** 。下一节会有有个大概的介绍但不会太详细。

在我们的工具箱里有了 Result.sequence 之后，终于可以编写代码来构建 ValidatedOrder ：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        result {
            let! orderId = ...
            let! customerInfo = ...
            let! shippingAddress = ...
            let! billingAddress = ...
            let! lines =
                unvalidatedOrder.Lines
                |> List.map (toValidatedOrderLine checkProductCodeExists)
                |> Result.sequence // convert list of Results to a single Result

            let validatedOrder : ValidatedOrder = {
                OrderId = orderId
                CustomerInfo = customerInfo
                ShippingAddress = shippingAddress
                BillingAddress = billingAddress
                Lines = lines
            }

            return validatedOrder
        }
```
如果你关心性能，可以将 List.map 和后面跟着 Result.sequence 组合进一个单独的 function (通常被称作 traverse<sup>1</sup> ) 来提高性能。但我们不会在这里偏离主题。

快要接近与完成了，但还有一个小问题。validateOrder 输出的 error type 是 ValidationError 。

然而对于整个 pipeline 都使用统一的 PlaceOrderError 。因此，在 placeOrder function 中 需要将 Result<ValidatedOrder, ValidationError> type 转换成 Result<ValidatedOrder,PlaceOrderError> type 。像之前一样可以使用 mapError 。类似的，需要将 priceOrder 的输出 从 PricingError 转换成 PlaceOrderError 。

下面是使用 mapError 实现的整个 workflow 的样子 ：
```
let placeOrder : PlaceOrder =       // definition of function
    fun unvalidatedOrder ->
        result {
            let! validatedOrder =
                validateOrder checkProductExists checkAddressExists unvalidatedOrder
                |> Result.mapError PlaceOrderError.Validation
            let! pricedOrder =
                priceOrder getProductPrice validatedOrder
                |> Result.mapError PlaceOrderError.Pricing
            let acknowledgementOption = ...

            let events = ...
            return events
    }
```
输入的是我们期望的 Result<ValidatedOrder,PlaceOrderError> 。

---
> 1.    https://fsharpforfunandprofit.com/posts/elevated-world-4/