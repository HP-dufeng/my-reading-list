## Implementing the Rest of the Steps

现在我们已经知道怎样实现 validateOrder ，当然也可以使用同样的方式来构建其余的 pipeline function 。

这是为 pricing 步骤设计的 function 的最初版本，具有 *effect* ：
```
type PriceOrder =
    GetProductPrice         // dependency
        -> ValidatedOrder   // input
        -> Result<PricedOrder, PlaceOrderError>   // output
```
和之前一样，暂时去掉 *effect* ，像这样：
```
type GetProductPrice = ProductCode -> Price
type PriceOrder =
    GetProductPrice         // dependency
        -> ValidatedOrder   // input
        -> PricedOrder      // output
```
以下是对 PriceOrder 的实现。它简单地将每个 order line 转换为一个 PricedOrderLine ，并使用转换后的 PricedOrderLine 构建一个新的 PricedOrder ：
```
let priceOrder : PriceOrder =
    fun getProductPrice validatedOrder ->
        let lines =
            validatedOrder.Lines
            |> List.map (toPricedOrderLine getProductPrice)
        let amountToBill =
            lines
            // get each line price
            |> List.map (fun line -> line.LinePrice)
            // add them together as a BillingAmount
            |> BillingAmount.sumPrices
        
        let pricedOrder : PricedOrder = {
            OrderId = validatedOrder.OrderId
            CustomerInfo = validatedOrder.CustomerInfo
            ShippingAddress = validatedOrder.ShippingAddress
            BillingAddress = validatedOrder.BillingAddress
            Lines = lines
            AmountToBill = amountToBill
        }

        pricedOrder
```

顺便说一下，如果 pipeline 中有很多步骤，你暂时不想做实现 (或者不知道如何实现)，那么你可以这样做：
```
let priceOrder : PriceOrder =
    fun getProductPrice validatedOrder ->
        failwith "not implemented"
```
在规划实现时，为了把最终实现的结构呈现出来，使用 “not implemented” exception 会很方便。这样确保项目在任何时候都可以编译。例如，可以使用此方法构建 pipeline 的某个步骤的 mock 版本，然后将其与其他步骤一起使用，这样可以确保每个步骤的 function 的输入和输出的类型是匹配的。

回到实现 priceOrder 的代码上来，这里引入了两个 helper function ： toPricedOrderLine 和 BillingAmount.sumPrices 。

在 BillingAmount module 中已经添加了 BillingAmount.sumPrices function (以及 create 和 value )BillingAmount.sumPrices 简单的将 prices list 中的元素相加求和，然后将结果包装成 BillingAmount 。为什么需要定义一个 BillingAmount type？这是为了区别 Price type ，二者的校验规则也许会不一样。
```
/// Sum a list of prices to make a billing amount
/// Raise exception if total is out of bounds
let sumPrices prices =
    let total = prices |> List.map Price.value |> List.sum
    create total
```

toPricedOrderLine 是一个 helper function ，功能和之前已经看到过的类似。它将 validatedOrder.Lines 这个 list 中的某一项转换成 PricedOrderLine 。
```
/// Transform a ValidatedOrderLine to a PricedOrderLine
let toPricedOrderLine getProductPrice (line:ValidatedOrderLine) : PricedOrderLine =
    let qty = line.Quantity |> OrderQuantity.value
    let price = line.ProductCode |> getProductPrice
    let linePrice = price |> Price.multiply qty

    {
        OrderLineId = line.OrderLineId
        ProductCode = line.ProductCode
        Quantity = line.Quantity
        LinePrice = linePrice
    }
```
在这个 function 中，有另外一个 helper function Price.multiply ，作用是将 price 和 quantity 相乘。
```
/// Multiply a Price by a decimal qty.
/// Raise exception if new price is out of bounds.
let multiply qty (Price p) =
    create (qty * p)
```
price 步骤到此完成！


### Implementing the Acknowledgement Step

这里是为 acknowledgement 步骤所设计的 type ，同样暂时去掉了 *effect* ：
```
type HtmlString = HtmlString of string
type CreateOrderAcknowledgmentLetter =
    PricedOrder -> HtmlString

type OrderAcknowledgment = {
    EmailAddress : EmailAddress
    Letter : HtmlString
}

type SendResult = Sent | NotSent
type SendOrderAcknowledgment =
    OrderAcknowledgment -> SendResult

type AcknowledgeOrder =
    CreateOrderAcknowledgmentLetter         // dependency
        -> SendOrderAcknowledgment          // dependency
        -> PricedOrder                      // input
        -> OrderAcknowledgmentSent option   // output
```
以下是具体的实现：
```
let acknowledgeOrder : AcknowledgeOrder =
    fun createAcknowledgmentLetter sendAcknowledgment pricedOrder ->
        let letter = createAcknowledgmentLetter pricedOrder
        let acknowledgment = {
            EmailAddress = pricedOrder.CustomerInfo.EmailAddress
            Letter = letter
        }

        // if the acknowledgement was successfully sent,
        // return the corresponding event, else return None
        match sendAcknowledgment acknowledgment with
        | Sent ->
            let event = {
                OrderId = pricedOrder.OrderId
                EmailAddress = pricedOrder.CustomerInfo.EmailAddress
            }
            Some event
        | NotSent ->
            None
```
实现起来很简单——不需要 helper function ——所以简单。

