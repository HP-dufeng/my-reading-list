## Implementing the Validation Step

现在可以实现 validation 这个步骤了。validation 这一步骤将接受 unvalidated order 及其所有的 primitive field 作为参数，然后将其转换为正确的、完全验证过的 domain object 。

为 validation 这个步骤创建 function type ：
```
type CheckAddressExists =
    UnvalidatedAddress -> AsyncResult<CheckedAddressAddressValidationError>
    
type ValidateOrder =
    CheckProductCodeExists      // dependency
        -> CheckAddressExists   // AsyncResult dependency
        -> UnvalidatedOrder     // input
        -> AsyncResult<ValidatedOrder,ValidationError list>   // output
```

正如我们所说的，这一章不会处理 *effect* ，所以暂时移除 AsyncResult 部分 ：
```
type CheckAddressExists =
    UnvalidatedAddress -> CheckedAddress
type ValidateOrder =
    CheckProductCodeExists      // dependency
        -> CheckAddressExists   // dependency
        -> UnvalidatedOrder     // input
        -> ValidatedOrder       // output 
```
实现这个 function type 定义，从 UnvalidatedOrder 创建 ValidatedOrder 的步骤如下：
* 创建 OrderId domain type ，值可以从 unvalidated order 中的相应的 OrderId field 获取。
* 创建 CustomerInfo domain type ，值可以从 unvalidated order 中的相应的 UnvalidatedCustomerInfo field 获取。
* 创建 Address domain type ，值可以从 unvalidated order 中的相应的 ShippingAddress field 获取，这个 ShippingAddress field 的类型是 UnvalidatedAddress 。
* 类似地，BillingAddress 和所有其他的 field 也是如此创建。
* 一旦 ValidatedOrder 的所有 field 创建完成，就可以创建一个 ValidatedOrder object 了。

这里是代码：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        let orderId =
            unvalidatedOrder.OrderId
            |> OrderId.create
        
        let customerInfo =
            unvalidatedOrder.CustomerInfo
            |> toCustomerInfo   // helper function

        let shippingAddress =
            unvalidatedOrder.ShippingAddress
            |> toAddress        // helper function
        
        // and so on, for each property of the unvalidatedOrder
        // when all the fields are ready, use them to
        // create and return a new "ValidatedOrder" record

        {
            OrderId = orderId
            CustomerInfo = customerInfo
            ShippingAddress = shippingAddress
            BillingAddress = ...
            Lines = ...
        }
```
这里使用了一些 helper function ，比如我们还没有定义的 toCustomerInfo 和 toAddress 。这些 function 的作用是从 unvalidated type 构建 domain type 。比如，toAddress 将 UnvalidatedAddress type 转换成相应的 domain type Address ，如果转换时未能满足约束条件(如，非空，小于50个字符...)，则引发错误。

一旦准备好了所有这些 helper function ，将 unvalidated order (or any non-domain type) 转换为 domain type 的逻辑就很简单了：对于 domain type 的每个 field (在本例中为 ValidatedOrder )，找到 non-domain type ( UnvalidatedOrder ) 中与之对应的 field ，并使用一个 helper function 将该 field 转换为 domain type 。

在转换 order 的子组件时，也可以使用完全相同的方法。比如，这里实现了 toCustomerInfo function , 它将 UnvalidatedCustomerInfo 转换成 CustomerInfo ：
```
let toCustomerInfo (customer:UnvalidatedCustomerInfo) : CustomerInfo =
    // create the various CustomerInfo properties
    // and throw exceptions if invalid
    let firstName = customer.FirstName |> String50.create
    let lastName = customer.LastName |> String50.create
    let emailAddress = customer.EmailAddress |> EmailAddress.create

    // create a PersonalName
    let name : PersonalName = {
        FirstName = firstName
        LastName = lastName
    }

    // create a CustomerInfo
    let customerInfo : CustomerInfo = {
        Name = name
        EmailAddress = emailAddress
    }

    // ... and return it
    customerInfo
