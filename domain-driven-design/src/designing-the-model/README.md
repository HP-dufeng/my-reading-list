# Designing the Model
---

许多人认为 domain model（领域模型）就是 data model。通过在 google 上搜索 domain model，您可以很容易地看到这一点——您找到的所有东西都是数据图或类图。尽管有时类图也包含一些有用的行为（method），但这种情况并不多。然而，由于业务的复杂性很少体现在它的数据中，所以我们需要认识到行为是 domain model 不可分割的一部分。

EventStorming 可以帮助我们了解业务的整体或某个部分，但我们还需要进一步深入才能实现。这就是 Design-level EventStorming 的作用——我们着眼于系统中最有趣的部分，并深入其中，发现更多的事件和新的流程。

本章，我们将讨论以下主题：
* 领域模型表示什么？
* 模式与反模式
* Design-level EventStorming