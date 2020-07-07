## Adding the Async Effect

在最初的设计中，我们不仅仅使用了 error effect ( Result ) 。多数的 pipeline 也都会使用 async effect 。组合 effect 通常会有点棘手，但是由于这两种 effect 经常同时出现，所以我们将定义一个 asyncResult computation expression 来配合前面定义的 asyncResult 类型。这里不会展示 asyncResult computation expression 实现，像往常一样，你可以在本书的 code repository 中看到它。

使用 asyncResult 就像使用 Result 一样。比如， validateOrder 的实现如下：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        asyncResult {
            let! orderId =
                unvalidatedOrder.OrderId
                |> OrderId.create
                |> Result.mapError ValidationError
                |> AsyncResult.ofResult         // lift a Result to AsyncResult

            let! customerInfo =
                unvalidatedOrder.CustomerInfo
                |> toCustomerInfo
                |> AsyncResult.ofResult
            let! checkedShippingAddress =       // extract the checked address
                unvalidatedOrder.ShippingAddress
                |> toCheckedAddress checkAddressExists
            let! shippingAddress =              // process checked address
                checkedShippingAddress
                |> toAddress
                |> AsyncResult.ofResult
            let! billingAddress = ...
            let! lines =
                unvalidatedOrder.Lines
                |> List.map (toValidatedOrderLine checkProductCodeExists)
                |> Result.sequence      // convert list of Results to a single Result
                |> AsyncResult.ofResult

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

除了用 asyncResult 替换结果之外，还必须确保所有的东西都是 AsyncResult 。例如，OrderId.create 的输出是 Result ，因此必须将其 “lift (提升)” 为 AsyncResult ——这里使用了 helper function AsyncResult.ofResult 来完成 “lift” 。

我们还将 address 的校验分成了两部分。原因是，当将所有 effect 添加回来时，CheckAddressExists function 将返回一个 AsyncResult 。
```
type CheckAddressExists =
    UnvalidatedAddress -> AsyncResult<CheckedAddress,AddressValidationError>
```
这使得 workflow 的 error type 不相匹配了。因此我们创建一个 helper function ( toCheckedAddress ) 来处理这个结果。并且将 service-specific error ( AddressValidationError ) 映射到我们自定义的 ValidationError ：
```
/// Call the checkAddressExists and convert the error to a ValidationError
let toCheckedAddress (checkAddress:CheckAddressExists) address =
    address
    |> checkAddress
    |> AsyncResult.mapError (fun addrError ->
        match addrError with
        | AddressNotFound -> ValidationError "Address not found"
        | InvalidFormat -> ValidationError "Address has bad format"
    )
```
toCheckedAddress function 会返回一个 AsyncResult ，这个 AsyncResult 包装了 CheckedAddress 。因此我们是用 let! 将其 unwrap 到 checkedAddress ，然后就可以将 checkedAddress 传入到 validation stage ( toAddress ) 。

palceOrder function 中使用 asyncResult 也很简单：
```
let placeOrder : PlaceOrder =
    fun unvalidatedOrder ->
        asyncResult {
            let! validatedOrder =
                validateOrder checkProductExists checkAddressExists unvalidatedOrder
                |> AsyncResult.mapError PlaceOrderError.Validation
            let! pricedOrder =
                priceOrder getProductPrice validatedOrder
                |> AsyncResult.ofResult
                |> AsyncResult.mapError PlaceOrderError.Pricing
            let acknowledgementOption = ...

            let events = ...
            
            return events
        }
```
pipeline 中其余的代码也可以以同样的方式转换为使用 asyncResult ，你可以到 code repository 中查看具体的实现。