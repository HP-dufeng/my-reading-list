# Evolving a Design and Keeping It Clean
---

从对 domain 的建模到实现，我们走完了整个过程，但是我们都知道这并不是故事的结尾。对于一个领域模型来说，一开始总是很干净和优雅的，但是随着需求的变化，模型开始变得混乱，各种子系统开始相互纠缠，并且难以测试。所以，这是我们最后的挑战： 能否在不让模型变成一个大泥球的情况下演进模型？

domain-driven design 并不是一个静态的过程。它是一个由 开发人员，领域专家和其他相关者 之间的 持续协作 的过程。因此，如果需求发生了变化，我们必须总是首先重新评估领域模型，而不是仅仅对实现修修补补。

这一章，我们将看到对需求的一些更改，首先跟踪这些变化，看看它们是否会影响我们对领域模型的理解，然后再对实现做出改变。此外，还将看到在设计中大量使用类型系统所带来的好处，这就是，我们会很有信心，当对模型进行更改时，代码不会意外地崩溃。


我们将看到四种变化：
* 在 workflow 中添加一个新的步骤。
* 改变 workflow 的输入。
* 更改领域中的一个关键类型 ( order ) 的定义，并观察其在系统中的影响。
* 将 workflow 转换为一个整体，以使其符合业务规则。









本章包含以下小节：

[Change 1: Adding Shipping Charges](./Change-1-Adding-Shipping-Charges.md)

[Change 2: Adding Support for VIP Customers](./Change-2-Adding-Support-for-VIP-Customers.md)

[Change 3: Adding Support for Promotion Codes](./Change-3-Adding-Support-for-Promotion-Codes.md)

[Change 4: Adding a Business Hours Constraint](./Change-4-Adding-a-Business-Hours-Constraint.md)

[Dealing with Additional Requirements Changes](./Dealing-with-Additional-Requirements-Changes.md)

[Wrapping Up](./Wrapping-Up.md)

[Wrapping Up the Book](./Wrapping-Up-the-Book.md)