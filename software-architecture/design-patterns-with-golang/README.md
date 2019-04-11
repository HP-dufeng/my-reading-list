# Design Patterns
> 老生常谈的话题，咱在参加工作之前就熟悉的话题。  
> 借着 golang，再回顾以下。  

请记住：  
**Go isn't exactly object oriented language and promotes simplicity.**  
所以，也不能生搬硬套 那些设计模式的概念，首先要理解这些模式要解决的问题，再想想如何以go的方式来解决这些问题，我想，这个才是重点。

## Design principles
>关键人物：Robert C Martin (Uncle Bob)  
关键词：SOLID

- **S** Single Responsibility Principle  
`"One class should have one, and only one, responsibility."`
- **O** Open/Closed Principle  
`"You should be able to extend a class's behavior without modifying it."`
- **L** Liskov Substitution Principle  
`"Derived types must be substitutable for their base types."`
- **I** Interface Segregation Principle  
`"Many client-specific interfaces are better than one general-purpose interface."`
- **D** Dependency Inversion Principle  
`"Depend on abstractions, not on concretions."`

### 简单解释一下：
 - **S** , seems obvious, nothing to explain.
 - **O** , classes should be open for extension but closed for modification, so it should be possible to extend or override class behavior without having to modify code.
 - **L** , derived classes must be usable through the base class interface without the need for the client to know the specific derived class.
 - **I** , it is a common symptom for base classes to be a *collect-all* for behavior. However, this makes the whole code  brittle: derived classes have to implement methods that don't make sense to them. Clients also can get confused by this variable nature of derived classes. To avoid this, this principle recommends having an interface for each type of client.
 - **D** , higher-level modules should depend only on interfaces and not on concrete implementations.

 借着这些设计准则，来看看设计模式。这些设计模式分为三大类：
 - Creational
 - Structural
 - Behavioral

 ## 设计模式
 - **Creational**
    - [Factory method](factory-method.md)
    - Builder
    - Abstract factory
    - Singleton
- **Structural**
    - Adaptor
    - Bridge
    - Composite
    - Decorator
    - Facade
    - Proxy
- **Behavioral**
    - Command
    - Chain of Responsibility
    - Mediator
    - Memento
    - Observer
    - Visitor
    - Strategy
    - State
