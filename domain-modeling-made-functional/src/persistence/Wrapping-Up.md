## Wrapping Up

这一章，我们从 persistence 的一些高层原则开始： 将 query 与 command 分离，将 I/O 推迟到边界，并确保有 bounded context 拥有自己的 data store 。然后深入到与 relational database 交互的底层机制。

这将我们带到了本书第三部分的结尾。当 设计和创建 一个 bounded context 的完整实现时，我们已经有了所有需要的工具： 
* the pure types and functions inside the domain ( 第9章 [Implementation: Composing a Pipeline]() )，
* the error handling ( 第10章 [Implementation: Working with Errors]() )，the serialization at the edges ( 第11章 [Serialization]() )，
* the database for storing state ( 本章 )。

但是我们真的还没有全部完成。正如一句军事谚语所说: “no plan survives contact with the enemy” 。因此，当我们学习新的东西，需要改变设计时，会发生什么呢？ 这将是下一章，也是最后一章的主题。