```

### Creating a Valid, Checked, Address

toAddress function 有点复杂，因为它不仅需要将 primitive type 转换为 domain object ，而且还必须检查 address 是否存在(使用 CheckAddressExists service)。下面是完整的实现，如下:
```
let toAddress (checkAddressExists:CheckAddressExists) unvalidatedAddress =
    // call the remote service
    let checkedAddress = checkAddressExists unvalidatedAddress
    // extract the inner value using pattern matching
    let (CheckedAddress checkedAddress) = checkedAddress

    let addressLine1 =
        checkedAddress.AddressLine1 |> String50.create
    let addressLine2 =
        checkedAddress.AddressLine2 |> String50.createOption
    let addressLine3 =
        checkedAddress.AddressLine3 |> String50.createOption
    let addressLine4 =
        checkedAddress.AddressLine4 |> String50.createOption
    let city =
        checkedAddress.City |> String50.create
    let zipCode =
        checkedAddress.ZipCode |> ZipCode.create

    // create the address
    let address : Address = {
        AddressLine1 = addressLine1
        AddressLine2 = addressLine2
        AddressLine3 = addressLine3
        AddressLine4 = addressLine4
        City = city
        ZipCode = zipCode
    }

    // return the address
    address
```
注意，代码中使用了 String50 module 中的另一个构造函数——String50.createOption——它允许输入为 null 或 empty ，并在这种情况下返回 None 。

toAddress function 需要调用 checkAddressExists ，因此我们将其添加到参数列表里，validateOrder function 在调用 toAddress 时， 必须将 checkAddressExists 传递给它 ：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        let orderId = ...
        let customerInfo = ...
        let shippingAddress =
        unvalidatedOrder.ShippingAddress
        |> toAddress checkAddressExists  // new parameter
        ...
```
您可能想知道，为什么我们只传递一个参数给 toAddress ，而实际上它有两个参数。第二个参数 (shipping address) 是通过 pipeline 传递给它的。 这是我们在第8章 Partial Application 这一节里讨论过的 partial application 技术的一个例子。

### Creating the Order Lines

创建 order line list 更加复杂一点。首先，需要将单个的 UnvalidatedOrderLine 转换成 ValidatedOrderLine 。toValidatedOrderLine 实现如下：
```
let toValidatedOrderLine checkProductCodeExists (unvalidatedOrderLine:UnvalidatedOrderLine) =
    let orderLineId =
        unvalidatedOrderLine.OrderLineId
        |> OrderLineId.create
    let productCode =
        unvalidatedOrderLine.ProductCode
        |> toProductCode checkProductCodeExists // helper function
    let quantity =
        unvalidatedOrderLine.Quantity
        |> toOrderQuantity productCode // helper function
    
    let validatedOrderLine = {
        OrderLineId = orderLineId
        ProductCode = productCode
        Quantity = quantity
    }

    validatedOrderLine
```
这和上面的 toAddress 类似。有两个 helper function , toProductCode 和 toOrderQuantity ，我们将很快讨论它们。

此刻，已经有了转换单个元素的方法，接下来就可以对整个 list 进行转换：
```
let validateOrder : ValidateOrder =
    fun checkProductCodeExists checkAddressExists unvalidatedOrder ->
        let orderId = ...
        let customerInfo = ...
        let shippingAddress = ...

        let orderLines =
            unvalidatedOrder.Lines
            // convert each line using `toValidatedOrderLine`
            |> List.map (toValidatedOrderLine checkProductCodeExists)
        ...
```

接下来看看 toOrderQuantity 这个 htlper function 。这是一个很好的关于 bounded context 处理校验边界的例子：输入是来自 UnvalidatedOrderLine 的未经校验的原始 unvalidated decimal , 但是输出 (OrderQuantity) 是一个 choice type ，这个 choice type 的每个分支上都有对应的检验方法：
```
let toOrderQuantity productCode quantity =
    match productCode with
    | Widget _ ->
        quantity
        |> int                  // convert decimal to int
        |> UnitQuantity.create  // to UnitQuantity
        |> OrderQuantity.Unit   // lift to OrderQuantity type
    | Gizmo _ ->
        quantity
        |> KilogramQuantity.create  // to KilogramQuantity
        |> OrderQuantity.Kilogram   // lift to OrderQuantity type
```
match 根据 ProductCode 选择合适的分支构建 OrderQuantity 。如果 ProductCode 是 Widget ，则将原始的 decimal 转换成 int ，然后再以此创建 UnitQuantity 。Gizmo 分支也是类似的处理方式。

但是不能止步于此。如果这样做，一个分支将返回 UnitQuantity，而另一个分支将返回一个 KilogramQuantity 。这两个是不同的类型，因此会产生编译错误。将两个分支的结果都转换成 choice type OrderQuantity ，就可以确保返回的类型是相同的，当然编译也能通过了。

