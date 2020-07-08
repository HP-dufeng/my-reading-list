## Serialization

本书的示例，我们将 workflow 设计成具有输入和输出的 function ，其中输入来自 Command ，输出是 Event 。但是这些 command 来自那里？ event 又将去向何处？答案是它们依赖于 bounded context 之外的一些基础设施——消息队列、web请求，等等。

这些基础设施不需要知道 domain 的细节，因此必须将 domain model 中的 type 转换为基础设施能够理解的格式，例如 JSON、XML 或类似于 protobuf 的二进制格式<sup>1</sup>。

我们还需要某种方法来跟踪 workflow 所需的内部状态，比如 Order 的当前状态。这样，我们可能就会用到外部服务，比如数据库。

很明显，使用外部基础设施的一个重要方面是能够将 domain model 中的 type 轻松的 序列化 和 反序列化 。

因此，在本章中，将学习如何做到这一点：我们将了解如何设计能够被 序列化 的 type ，然后将了解如何将 domain model 与这些 type 进行转换。

本章包含以下小节：

[Persistence vs. Serialization](./Persistence-vs-Serialization.md)  

[Designing for Serialization](./Designing-for-Serialization.md)  

[Connecting the Serialization Code to the Workflow](./Connecting-the-Serialization-Code-to-the-Workflow.md)  

[A Complete Serialization Example](./A-Complete-Serialization-Example.md)  

[How to Translate Domain types to DTOs](./How-to-Translate-Domain-types-to-DTOs.md)  

[Wrapping Up](./Wrapping-Up.md)  


---
1.  https://developers.google.com/protocol-buffers/