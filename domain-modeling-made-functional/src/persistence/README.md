# Persistence
---

在本书中，有意的将 domain model 设计成 “persistence ignorant (持久化无感知) ” 的——确保 实现数据存储的逻辑 或 service 之间的交互逻辑，不会干扰我们对 domain model 的设计。

但大多数应用程序，都会出现这样的情况：程序运行时的状态需要被保留，它需要存活于 进程 或 workflow 的生命周期之外。这时，就需要一种持久化机制，比如，file system 或 database 。遗憾的是，当我们从完美的 domain 转向混杂的基础设施世界时，几乎总是会出现不匹配的情况。

持久化以 domain 驱动设计的数据模型时，将会遇到很多的问题，本章的目的就是帮助你解决这些问题。我们先讨论一些高层原则，比如 command-query separation ，然后切换到低层实现。特别的，我们将看到如何以两种不同的方式来 持久化 domain model ： NOSQL 和 关系型 SQL database 。

本章之后，你就有了一些工具，这些工具可以帮助你 将数据库或其他持久化机制集成到应用程序中。

在深入探讨持久化机制之前，先来看一些指导原则 ，这些原则将指导我们在 domain-driven design 的上下文中更好的使用持久化。它们是：
* 把持久化推迟到 boundec context 的边界。
* 分离 command ( update ) 和 query ( read ) 。
* bounded context 必须有它自己的 data store 。


本章包含以下小节：

[Pushing Persistence to the Edges](./Pushing-Persistence-to-the-Edges.md)

[Command-Query Separation](./Command-Query-Separation.md)

[Bounded Contexts Must Own Their Data Storage](./Bounded-Contexts-Must-Own-Their-Data-Storage.md)

[Working with Document Databases](./Working-with-Document-Databases.md)

[Working with Relational Databases](./Working-with-Relational-Databases.md)

[Transactions](./Transactions.md)

[Wrapping Up](./Wrapping-Up.md)