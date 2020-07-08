## Using the Result Type to Make Errors Explicit

functional programming 的重点是尽可能显示的表达事物 ( making things *explicit* ) ，这也适用于 error handling 。function 要明确的表达出调用结果是成功，还是失败，如果失败，失败的原因是什么。

通常的情况是，我们都将 error 视为二等公民来对待。但是为了创建一个 robust ， production-worthy system ，则必须将 error 视为一等公民。对于 domain error 更应如此。

前一章，使用了 exception 抛出 error 。虽然方便，但这也意味着所有的 function signature 都具有误导性。比如，某个校验 address 的 function ：
```rust
type CheckAddressExists =
    UnvalidatedAddress -> CheckedAddress
```
这个 signature 对你理解代码的逻辑没有任何帮助，因为它并没有标记出可能会出现什么错误。相反，这里需要一个 *total function* (参见第8章 Total Functions 小节) ，所有可能的输出结果都由 type signature 明确的标记。在第4章 Modeling Errors 这一节，学习过可以使用 Result type 清晰的表达 function 成功或是失败，function signature 如下：
```rust
type CheckAddressExists =
    UnvalidatedAddress -> Result<CheckedAddress,AddressValidationError>

and AddressValidationError =
    | InvalidFormat of string
    | AddressNotFound of string
```
这个 signature 告诉我们：
* 输入参数是 UnvalidatedAddress 。
* 如果校验成功，输出 CheckedAddress 。
* 如果校验失败，输出 AddressValidationError 。

这展示了完全可以将 function signature 当作文档。这才是真的 代码即文档 。如果另外一个开发者要使用这个 function ，他只需要看 function signature 就可以知道 function 所做的事情，不用再去查看需求文档了。