## Other Approaches To Domain Modeling

这本书主要关注的是领域建模的 “主流” 方式——定义 data structure 和 作用于它们的 function ，但是其他方式可能在某些情况下更适用。我将在这里提到其中的两个，以备你进一步探索。
* 如果领域是围绕 semi-structured ( 半结构化 ) 数据进行的，那么本书中讨论的那种严谨型的模型就不合适了，更好的方法是使用灵活的数据结构，比如 map (又称为 dictionary ) 来存储 key-value pairs 。Clojure 社区在这方面有很多非常好的实践。
* 如果领域的重点是将元素组合在一起形成其他元素，那么在将关注点放在数据之前，先关注这些组合规则 ( 所谓的 “algebra” ) 通常是有益的。像这样的领域很普遍，从 金融合同 到 图形设计工具，“composition everywhere” ( 无处不组合 ) 的原则使它们特别适合用 functional 的方式建模。sorry ，由于篇幅的限制，本书不会介绍这类领域。