## Wrapping Up

现在我们已经修订了 pipeline 的实现代码，以 type-safe 的方式实现了 error-handling 和 async effect 。这个 placeOrder 的实现代码是很干净的，没有出现很丑陋的 error-handling 代码。是的，我们确实必须进行一些笨拙的转换以使所有的 type 正确匹配，但是这些额外的工作本身是值得的，因为这之后，我们可以确信所有的 pipeline 组件都能够组合在一起，而没有任何问题。

下一章，我们将实现 domain 和外部世界之间的交互：serialize 和 deserialize ，以及如何将状态持久化到 database 。