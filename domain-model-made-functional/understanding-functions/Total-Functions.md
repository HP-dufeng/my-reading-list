## Total Functions

数学函数将每个可能的输入与输出连接起来。在函数式编程中，我们尝试以同样的方式设计函数，以便每个输入都有一个对应的输出。这类函数被称为 **total functions** (全函数)。

何苦呢? 这是因为我们希望尽可能显示的表达事物( make things *explicit* )  ， 避免出现这样的 hidden effects 和 behavior ——就是这些 hidden effects 和 behavior 并没有在 type 签名中显示的表达出来。

用一个 很二 的例子来解释这个概念，twelveDividedBy 这个 function 返回 12 除以 输入参数 n  的结果，看看下面的伪代码：
```
let twelveDividedBy n =
    match n with
    | 6 -> 2
    | 5 -> 2
    | 4 -> 3
    | 3 -> 4
    | 2 -> 6
    | 1 -> 12
    | 0 -> ???
```
如果输入 0 ，那结果是什么呢？ 12 除以 0 结果是 undefined 。

暂时忽略每一个可能的输入都有一个对应的输出，只需要在 0 的情况下抛出一个异常：
```
let twelveDividedBy n =
    match n with
    | 6 -> 2
    ...
    | 0 -> failwith "Can't divide by zero"
```
但是，请再看看 function 的签名：
```twelveDividedBy : int -> int```

这个签名表达的意思是：传入一个 int 值，并且返回一个 int 值。但这是骗人的，签名说谎了！你并不总是得到一个 int 值，有时会得到一个 exception。签名中并没有表达出 有可能会返回一个 exception 这种情况。

如果类型签名不会说谎那就太好了。在这种情况下，function 的每个输入都有一个有效的输出，没有例外。来看看怎么做。

**一种做法是限制输入以消除不合法的值**。例如，可以创建一个新的 type NonZeroInteger ，限制这个 type 只能是 非 0 值，然后使用这个 type 作为输入。这样就永远不需要对 0 值做特别处理。
```
type NonZeroInteger =
    // Defined to be constrained to non-zero ints.
    // Add smart constructor, etc
    private NonZeroInteger of int

/// Uses restricted input
let twelveDividedBy (NonZeroInteger n) =
    match n with
    | 6 -> 2
    ...
    // 0 can't be in the input
    // so doesn't need to be handled
```
签名的新版本是：```twelveDividedBy : NonZeroInteger -> int```

这个要比前一个版本好一些。无需阅读文档或查看源代码，从签名信息就可以看到对输入参数的需求是什么。这个 function 没有说谎。所有信息都是显示表达的。

**另一种方式是 *extend the output*** ， 可以接受 0 作为输入值，但是将输出扩展为：可以在 有效的整型值 和 未定义的值 之间进行选择的 type 。我们将使用Option type 来表示这种 “something” 和 “nothing” 之间的选择。实现如下：
```
/// Uses extended output
let twelveDividedBy n =
    match n with
    | 6 -> Some 2 // valid
    | 5 -> Some 2 // valid
    | 4 -> Some 3 // valid
    ...
    | 0 -> None   // undefined
```
这时的 签名是：```twelveDividedBy : int -> int option```  
表达的意思是，输入一个 int，***也许*** 会返回一个 int 。同样，签名是明确的，不会误导您。

即使在这样一个 很二 的示例中，我们也可以看到使用 total function 的好处。这两种实现方式，function 签名明确说明了所有可能的输入和输出是什么。后面在 error handling 的章节里，我们将看到使用 function 签名来表达出所有可能的输出的真实的例子。
