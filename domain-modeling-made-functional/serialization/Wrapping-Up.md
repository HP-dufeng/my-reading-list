## Wrapping Up

这一章，我们暂时离开了 bounded context 和 clean domain ，进入了杂乱的基础设施的世界。学习了如何设计 Data Transfer Objects ，使用 DTO 充当 bounded context 和外部世界之间的中介，我们还了解了一些指导原则，帮助你编写自己的实现。

程序中的许多代码都需要与外部世界交互，序列化只是其中的一种。多数的应用程序需要与 database 通信。下一章，我们将把关注点转向 persistence ——如何使我们的 domain model 与 关系型 和 NoSQL 数据库一起工作。