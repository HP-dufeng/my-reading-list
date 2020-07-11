## Transactions

到目前为止，所有代码的形式都是 “one-aggregate = one transaction” 。但在很多情况下，有许多东西需要原子地一起保存——要么全部保存，要么全部不保存。

一些 data store 的 API 支持事务。可以在同一个事务中调用多个 API ，如下：
```rust
let connection = new SqlConnection()
let transaction = connection.BeginTransaction()

// do two separate calls to the database
// in the same transaction
markAsFullyPaid connection invoiceId
markPaymentCompleted connection paymentId

// completed
transaction.Commit()
```

有些 data store 的事务只支持单个 connection 内的操作。在实践中，这意味着将不得不在一个调用中合并多个操作，像这样：
```rust
let connection = new SqlConnection()
// do one call to service
markAsFullyPaidAndPaymentCompleted connection paymentId invoiceId
```

然而有时候，要与不同的 service 进行通信，但是没有办法进行 跨 service 的事务处理。

Gregor Hohpe 有篇文章： [Starbucks Does Not Use Two-Phase Commit]()<sup>9</sup> ——在第6章 [Consistency Between Different Contexts]() 这一节提到过，文章中指出业务通常不需要跨不同系统的事务，因为开销和协调成本太大而且太慢。相反，我们假设在大多数情况下事情会进展顺利，然后使用 协调进程 来检测不一致，并使用 compensating transaction ( 事务补偿 ) 来纠正错误。

例如，下面是一个简单的 compensating transaction ，它回滚一个 database 的更新操作：
```rust
// do first call
markAsFullyPaid connection invoiceId
// do second call
let result = markPaymentCompleted connection paymentId

// if second call fails, do compensating transaction
match result with
| Error err ->
// compensate for error
unmarkAsFullyPaid connection invoiceId
| Ok _ -> ...
```


---
9.  http://www.enterpriseintegrationpatterns.com/ramblings/18_starbucks.html