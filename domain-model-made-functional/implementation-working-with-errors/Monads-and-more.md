## Monads and more

这本书试着避免使用过多的术语，但是有一个词是 functional programming 无法避免的—— **monad** 。因此我们先暂停一下，讨论一点关于 monad 的话题。monad 这个词的名声非常可怕，但是，事实上我们已经创建和使用过一个。

monad 只是一个编程模式，允许你将 “monadic” function 串联起来。  

好的， "monadic" function 又是什么呢？这个 function 接受一个 “normal” value，然后返回一个 “enhanced” value (像是包装进 Result type 的 value )，因此 monadic function 就是我们一直所研究那种 Result-generating “switch” function 。

技术上来说，monad 是一个简单的术语，由三个部分组成：
* 一个数据结构。
* 和这个数据结构相关的一些 function 。
* 一些关于 function 必须如何工作的规则。

这个数据结构在我们的例子中就是 Result type 。

要成为一个 monad ，这个数据结构必须要有两个 相关的 function ，return 和 bind ：
* return (也称做 pure) 这个 function 的功能是把 normal value 转换成 monadic type 。在我们的例子中，monadic type 就是 Result ，return 就是 Ok function 。
*  
