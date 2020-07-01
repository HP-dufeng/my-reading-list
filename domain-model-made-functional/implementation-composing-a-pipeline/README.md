## Implementation: Composing a Pipeline

到目前为止，我们已经花费了大量时间，只是使用 type 对 domain 进行了建模。但是还没有做任何的实现！不要着急，马上就开始。

后面的两个章节，我们将使用 functional 的方式实现第7章 (Modeling Workflows as Pipelines) 中的设计方案。

回顾以下第7章的设计，workflow 可以被看作是一系列的 document transformations —— a pipeline —— pipeline 中的每一个步骤都被设计为 “pipe” 的一小节。

从技术角度来看，pipeline 中有以下几个阶段：
* 从 UnvalidatedOrder 开始，将其转换成 ValidatedOrder ，如果出现验证错误，则返回 error 。
* 将验证步骤 (ValidatedOrder) 中的输出，添加一些额外的信息后，转换成 PricedOrder 。
* 获取定价步骤 (PricedOrder) 的输出，以输出的信息为基础创建一封确认信，并发送它。
* 创建一组 event ，这些 event 表示了所发生的事情，然后返回这些 event 。

我们希望将这些转换成代码的同时，保留原始的需求，而不与技术实现细节混杂在一起。

看看下面的代码，这个例子使用了第8章 Composition 这一节中，所介绍的 pipeline 的技术，将每一个步骤中的 function 连接起来：
```
let placeOrder unvalidatedOrder =
    unvalidatedOrder
    |> validateOrder
    |> priceOrder
    |> acknowledgeOrder
    |> createEvents
```
即使是非程序员，也很容易理解上面代码中的——一系列步骤。来看看怎样实现它。实现它需要两个部分：创建每一个步骤，然后将它们组合起来。

首先，把 pipeline 中的每个步骤实现成标准的 function ，确保 function 是 stateless 的，没有 side-effects，这样就可以独立地对其进行测试。

接下来，将这些较小的 function 组合成一个较大的 function 。好像很简单，但正如我们之前提到的，真正尝试组合这些 function 时，我们会遇到一些问题。这些 function 不能很好地组合在一起——一个函数的输出与下一个函数的输入不匹配。为了解决这个问题，我们需要学习如何操作每个步骤的输入和输出，以便它们可以组合。

function 不能组合的原因有两个：
* function 拥有额外的参数，这些参数不是 pipeline 的一部分，但却是实现所必需的——我们称之为 “dependencies” 。
* 在 function 签名中使用类似 Result 这样的 wrapper type，显式地指出了 function 具有的一些 “effect”，比如错误处理。但是像这样的对输出做了 wrapper 的 function 是不能直接连接到 仅将 unwrapped plain data 作为输入的 function 的。

在这一章，我们将处理第一个问题，处理作为依赖项的输入，并且我们将看到如何使用 functional 的方式实现 “dependency injection” 。下一章再讨论如何处理具有 effect 参数的 function 。

> **！我的提示：**  
> **effect** : 在 functional programming 的世界里，通常是指，将某个 plan data（原始类型的值，比如： int，string ...）包装一下，将其提升为具有一定 effect 的值。这里不将 *effect* 翻译为 *作用*，因为 *作用* 这个词不能代表上面的含义。*effect* 在我看来是一个术语。

编写实际的代码时，第一步先忽略这些具有 *effect* 的 wrapper type，比如，Resut 和 Async ...， 等等，而只是实现所有的步骤。我们将关注点放在如何做最基础的组合上。