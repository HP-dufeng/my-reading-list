## Applying “bind” and “map” to Our Pipeline

理解了这些概念之后，应当将它们付诸实践。使用这些 error-generating function 组合 workflow pipeline ，并在需要时对它们进行调整，以便它们能够相互匹配。

快速的回顾一下 pipeline 中的组件，重点关注 Result type ，暂时忽略 Async effect 和 service dependency ：

首先，如果输入参数的格式错误，ValidateOrder 将返回 error ，因此它是一个 switch function ，它的 signature 将会是：
```
type ValidateOrder =
    // ignoring additional dependencies for now
    UnvalidatedOrder                            // input
    -> Result<ValidatedOrder, ValidationError> // output
```

PriceOrder 这个步骤也有可能会出错，因此它的 signature 将会是：
```
type PriceOrder =
    ValidatedOrder                          // input
    -> Result<PricedOrder, PricingError>    // output
```

AcknowledgeOrder 和 CreateEvents 这两个步骤将始终是成功的，因此它们的 signature 为：
```
type AcknowledgeOrder =
    PricedOrder                             // input
        -> OrderAcknowledgmentSent option   // output

type CreateEvents =
    PricedOrder                             // input
        -> OrderAcknowledgmentSent option   // input (event from previous step)
        -> PlaceOrderEvent list             // output
```

从组合 ValidateOrder 和 PriceOrder 开始。ValidateOrder 的 error type 是 ValidationError ，PriceOrder 的 error type 是 PricingError 。正如在前面看到的，由于 error type 不同，所以这些 function 是不兼容的。必须让它们返回相同的 error type ——某个 common error type ，贯穿于整个 pipeline ，把它称作 PlaceOrderError 。PlaceOrderError 的定义如下：
```
type PlaceOrderError =
    | Validation of ValidationError
    | Pricing of PricingError
```
定义 validateOrder 和 priceOrder 的新版本，使用 mapError 组合它们，就像前面的FruitError 示例所做的那样：
```
// Adapted to return a PlaceOrderError
let validateOrderAdapted input =
    input
    |> validateOrder // the original function
    |> Result.mapError PlaceOrderError.Validation

// Adapted to return a PlaceOrderError
let priceOrderAdapted input =
    input
    |> priceOrder // the original function
    |> Result.mapError PlaceOrderError.Pricing
```

好了，可以使用 bind 将它们串起来了：
```
let placeOrder unvalidatedOrder =
    unvalidatedOrder
    |> validateOrderAdapted             // adapted version
    |> Result.bind priceOrderAdapted // adapted version
```
注注意，validateOrderAdapted function 前面不需要加 bind ，因为它是 pipeline 中的第一个 function 。

接下来，validateOrderAdapted 和 createEvents 没有 error ——它们是 one-track function ——因此可以使用 Result.map 将它们转换成 two-track function ： 
```
let placeOrder unvalidatedOrder =
    unvalidatedOrder
    |> validateOrderAdapted
    |> Result.bind priceOrderAdapted
    |> Result.map acknowledgeOrder      // use map to convert to two-track
    |> Result.map createEvents          // convert to two-track
```
这个 placeOrder function 的 signature 是：
```
UnvalidatedOrder -> Result<PlaceOrderEvent list,PlaceOrderError>
```

和我们想要的非常接近了。

分析一下这个新的实现：
* pipeline 中的每个 function 都有可能产生 error ，在 signature 中标识了的这些 error 。可以对 function 进行隔离测试，组合之后不会产生不可预期的行为。
* 使用 two-track 模型将 function 串联起来。步骤中如果发生了 error ，那么会跳过 pipeline 中其余的  function 。
* top-level placeOrder 的整体执行流程任然是干净的。不需要 try/catch 这种语句块。