## Functions, Functions, Everywhere

首先，让我们看看为什么 **functional programming** 与 **object-oriented programming** 是如此的不同，functional programming 的定义有很多，但我要选择一个简单的：  
* Functional programming is programming as if functions really mattered  

许多高级语言的 functions 是 first-class objects，但是使用这些 functions(or lambdas) 并不代表你就在以 functional programming 的理念编程。functional programming 的关键特征是： **functions are used everywhere, for everything** 。  

来看一个例子，假如一个大型的程序是由许多 小的部件 组成。  
* 从 orject oriented 的角度看，这些 小的部件 是 **classes** 和 **objects**
* 从 functional 的角度看，这些 小的部件 是 **functions**   

又假如说我们需要 参数化程序的某个部分，或者需要解耦组件。  
* 从 orject oriented 的角度看，需要使用 **interfaces** 和 **dependency injection** 。
* 从 functional 的角度看，需要使用 **function** 来参数化  

又假如说我们需要遵循 “**don’t repeat yourself**” 的原则，复用组件
* 从 orject oriented 的角度看，可能会使用 **inheritance** ，或技术上称为 **Decorator** 的模式。
* 从 functional 的角度看，会把可复用的代码放入functions中，再以 composition 的 方式把这些functions粘合在一起  

重要的是要理解 **functional programming** 不仅仅是语法上的不同，这是一种完全不同的编程思维方式。  

如果你是个 functional programming 的新手，你必须以初学者的心态去学习。  
也就是说，不要试图从 object oriented 的编程范式来提出问题(例如：“how do I loop through a collection” or “how do I implement the Strategy pattern”)，更好的方式是找出这些问题的潜在需求是什么(例如，“how can I perform an action for each element of a collection,” “how can I parameterize behavior”)