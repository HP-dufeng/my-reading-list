## Monads and more

这本书试着避免使用过多的术语，但是有一个词是 functional programming 无法避免的—— **monad** 。因此我们先暂停一下，讨论一点关于 monad 的话题。monad 这个词的名声非常可怕，但是，事实上我们已经创建和使用过一个。

monad 只是一个编程模式，允许你将 “monadic” function 串联起来。  

好的， "monadic" function 又是什么呢？这个 function 接受一个 “normal” value，然后返回一个 “enhanced” value (像是包装进 Result type 的 value )，因此 monadic function 就是我们一直所研究那种 Result-generating “switch” function 。

技术上来说，monad 是一个简单的术语，由三个部分组成：
* 一个数据结构。
* 和这个数据结构相关的一些 function 。
* 一些关于 function 必须如何工作的规则。

这个*数据结构*在我们的例子中就是 Result type 。

要成为一个 monad ，这个数据结构必须要有两个 *相关的 function* ，return 和 bind ：
* return (也称作 pure ) 这个 function 的功能是把 normal value 转换成 monadic type 。在我们的例子中，monadic type 就是 Result ，return 就是 Ok function 。
*  bind (也称作 flatMap ) 它将 monadic function 串接在一起。在我们的例子中就是 Result -generating function 。本章的前面已经看到过如何为 Result 实现 bind 。

关于 function 必须如何工作的*规则* 被称作 “monad laws” ，这听起来有点吓人，但实际上这是常识性的指导方针，用来确保这些 function 的实现是正确的，不做任何怪异的事情。这里不会深入讲解 monad laws ——你可以在互联网上检索相关的内容。

所以，这就是 monad 的全部。我希望你能明白，它并不像你想象的那么神秘。

### Composing in Parallel With Applicatives

好的，既然谈到了 monad ，让我们也谈谈一个和它相关的模式叫做 “applicative” 。

applicative 和 monad 相似，但是 monad 是将 monadic function *串联*在一起，而 applicative 是将 monadic value *并行*的组合在一起。例如，如果做校验，应该用 applicative 的方式组合所有的 error ，而不应该只保留第一个 error 。可以到 fsharpforfunandprofit.com <sup>2</sup> 更详细的介绍。

本书不会过多的使用术语 ”monad“ 或者 ”applicative“ ， 但是此刻，如果你真的碰到它们，你会知道它们所表到的意思是什么。

> **术语 提示**  
> 以下是在本章中所介绍的术语，以供参考：  
> * 在关于 error-handling 的内容里，介绍了 *bind* function 将 Result-generating function 转换成 two-track function 。它用来将 Result-generating function 串联在一起。更一般的，bind function 是 monad 的关键组件之一。  
> * 在关于 error-handling 的内容里，介绍了 *map* function 将 one-track function 转换成two-track function 。  
> * *monadic* ，这种组合方式是指，使用 bind 将 function 串联在一起。  
> * *applicative* ，这种组合方式是指，将 结果值 并行的组合在一起。


---
> 2. https://fsharpforfunandprofit.com/posts/elevated-world-3/#validation