其它的 helper function ，toProductCode ,乍一看应该很容易实现。要尽可能的使 function 使用 pipeline ，因此代码看起来是这样的：
```
let toProductCode (checkProductCodeExists:CheckProductCodeExists) productCode =
    productCode
    |> ProductCode.create
    |> checkProductCodeExists
    // returns a bool :(
```

但是此刻会产生一个问题。我们想要 toProductCode function 返回一个 ProductCode ，但是 checkProductCodeExists function 返回了一个 bool ，这意味着整个 pipeline 将返回一个 bool 。无论如何，我们需要 checkProductCodeExists 返回是一个 ProductCode 。哦，天哪，这是不是意味着我们必须改变设计规范? 不，很幸运，可以不用改现有的设计，让我们来看看如何实现。

### Creating Function Adapters

某个 function 返回 bool ，但是我们真的希望它返回 ProduceCode ，与其改变设计规范，不如创建一个 "adapter" function ，它以原始的 function 作为输入，产生一个新的 function ，这个新的 function 就可以返回你需要的类型了。  
![image](./../image/adapter-function.png)  

实现是显而易见的，定义一个 function ，参数是 bool-returning
predicate (checkProductCodeExists) 和 需要被校验的值 (productCode) ：
```
let convertToPassthru checkProductCodeExists productCode =
    if checkProductCodeExists productCode then
        productCode
    else
        failwith "Invalid Product Code"
```

有趣的是，编译器认为这个实现是 generic 的——它不涉及特定的 type，如果查看 function 的签名，你会看到没有出现特定的 ProductCode type ：
```
val convertToPassthru :
        checkProductCodeExists:('a -> bool) -> productCode:'a -> 'a
```
实际上，我们意外地创建了一个 generic adapter ，它将把任何 predicate function 转换成适合于 pipeline 的 “passthrough” function 。

把参数名称叫做 “checkProductCodeExists” 和 “productCode” 已经没有意义了，因为 function 是通用的。这就是为什么如此多的标准库函数具有如此短的参数名的原因，比如 f 和 g 表示函数参数，x 和 y 表示其他值。

让我们重构这个 function，使用更抽象的名称，像这样:
```
let predicateToPassthru f x =
    if f x then
        x
    else
        failwith "Invalid Product Code"
```
接着把硬编码的错误信息也参数化，最终版本如下：
```
let predicateToPassthru errorMsg f x =
    if f x then
        x
    else
        failwith errorMsg
```
注意，在参数顺序中，将 errorMsg 放在首位，这样可以使用 partial application 这种技术手段，将它 bake in (传入) function 的内部中去。

function 签名 ：
```
val predicateToPassthru : errorMsg:string -> f:('a -> bool) -> x:'a -> 'a
```
可以理解成：你给我一个 error message 和一个类型为 'a -> bool 的 function ，我将返回给你一个类型为 'a -> 'a 的 function 。因此，这个 predicateToPassthru function 是一个 “function transformer” ——您给它一个 function ，它将把它转换为另一个 function 。

这种技术在 functional programming 中非常常见，因此理解并能识别出这种模式非常重要。即使是 List.map 这个 function 也可以被看成是一个 “
function transformer” ——它将正常的 function 'a -> 'b 转换成可以操作 list 的 function 'a list -> 'b list 。

好的，现在让我们使用这个 generic function 来创建一个新版的 toProductCode :
```
let toProductCode (checkProductCodeExists:CheckProductCodeExists) productCode =
    // create a local ProductCode -> ProductCode function
    // suitable for using in a pipeline
    let checkProduct productCode =
        let errorMsg = sprintf "Invalid: %A" productCode
        predicateToPassthru errorMsg checkProductCodeExists productCode

    // assemble the pipeline
    productCode
    |> ProductCode.create
    |> checkProduct
```
现在针对 validateOrder ，已经实现了它的基本结构，在这个基础上可以继续进行构建。注意，更底层的校验逻辑，类似，“product 必须以 W 或 G 开头”，并没有出现在 validation functions (validateOrder) 里，而是内建 simple type (如 OrderId 和 ProductCode) 的构造函数中。使用 type 可以真正的增强我们对代码正确性的信心——我们可以从一个 UnvalidatedOrder 成功创建 ValidatedOrder 这一事实，意味着我们可以相信它是经过校验的!