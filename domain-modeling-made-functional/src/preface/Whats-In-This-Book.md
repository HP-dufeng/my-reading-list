## What’s In This Book?

这本书分成三个部分：
* Understanding the domain
* Modeling the domain
* Implementing the model

每个部分都建立在前一部分的基础上，所以最好是按顺序阅读。

在**第一部分** “Understanding the Domain” 中，我们将研究 Domain-Driven Design 背后的思想，以及在团队中共享领域知识的重要性。还将简要介绍有助于构建这种共享领域知识的技术，如 event storming ( 事件风暴 )，然后我们会着眼于将一个较大的领域 分解成 可以独立实现和演进的较小的组件。

先说明一下，本书并不是要彻底地探讨 domain-driven design 。这会是一个很大的主题，已经有许多优秀的书籍和网站对此进行了详细的介绍。相反，本书的目的是向你介绍如何将 domain-driven design 和 functional programming 结合起来进行函数式领域建模，因此，我们不会太深入讨论 DDD 这个主题，我们会停留在一个较高的层次，强调两件事：(a) 与领域专家和其他非技术团队成员交流的重要性，(b) 基于真实的概念构建共享的领域模型的价值。

在**第二部分** “Modeling the Domain” 中，我们将从领域中提取一个工作流，并以 functional 的方式对其建模。我们将看到使用 functional 的方式来解耦工作流的组件与使用 object-oriented 的方式的有何不同，并且将学习使用 type 系统来捕获领域需求。最后，我们会编写既简洁又有表现力的代码，这些代码具有两个重要的特征：首先，这些代码就可以作为 可读性非常好的 关于领域的文档，其次，这些代码还可以作为一个可编译的框架 ( 其他的功能实现可以在此基础上进行构建 ) 。

在**第三部分** “Implementing the Model” 中，我们将实现这些模型。实现的过程中，我们将学习到如何使用常见的 functional programming 技术，如composition ，partial application ，和 听起来好像很恐怖的 “monad” 。


这本书并不打算成为 functional programming 的一本大全。我们将只讨论那些与 领域建模和实现模型 所相关的内容，而不会讨论更高级的技术。然而，在第三部分结束时，你将熟悉所有最重要的 functional programming 的概念，并且你将获得一个可以应用于大多数编程场景的工具箱。

正如太阳照常升起，需求也会更改，因此在最后一章，我们将看到领域有可能演进的方向，以及我们的设计如何适应这些变化。