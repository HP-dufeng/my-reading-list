# Consistency Boundary
---

在单体系统中，对于一致性似乎没有什么要求。为了实现一致性，大量的逻辑被外包给数据库引擎，从而变得隐式，很难一目了然，也很难测试。数据库事务经常用于确保一次执行多个状态的改变。如果数据变得不一致，这通常意味着失败，这需要进行大量的检查来解决问题。

**Domain-Driven Design (DDD)** 意味着避免复杂的实体图。相反，开发人员需要找到属于一个逻辑单元的实体的集合，它们需要一起更新以确保一致性。这样一组实体称为聚合。

本章包含下列主题：
* Command handling as a unit of work
* Consistency and transactions
* Aggregates and aggregate root patterns
* Constraints and invariants