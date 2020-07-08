## Working with Domain Errors

软件系统是复杂的，不能使用 type 来建模所有可能出现的错误，我们也不想这样做。因此，让我们把有可能出现的错误分分类。

可以把 error 分成以下三类：
* **Domain Errors** ，这些 error 预期是业务流程的一部分，因此必须包含在 domain 的设计中。比如，被拒绝的 order ，或者 order 包含无效的 product code 。实际业务场景中已经有处理这类事情的方案，因此代码需要反映出这些方案。
* **Panics** ，这些 error 会使系统处于 未知状态，比如，系统无法处理的 error (e.g. “divide by zero,” “null reference” )。
* **Infrastructure Errors** ，这些 error 预期是 architecture 的一部分，但和业务流程无关，所以没有包含在 domain 里。比如，network timeout ，authentication failure 。

有时无法清晰的界定是否是 domain error 。这是你可以请教熟悉 domain 业务人员。
```rust
    You： Hi Ollie. 问个小问题，如果我们在访问负载均衡器时遇到连接中断，你会关心这个问题吗?
    Ollie: ????
    You: 好的，我们把它叫做 infrastructure error ，然后告诉用户稍后再试一次。
```
不同的 error 需要不同的实现方式。

domain error 是 domain 的一部分，就像其它的部分一样，需要被合并到 domain model 里，与领域专家一起讨论它，并且如果可能的话使用 type system 对其文档化。

panics ，处理它的最佳方式是放弃正在执行的 workflow ，并抛出 exception ，然后在程序的最高层 (例如 main function ) 捕获 exception 。请看例子：
```rust
/// A workflow that panics if it gets bad input
let workflowPart2 input =
    if input = 0 then
        raise (DivideByZeroException())
    ...

/// Top level function for the application
/// which traps all exceptions from workflows.
let main() =
    // wrap all workflows in a try/with block
    try
        let result1 = workflowPart1()
        let result2 = workflowPart2 result1
        printfn "the result is %A" result2

    // top level exception handling
    with
    | :? OutOfMemoryException ->
        printfn "exited with OutOfMemoryException"
    | :? DivideByZeroException ->
        printfn "exited with DivideByZeroException"
    | ex ->
        printfn "exited with %s" ex.Message
```

infrastructure error 可以使用上述任何一种方法来处理。具体的选择取决于所使用的 architecture 。如果代码由许多小 service 组成，那么使用 exception 可能会更清晰，但如果是一个更独立的应用程序，你可能希望使 error handling 更显式。事实上，采用和 domain error 相同的方式处理许多 infrastructure error 通常会很有益处，因为这将迫使作为开发人员的我们去思考什么会导致出错。确实，在某些情况下，这种类型的 error 需要和领域专家讨论。例如，如果 remote address validation service 不可用，业务流程应该如何更改？我们应该告诉客户什么？
这样的问题开发人员是不能回答的，由领域专家和产品负责人考虑比较合适。

本章的剩余部分，将只关注那些我们想要作为 domain 的一部分而显式建模的 error 。和 panic 有关的 error 应该只抛出 exception ，并由 top-level function 捕获。 

### Modeling Domain Errors in Types

当为 domain 建模时，我们避免使用 primitive type (e.g. string ) 。作为代替，我们使用和 domain 相关的词汇 (Ubiquitous Language) ， 创建特定于 domain 的 type 。

error 应该以同样的方式建模。如果在关于 domain 的讨论中出现了某些类型的 error ，那么应该像对待 domain 中的其他东西一样对它们进行建模，一般会把 error 设计成 choice type 。

例如，一下为 order-placing workflow 设计的 error 模型 ：
```rust
type PlaceOrderError =
    | ValidationError of string
    | ProductOutOfStock of ProductCode
    | RemoteServiceError of RemoteServiceError
    ...
```

* ValidationError 校验 property ，比如，任何长度或格式的错误。
* ProductOutOfStock ，当客户试图购买缺货产品时的错误。可能会有一个特殊的业务流程来处理这个 error 。
* RemoteServiceError ，这是一个关于 infrastructure error 的例子。比起只是抛出 exception ，我们可以通过，比如说，在放弃之前重新尝试一定的次数来处理这个 error 。


任何与错误相关的额外信息也会被明确显示出来。此外，随着需求的变化，扩展(或收缩) choice type 不仅很容易，而且也是安全的，因为编译器会在 pattern matches 错过某个 case 时产生警告。  
使用 choice type 建模 error ，代码即文档，又一次，very nice !

第7章设计 workflow 时，我们知道可能会发生 error ，但当时并没有深入研究它们。这是有意的。在设计阶段没有必要预先定义所有可能出现的 error 。通常，在开发阶段 error 会逐步浮现出来，然后你可以考虑是否将它们视为 domain error ，如果是，就将它们添加到 choice type 中。

当然，向 choice type 添加一个新的 case ，可能会在某些代码中产生警告，警告你没有处理所有的case 。这样很好，因为这会迫使你与领域专家或产品负责人讨论如何处理这种情况。当像这样使用choice type 时，很难出现忽略掉某种 case 的情况。

### Error Handling Makes Your Code Ugly

exception 也不是一无是处，它可以使您的 “happy path” 代码保持干净。例如，前一章定义的 validateOrder function (伪代码)：
```rust
let validateOrder unvalidatedOrder =
    let orderId = ... create order id (or throw exception)
    let customerInfo = ... create info (or throw exception)
    let shippingAddress = ... create and validate shippingAddress...
    // etc
```
如果每一步都返回 error ，代码会比较丑。每个潜在的 error 都会有一个 case 语句，和 try/catch 一样捕获潜在的 exception 。下面是一些伪代码：
```rust
let validateOrder unvalidatedOrder =
    let orderIdResult = ... create order id (or return Error)
    if orderIdResult is Error then
        return

    let customerInfoResult = ... create name (or return Error)
    if customerInfoResult is Error then
        return

    try
        let shippingAddressResult = ... create valid address (or return Error)
        if shippingAddress is Error then
            return
        // ...
    with
        | ?: TimeoutException -> Error "service timed out"
        | ?: AuthenticationException -> Error "bad credentials"
    // etc
```
这种方式的问题是，现在三分之二的代码都用于 error handling ——原来简单干净的代码现在已经被破坏了。我们面临一个挑战:如何引入正确的 error handling ，同时保持 pipeline 模型的优雅?