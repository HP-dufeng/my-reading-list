## Working With Simple Types

实现 workflow 的每个步骤之前，得先有一些 “simple types”，如： OrderId, ProductCode, ...等。

大部分 type 有一些约束条件，我们会按照第 6 章 Integrity of Simple Values 这一节里介绍的方法，来实现这些 type 。

所以，对于每个 simple type ，都需要至少两个 function ： 
* *create* function ， 它以 primitive type (如 string, int) 为基础，来构造这些 simple type 。
* *value* function ，提取内部的 primitive type 的值。

一般将这些 helper function 和 simple type 的定义放在同一个文件里，
为这些 helper function 定义一个 module ，这个 module 的名字和 simple type 的名字相同。请看下面的例子：
```
module Domain =
type OrderId = private OrderId of string

    module OrderId =
        /// Define a "Smart constructor" for OrderId
        /// string -> OrderId
        let create str =
            if String.IsNullOrEmpty(str) then
                // use exceptions rather than Result for now
                failwith "OrderId must not be null or empty"
            elif str.Length > 50 then
                failwith "OrderId must not be more than 50 chars"
            else
                OrderId str

        /// Extract the inner value from an OrderId
        /// OrderId -> string
        let value (OrderId str) = // unwrap in the parameter!
            str                   // return the inner value
```
* create function ，因为我们现在不使用 effect ，所以对错误使用了 exception (failwith)，而不是返回一个具有 effect 的 wrapper type —— Result 。
* value function ，演示了如何获取内部的 primitive type 的值。


