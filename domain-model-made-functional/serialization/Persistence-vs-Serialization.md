## Persistence vs. Serialization

让我门从概念的定义开始，*persistence* 指的是一些状态——这些状态可以存活与进程之外，并且是由进程所创建的。*serialization* 指的是将特定于 domain 的表示 转换为 容易被持久化的表示(如，binary ，JSON 或 XML ) 的过程。

比如，每当有 “order form arrived” event 出现时，我们的 order-placing workflow 就会实例化并运行，但是当代码停止运行时，我们希望它的输出以某种方式被保留下来 (“被持久化”)，以便业务的其他部分可以使用该数据。“保留” 并不一定意味着存储在适当的 database 中——它可以存储为file 或 queue 。我们不应该对持久化数据的生命周期做任何假设——它可以只保存几秒钟 (比如在 queue 中)，也可以保存几十年 (比如在数据仓库中) 。

这一章，将重点关注 serialization ，下一章将会介绍 persistence 。