## The Assembled Pipeline

在本章中已经看到了很多分散的代码片段。让我们将它们整合在一起，看看完整的 pipeline 是如何构建的。
* 将实现特定 workflow 的所有代码放在同一个 module 中，该 module 以 workflow 的名字命名 (比如，PlaceOrderWorkflow.fs ) 。
* 在文件内，先定义 type 。
* 接着是每一个步骤的实现代码。
* 在最后，把这些步骤组装进 main workflow function 。

为了节省篇幅，这里只显示了文件内容的大概。如果您想查看完整的文件内容，本章的所有代码都可以在与本书相关联的 code repository 中找到。

首先是这些 type：

有些和 workflow 相关的 “public” type 是在别的地方定义的，也许是在 API module 中，
第7章定义了一些这样的 “public” type ，它们是实现 workfow 的每个步骤所需要的，所以需要引入这些 type 。  
> **! 我的提示：**  
> 这里所说的也就是，定义和实现被分割到不同的文件，或者 module 。  
> 定义，主要是定义一些 type ，这些 type 是表示 Domain 的 模型。  
> 实现，是对这些模型的具体实现。
```
module PlaceOrderWorkflow =
    // make the shared simple types (such as
    // String50 and ProductCode) available.
    open SimpleTypes

    // make the public types exposed to the
    // callers available
    open API

    // ==============================
    // Part 1: Design
    // ==============================

    // NOTE: the public parts of the workflow -- the API --
    // such as the `PlaceOrderWorkflow` function and its
    // input `UnvalidatedOrder`, are defined elsewhere.
    // The types below are private to the workflow implementation.

    // ----- Validate Order -----
    type CheckProductCodeExists =
        ProductCode -> bool
    type CheckedAddress =
        CheckedAddress of UnvalidatedAddress
    type CheckAddressExists =
        UnvalidatedAddress -> CheckedAddress
    type ValidateOrder =
        CheckProductCodeExists  // dependency
        -> CheckAddressExists   // dependency
        -> UnvalidatedOrder     // input
        -> ValidatedOrder       // output

    // ----- Price order -----
    type GetProductPrice = ...
    type PriceOrder = ...
    // etc
```
type 之后，在同一个文件中，可以放置基于这些 type 的实现。以下是第一个步骤，validateOrder ，摘自本章 Implementing the Validation Step 这一节 ：
```
// ==============================
// Part 2: Implementation
// ==============================

// ------------------------------
// ValidateOrder implementation
// ------------------------------

let toCustomerInfo (unvalidatedCustomerInfo: UnvalidatedCustomerInfo) =
    ...
let toAddress (checkAddressExists:CheckAddressExists) unvalidatedAddress =
    ...
let predicateToPassthru = ...
let toProductCode (checkProductCodeExists:CheckProductCodeExists) productCode =
    ...
let toOrderQuantity productCode quantity =
    ...
let toValidatedOrderLine checkProductExists (unvalidatedOrderLine:UnvalidatedOrderLine) =
    ... 

/// Implementation of ValidateOrder step
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        let orderId =
            unvalidatedOrder.OrderId
            |> OrderId.create
        let customerInfo = ...
        let shippingAddress = ...
        let billingAddress = ...
        let lines =
            unvalidatedOrder.Lines
            |> List.map (toValidatedOrderLine checkProductCodeExists)
        let validatedOrder : ValidatedOrder = {
            OrderId = orderId
            CustomerInfo = customerInfo
            ShippingAddress = shippingAddress
            BillingAddress = billingAddress
            Lines = lines
        }

        validatedOrder
```
这里就不呈现其他步骤的实现了 (可以到和本书相关的 code repository 查看代码)，直接跳到文件的最底部——实现 top level PlaceOrder function 的地方：
```
// ------------------------------
// The complete workflow
// ------------------------------
let placeOrder
    checkProductExists                  // dependency
    checkAddressExists                  // dependency
    getProductPrice                     // dependency
    createOrderAcknowledgmentLetter     // dependency
    sendOrderAcknowledgment             // dependency
    : PlaceOrderWorkflow =              // definition of function

    fun unvalidatedOrder ->
        let validatedOrder =
            unvalidatedOrder
            |> validateOrder checkProductExists checkAddressExists
        let pricedOrder =
            validatedOrder
            |> priceOrder getProductPrice
        let acknowledgementOption =
            pricedOrder
            |> acknowledgeOrder createOrderAcknowledgmentLetter sendOrderAcknowledgment

        let events =
            createEvents pricedOrder acknowledgementOption
            
        events
```