sendAcknowledgment 这个依赖项呢？在某个时刻，将不得不实现它。然而，现在可以不去管它。这是使用 function 参数化依赖项的好处之一——你可以留到最后在决定如何实现它，但是你仍然可以构建和编译大部分代码。

### Creating the Events

最后，只需要创建要从工作流返回的 event 。这里有一个小的需求，即 billing event 只应该在 amount 大于 0 时发送。type 的设计如下：
```
/// Event to send to shipping context
type OrderPlaced = PricedOrder

/// Event to send to billing context
/// Will only be created if the AmountToBill is not zero
type BillableOrderPlaced = {
    OrderId : OrderId
    BillingAddress: Address
    AmountToBill : BillingAmount
}

type PlaceOrderEvent =
    | OrderPlaced of OrderPlaced
    | BillableOrderPlaced of BillableOrderPlaced
    | AcknowledgmentSent of OrderAcknowledgmentSent

type CreateEvents =
    PricedOrder                             // input
        -> OrderAcknowledgmentSent option   // input (event from previous step)
        -> PlaceOrderEvent list             // output
```

不需要创建 OrderPlaced event ，因为它与 PricedOrder 是一样的，OrderAcknowledgmentSent  event 在上一步中也应该已经创建了。

但是这里需要 BillableOrderPlaced event ，因此我们创建一个 createBillingEvent function 。这个 function 需要对 billingAmount field 校验 non-zero 的约束条件，所以它应该返回 *optional* event 。
```
// PricedOrder -> BillableOrderPlaced option
let createBillingEvent (placedOrder:PricedOrder) : BillableOrderPlaced option =
    let billingAmount = placedOrder.AmountToBill |> BillingAmount.value
    if billingAmount > 0M then
        let order = {
            OrderId = placedOrder.OrderId
            BillingAddress = placedOrder.BillingAddress
            AmountToBill = placedOrder.AmountToBill
        }
        Some order
    else
        None
```

现在有了一个 OrderPlaced event ，一个 optional OrderAcknowledgmentSent event 和一个optional BillableOrderPlaced event ——我们应该如何返回它们?

为每个 event 创建一个 choice type ( PlaceOrderEvent ) ，然后将它们添加到 list 中，返回这个 list 。所以，首先将每个 event 转换成 choice type 。对于 OrderPlaced event ，可以直接使用 PlaceOrderEvent.OrderPlaced 构造函数构建，而 OrderAcknowledgmentSent 和 BillableOrderPlaced 需要使用 Option.map ，因为它俩是 optional 的。
```
let createEvents : CreateEvents =
    fun pricedOrder acknowledgmentEventOpt ->
        let event1 =
            pricedOrder
            |> PlaceOrderEvent.OrderPlaced
        let event2Opt =
            acknowledgmentEventOpt
            |> Option.map PlaceOrderEvent.AcknowledgmentSent
        let event3Opt =
            pricedOrder
            |> createBillingEvent
            |> Option.map PlaceOrderEvent.BillableOrderPlaced

        // return all the events how?
        ...
```
现在它们都是 PlaceOrderEvent type ，但有些是 optional 。怎么解决这个问题？好的，再一次，用同样的技巧把它们转换成更一般的 type 。就这个例子来说是 list 。

对于 OrderPlaced 可以使用 List.singleton 转换，对于 optional 的 我们创建一个 helper function 叫做 listOfOption ：
```
/// convert an Option into a List
let listOfOption opt =
    match opt with
    | Some x -> [x]
    | None -> []
```

好了，所有三个 event 的类型都是相同的了，可以在另一个 list 中返回它们:
```
let createEvents : CreateEvents =
    fun pricedOrder acknowledgmentEventOpt ->
        let events1 =
            pricedOrder
            |> PlaceOrderEvent.OrderPlaced
            |> List.singleton
        
        let events2 =
            acknowledgmentEventOpt
            |> Option.map PlaceOrderEvent.AcknowledgmentSent
            |> listOfOption

        let events3 =
            pricedOrder
            |> createBillingEvent
            |> Option.map PlaceOrderEvent.BillableOrderPlaced
            |> listOfOption

        // return all the events
        [
            yield! events1
            yield! events2
            yield! events3
        ]
```

这种将不兼容的类型转换或 “lifting” 为共享类型的方法是处理 function composition 问题的关键技术。在下一章中，我们将使用它来处理不同的 Result type 之间不匹配的问题。

