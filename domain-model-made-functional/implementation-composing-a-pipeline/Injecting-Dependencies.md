## Injecting Dependencies

我们有许多 low-level helper function ，比如 toValidProductCode，它接受一个 被描述为 service 的 function 作为参数。

这些作为 service 的 function ，通常被叫做依赖项，在设计中层级是相当深的，那么我们如何将这些依赖从顶层传递到需要它们的 function 呢?

object-oriented programming 使用的技术是 dependency injection ，并且有很多的 IoC container 可以用。  
functional programming 不会这么做，因为这样会使依赖项变得 implicit 。相反，我们总是希望将依赖项作为 explicit parameter 传递，以确保依赖项是显而易见的。

functional programming 领域有很多种技术可以做这样的事情，比如，“Reader Monad” 和 “Free Monad” ，由于这是一本介绍性的书，我们将坚持使用最简单的方式，即将所有依赖项传递到 top level function ，然后再将它们传递到 inner function ，inner function 再将它们向下传递到 inner function ，依此类推。

假设已经实现了之前定义的 helper function：
```
// low-level helper functions
let toAddress checkAddressExists unvalidatedAddress =
    ...
let toProductCode checkProductCodeExists productCode =
    ...
```
它们都有一个 explicit parameter 作为它们的依赖项。

现在作为创建 order line 的一部分，我们需要一个 product code ，这表示 toValidatedOrderLine 需要调用 toProductCode ，也就意味着 toValidatedOrderLine 也需要 checkProductCodeExists 这个参数：
```
// helper function
let toValidatedOrderLine checkProductExists unvalidatedOrderLine =
//                       ^ needed for toProductCode, below

    // create the components of the line
    let orderLineId = ...
    let productCode =
        unvalidatedOrderLine.ProductCode
        |> toProductCode checkProductExists //use service
    ...
```
向上移动一层，validateOrder function 需要调用 toAddress 和 toValidatedOrderLine，因此它又需要将这两个 function 它们本身的依赖项， 作为额外的参数传递进来：
```
let validateOrder : ValidateOrder =
    fun checkProductExists      // dependency for toValidatedOrderLine
        checkAddressExists      // dependency for toAddress
        unvalidatedOrder ->

        // build the validated address using the dependency
        let shippingAddress =
            unvalidatedOrder.ShippingAddress
            |> toAddress checkAddressExists
        ...

        // build the validated order lines using the dependency
        let lines =
            unvalidatedOrder.Lines
            |> List.map (toValidatedOrderLine checkProductExists)
        ...
```
依此类推，直到到达 top level function ,所有的依赖项都将在此处设置。在 object-oriented design 中，这个 top level function 通常称为 *Composition Root* ，所以在这里我们也使用这个术语。

是否应该把 placeOrder workflow function 作为 composition root ？不，因为通常设置 service 时都需要访问环境配置。更好的做法是 placeOrder 只提供它自己需要的 service 作为参数，像这样：
```
let placeOrder
    checkProductExists                  // dependency 
    checkAddressExists                  // dependency   
    getProductPrice                     // dependency  
    createOrderAcknowledgmentLetter     // dependency                 
    sendOrderAcknowledgment             // dependency               
    : PlaceOrderWorkflow =              // function definition     
        fun unvalidatedOrder ->                     
        ...
```
这样做的好处是，整个 workflow 易于测试，因为所有依赖项都是 fake-able 。

实际上，composition root function 应该尽可能接近于应用程序的入口点，这些入口点像是：console app 的 main function ，或者 long-running app 的 OnStartup/Application_Start 。

下面是 composition root 的示例，这个示例是一个 web service ，使用了 Suave 框架。 首先设置 service 这些依赖项，然后将这些依赖项输入到 workflow ，最后设置 route 。
```
let app : WebPart =
    // setup the services used by the workflow
    let checkProductExists = ...
    let checkAddressExists = ...
    let getProductPrice = ...
    let createOrderAcknowledgmentLetter = ...
    let sendOrderAcknowledgment = ...
    let toHttpResponse = ...

    // partially apply the services to the workflows
    let placeOrder =
        placeOrder
            checkProductExists
            checkAddressExists
            getProductPrice
            createOrderAcknowledgmentLetter
            sendOrderAcknowledgment

    let changeOrder = ...
    let cancelOrder = ...

    // set up the routing
    choose
    [ POST >=> choose
        [ 
            path "/placeOrder"
                >=> deserializeOrder  // convert JSON to UnvalidatedOrder
                >=> placeOrder        // do the workflow
                >=> postEvents        // post the events onto queues
                >=> toHttpResponse    // return 200/400/etc based on the output
            path "/changeOrder"
                >=> ...
            path "/cancelOrder"
                >=> ...
        ]
    ]
```
可以看到，如果 path 是 /placeOrder ，将启动 “place order” 流程，首先 deserializing 输入参数，然后调用 placeOrder pipeline ，然后发布 event ，然后将输出转换为 HTTP response 。限于篇幅，除了 placeOrder 不会讨论其他的 function 。但是在第11章会讨论 deserialization 的技术。

### Too Many Dependencies?

validateOrder 有两个依赖项。如果它需要 4个，5个，或者更多的参数呢？如果其他的步骤也有很多的依赖项，哦，依赖泛滥！出现这种情况，你会怎么做？

首先，可能是你的 function 做了太多的事情。是否能把它重构成更小的 function ？如果不能，则可以将这么多的依赖项封装到一个 struct 中，然后将这个 struct 作为参数传递。

一种常见的情况是，child function 的依赖项特别复杂。例如，checkAddressExists 需要调用一个 web service ，这个 web service 有两个参数 URI 和 credentials ：
```
let checkAddressExists endPoint credentials =
    ...
```
需要把这两个额外的参数传递给 checkAddressExists 的调用者 toAddress 吗?
```
let toAddress checkAddressExists endPoint credentials unvalidatedAddress =
//                           only ^ needed ^ for checkAddressExists

    // call the remote service
    let checkedAddress = checkAddressExists endPoint credentials unvalidatedAddress
    //                      2 extra parameters ^ passed in ^
    ...

```
然后我们还要把这些额外的参数传递给 toAddress 的调用者，等等，一直到 top level ：
```
let validateOrder
    checkProductExists
    checkAddressExists
    endPoint    // only needed for checkAddressExists
    credentials // only needed for checkAddressExists
    unvalidatedOrder =
    ...
```

当然，不必如此。这些 intermediate function 不需要知道 checkAddressExists 的依赖关系。一种更好的方法是，把这些依赖项包装进一个 child function，然后将这个 child function 传入到 top-level function 。

例如下面的代码，将 uri 和 credentials 这两个参数 bake in (传入) 到 checkAddressExists function 中，以便以后可以将其作为一个单参数 function 使用。这个简化的 function 可以在任何地方传递：
```
let placeOrder : PlaceOrderWorkflow =
    // initialize information (e.g from configuration)
    let endPoint = ...
    let credentials = ...

    // make a new version of checkAddressExists
    // with the credentials baked in
    let checkAddressExists = checkAddressExists endPoint credentials
    // etc

    // set up the steps in the workflow
    let validateOrder =
        validateOrder checkProductCodeExists checkAddressExists
        //          the new checkAddressExists ^
        //          is a one parameter function
    // etc

    // return the workflow function
    fun unvalidatedOrder ->
        // compose the pipeline from the steps
        ...
```

使用 “pre-built” helper function 来减少参数，这是一种隐藏复杂性的常见技术。将一个 function 传递给另一个 function 时，“interface”—— function type ——应该尽可能地隐藏所有依赖项，使其最小化。
