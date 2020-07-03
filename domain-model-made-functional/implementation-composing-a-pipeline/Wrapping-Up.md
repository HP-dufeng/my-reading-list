## Wrapping Up

本章，我们将重点放在了实现 pipeline 中的步骤和处理依赖项上。每个步骤的实现都限制在只做一个增量转换，并且很容易隔离测试。

当组合实现这些步骤的 function 时，type 不总是相匹配的，因此我们引入了三个重要的 functional programming 技术：
* **adapter function** ， 创建一个 “adapter function” 将 function 的输入或输出做一下转换。比如，我们把 checkProductCodeExists 的输出从 bool 转换成了 ProductCode 。
* **lifting** ，将不同的 type “提升” 为 common type 。比如，我们的例子中，将所有 event 都转换成了 common PlaceOrderEvent type 。
* **partial application** ，使用 partial application 将依赖项 inject 到 function 中，这样可以对 function 调用者隐藏实现的细节，组合 function 时更容易。

后续章节还会使用到这些技术。

有一个问题还没有解决。本章暂时去掉了那些 *effect* ，作为代替，我们使用了 exception 来处理错误信息。虽然很方便，但这会产生具有欺骗性的 function signature，而不是我们喜欢的更明确的 signature ，这样对于 “代码即文档” 来说是一种灾难。

下一章，会修复这个问题。我们将把所有 Result type 添加回 function signature，并学习如何使用它们。