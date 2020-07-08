# Implementation: Working with Errors
---

如果 product code 格式错误，或者 customer name 太长，或者 address validation service 超时，出现了这些情况，怎么办? 任何系统都会产生 error ，怎么处理这些 error 就显得很重要了。consistent 和 transparent 对于任何 production-ready system 都是至关重要的。

前一章，为了将重点放在 composition 和 dependency 上，我们有意的从 pipeline 的每一个步骤中去掉了 error *effect* ( Result type ) 。

但是这些 *effect* 是必要的，这一章，会把 Result 添加回 type signature ，学习如何使用它。

更一般的，我们将探讨如何使用 functional 的方式，优雅的处理 error ，再也不要编写 try/catch 这种丑陋的代码。以及为什么应该将某些类型的 error 视为 *domain* error ，并将其置于和 domain-driven design 的其他部分同等重要的位置。

本章包含以下小节：

[Using the Result Type to Make Errors Explicit](./Using-the-Result-Type-to-Make-Errors-Explicit.md)  

[Working with Domain Errors](./Working-with-Domain-Errors.md)  

[Chaining Result-Generating Functions](./Chaining-Result-Generating-Functions.md)  

[Applying “bind” and “map” to Our Pipeline](./Applying-bind-and-map-to-Our-Pipeline.md)  

[Adapting Other Kinds of Functions to the Two-Track Model](./Adapting-Other-Kinds-of-Functions-to-the-Two-Track-Model.md)  

[Making Life Easier with Computation Expressions](./Making-Life-Easier-with-Computation-Expressions.md)  

[Monads and more](./Monads-and-more.md)  

[Adding the Async Effect](./Adding-the-Async-Effect.md)  

[Wrapping Up](./Wrapping-Up.md)  