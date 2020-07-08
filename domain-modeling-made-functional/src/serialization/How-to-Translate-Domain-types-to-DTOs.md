## How to Translate Domain types to DTOs

domain type 的定义也许会相当的复杂，但是与之对应的 DTO type 必须是只包含 primitive type 的简单结构。那么，如何设计一个特定的 domain type 的 DTO 呢？ 我们来看一些指导原则。

### Single Case Unions

Single Case Unions ——在本书中我们称之为 “Simple Types” ——可以由 DTO 中的 primitive type 表示。

例如，如果 ProductCode 是 domain type ：
```rust
type ProductCode = ProductCode of string
```
相应的 DTO type 就是 string 。

### Options

对于 options ，将 None case 替换成 null 。如果这个 option type 包装的是一个 reference type ，那么我们不需要做任何改动，因为 null 是一个有效的值 。对于像 int 这样的 value type ，我们需要使用可空的等效值，比如 Nullable<int> 。

### Records

定义为 record 的 doamin type 可以在 DTO 中任就保持为一个 record type ，只要将每个 field 的 type 转换为等效的 DTO 即可。

这里有一例子用来演示 single-case unions ，optional values ， record type ：
```rust
/// Domain types
type OrderLineId = OrderLineId of int
type OrderLineQty = OrderLineQty of int
type OrderLine = {
    OrderLineId : OrderLineId
    ProductCode : ProductCode
    Quantity : OrderLineQty option
    Description : string option
}

/// Corresponding DTO type
type OrderLineDto = {
    OrderLineId : int
    ProductCode : string
    Quantity : Nullable<int>
    Description : string
}
```

### Collections

list ，sequence ，set 应该转换为 array ，每种序列化格式都支持 array 。
```rust
/// Domain type
type Order = {
    ...
    Lines : OrderLine list
}

/// Corresponding DTO type
type OrderDto = {
    ...
    Lines : OrderLineDto[]
}
```
对于 map 和其它复杂的 collection ，转换的方式依赖于序列化的格式。当使用 JSON 格式时，可以直接将 map 转换为 JSON object ，因为 JSON 只是 key-value collection 。

对于其它的序列化格式，也许你需要创建一个特殊的表现形式。比如，一个 map 可能会在 DTO 中被表示成一个 record array ，其中每个 record 是一个 key-value pair ：
```rust
/// Domain type
type PriceLookup = Map<ProductCode,Price>

/// DTO type to represent a map
type PriceLookupPair = {
    Key : string
    Value : decimal
}
type PriceLookupDto = {
    KVPairs : PriceLookupPair []
}
```
另外，map 也可以表示为两个并列的 array ，在反序列化时可以将它们压缩在一起。
```rust
/// Alternative DTO type to represent a map
type PriceLookupDto = {
    Keys : string []
    Values : decimal []
}
```

### Unions Used as Enumerations

多数情况下，Union 中每个 case 只是一个名称，没有额外的数据。这种情况它们可以被表示成 .NET enum ，序列化时通常被表示成整数。
```rust
/// Domain type
type Color =
    | Red
    | Green
    | Blue

/// Corresponding DTO type
type ColorDto =
    | Red = 1
    | Green = 2
    | Blue = 3
```
注意，在反序列化时，必须处理枚举值无效的情况。
```rust
let toDomain dto : Result<Color,_> =
    match dto with
    | ColorDto.Red -> Ok Color.Red
    | ColorDto.Green -> Ok Color.Green
    | ColorDto.Blue -> Ok Color.Blue
    | _ -> Error (sprintf "Color %O is not one of Red,Green,Blue" dto)
```
或者，也可以将 enum-style union 序列化成 string ，使用 case 的名称作为值。不过，将来如果涉及到重命名，那么这会成为一个问题。

### Tuples

tuple 不应该频繁的出现于 domain 中，但如果你这样做了，那么它们就需要一个特殊的 record type 来表示，因为很多序列化格式不支持 tuple 。下面的例子，domain type 是一个 tuple ，与之对应的 CardDto 是一个 record type 。
```rust
/// Components of tuple
type Suit = Heart | Spade | Diamond | Club
type Rank = Ace | Two | Queen | King // incomplete for clarity

// Tuple
type Card = Suit * Rank

/// Corresponding DTO types
type SuitDto = Heart = 1 | Spade = 2 | Diamond = 3 | Club = 4
type RankDto = Ace = 1 | Two = 2 | Queen = 12 | King = 13
type CardDto = {
    Suit : SuitDto
    Rank : RankDto
}
```

### Choice types

choice type 可以被表示为一个带有 “tag” 的 record ，这个 “tag” 表示 choice 里的那个 case 被使用了，然后为每个 case 提供一个 field ，该 field 包含与 case 相关的数据。当把某个 case 转换为 DTO 时，与这个 case 相关联的那个 field 将会包含数据，所有和其它的 case 关联的 field ，都将变为 null (或者对于类型为 list 的 field ，将变为 empty )。

> **! 提示**  
> 有些序列化器可以直接处理 F# 的 discriminated union type 。  
> 但是你将不能控制它们所用的序列化格式。如果另一个 bounded context 使用了不同的序列化器，并且也不知道如何解析这种格式，那么这会成为一个问题。因为 DTO 是契约的一部分，所以最好能够对序列化格式做明确的控制。

这里有一个 domain type ( Example ) 的例子，它包含四个 choice case ：
* An empty case ，tagged as A 。
* An integer ，tagged as B 。
* A list of strings ，tagged as C 。
* A name (using the Name type from above) ，tagged as D 。
```rust
/// Domain types
type Name = {
    First : String50
    Last : String50
}

type Example =
    | A
    | B of int
    | C of string list
    | D of Name
```
下面是与之对应的 DTO type ，每个 case 的 type 被替换为可序列化的版本： int 对应 Nullable， string list 对应 string[] ， Name 对应 NameDto 。
```rust
/// Corresponding DTO types
type NameDto = {
    First : string
    Last : string
}

type ExampleDto = {
    Tag : string // one of "A","B", "C", "D" 
    
    // no data for A case
    BData : Nullable<int>   // data for B case
    CData : string[]        // data for C case
    DData : NameDto         // data for D case
}
```
序列化操作就很简单了——你只需要转换与选定的 case 相关联的数据 ，并设置所有其他 case 的数据为null：
```rust
let nameDtoFromDomain (name:Name) :NameDto =
    let first = name.First |> String50.value
    let last = name.Last |> String50.value
    {First=first; Last=last}

let fromDomain (domainObj:Example) :ExampleDto =
    let nullBData = Nullable()
    let nullCData = null
    let nullDData = Unchecked.defaultof<NameDto>
    match domainObj with
    | A ->
        {Tag="A"; BData=nullBData; CData=nullCData; DData=nullDData}
    | B i ->
        let bdata = Nullable i
        {Tag="B"; BData=bdata; CData=nullCData; DData=nullDData}
    | C strList ->
        let cdata = strList |> List.toArray
        {Tag="C"; BData=nullBData; CData=cdata; DData=nullDData}
    | D name ->
        let ddata = name |> nameDtoFromDomain
        {Tag="D"; BData=nullBData; CData=nullCData; DData=ddata}
```
对这段代码做个简单的分析：
* 在 function 开头为每个 field 设置 null 值，然后将它们分配给与匹配的 case 无关的字段。
* 在 “B” case 中，不能直接将 null 赋予 Nullable<_> type 。必须使用 Nullable() function 代替。
* 在 “C” case 中，null 可以赋给 Array ，因为它是个 .NET class 。
* 在 “D” case 中，同样不能直接将 null 赋给 F# 的 record type ( 如，NameDto ) ，因此我们使用一个 “backdoor” function (这里是 Unchecked.defaultOf<_> ) 为它创建一个 null 值。这种方式不能在一般的代码中使用，只有在为序列化创建 null 值时可以使用。

当用这样的 tag 反序列化 choice type 时，我们匹配 “tag” field ，然后分别处理每种 case 。
尝试反序列化之前，必须总是对与 tag 相关联的数据做 null 值检查 ：
```rust
let nameDtoToDomain (nameDto:NameDto) :Result<Name,string> =
        result {
        let! first = nameDto.First |> String50.create
        let! last = nameDto.Last |> String50.create

        return {First=first; Last=last}
    }

let toDomain dto : Result<Example,string> =
    match dto.Tag with
    | "A" ->
        Ok A
    | "B" ->
        if dto.BData.HasValue then
            dto.BData.Value |> B |> Ok
        else
            Error "B data not expected to be null"
    | "C" ->
        match dto.CData with
        | null ->
            Error "C data not expected to be null"
        | _ ->
            dto.CData |> Array.toList |> C |> Ok
    | "D" ->
        match box dto.DData with
        | null ->
            Error "D data not expected to be null"
        | _ ->
            dto.DData
            |> nameDtoToDomain  // returns Result...
            |> Result.map D     // ...so must use "map"
    | _ ->
    // all other cases
        let msg = sprintf "Tag '%s' not recognized" dto.Tag
        Error msg
```
在 “B” 和 “C” 的 case 中，从 primitive 转换到 domain 是没有 error 的 (在确保数据不是 null 之后) 。在 “D” case 中，从 NameDto 转换到 Name 时可能会失败，因此 nameDtoToDomain function 返回 Result ，然后必须使用 Result.map D (这里的 D 是一个 constructor function ) 对 Result 做 map 遍历。

### Serializing Records and Choice Types Using Maps
对于复合类型 ( record 和 discriminated union )，另一种序列化方式是将所有内容序列化为 key-value map 。也就是，所有的 DTO 都将被实现为相同的类型——.NET type IDictionary<string,obj> 。这种方式特别适用于 JSON 格式，因为它与 JSON 对象模型非常一致。

这种方式的优点是 DTO 结构中没有隐含的 “契约” —— key-value map 可以包含任何内容——因此它有益于高度解耦的交互。缺点是根本就没有契约了！这意味着很难预期生产者和消费者之间是否匹配。有时候，一点点耦合是有用的。

来看一些代码。使用这种方式，可以像这样序列化一个 Name record ：
```rust
let nameDtoFromDomain (name:Name) :IDictionary<string,obj> =
    let first = name.First |> String50.value :> obj
    let last = name.Last |> String50.value :> obj
    [
        ("First",first)
        ("Last",last)
    ] |> dict
```
以上我们创建了一个 key/value pairs 的 list ，并使用内建的 dict function 以这个 list 为基础构建了一个 IDictionary 。如果之后将这个 dictionary 序列化成 JSON ，结果看起来和序列化一个 NameDto type 的一样。

值得注意的是， IDictionary 使用 obj 作为 value 。也就是说必须使用 upcast operator :> 将 record 的所有值显示的转换成 obj 。

对于 choice type ，返回的 dictionary 将只有一条数据，但是 key 的 value 将取决于 choice 。比如，对 Example type 做序列化，key 将是 “A” ，“B” ，“C” ，“D” 其中的一个。
```rust
let fromDomain (domainObj:Example) :IDictionary<string,obj> =
    match domainObj with
    | A ->
        [ ("A",null) ] |> dict
    | B i ->
        let bdata = Nullable i :> obj
        [ ("B",bdata) ] |> dict
    | C strList ->
        let cdata = strList |> List.toArray :> obj
        [ ("C",cdata) ] |> dict
    | D name ->
        let ddata = name |> nameDtoFromDomain :> obj
        [ ("D",ddata) ] |> dict
```
上面的代码显示了与 nameDtoFromDomain 类似的方式。对于每个 case ，将数据转换为序列化的格式，然后再将其转换为 obj 。在 “D” case 中，数据的类型是 Name ，它的序列化格式仅仅只是另一个 IDictionary 。

反序列化有点小小的麻烦。对于每个 field ，我们需要 (a)——在 dictionary 中查找它是否存在，(b)——如果存在，检索它并尝试将其转换为正确的 type 。

这需要一个 helper function ，我们称其为 getValue ：
```rust
let getValue key (dict:IDictionary<string,obj>) :Result<'a,string> =
    match dict.TryGetValue key with
    | (true,value) -> // key found!
        try
            // downcast to the type 'a and return Ok
            (value :?> 'a) |> Ok
        with
        | :? InvalidCastException ->
            // the cast failed
            let typeName = typeof<'a>.Name
            let msg = sprintf "Value could not be cast to %s" typeName
            Error msg
    | (false,_) ->  // key not found
        let msg = sprintf "Key '%s' not found" key
        Error msg
```
然后看看如何反序列化 Name ，首先需要获取 “First” 这个 key 的 value ( 可能是一个 error )。如果正常，对这个 value 调用 String50.create 以得到 First field ( 也可能是一个 error ) 。以类似的方式处理 “Last” key 和 Last field 。和往常一样，使用 result expression 来简化我们的工作。
```rust
let nameDtoToDomain (nameDto:IDictionary<string,obj>) :Result<Name,string> =
    result {
        let! firstStr = nameDto |> getValue "First"
        let! first = firstStr |> String50.create
        let! lastStr = nameDto |> getValue "Last"
        let! last = lastStr |> String50.create

        return {First=first; Last=last}
    }
```

反序列化 choice type (如，Example ) ，我们需要测试给定的一个 key 是否有与之对应的 case 。如果有，则尝试将其转换成 domain object 。同样，有很多潜在的 error ，所以对于每种 case ，都将使用一个 result expression 。
```rust
let toDomain (dto:IDictionary<string,obj>) : Result<Example,string> =
    if dto.ContainsKey "A" then
        Ok A    // no extra data needed
    elif dto.ContainsKey "B" then
        result {
            let! bData = dto |> getValue "B" // might fail
            return B bData
        }
    elif dto.ContainsKey "C" then
        result {
            let! cData = dto |> getValue "C" // might fail
            return cData |> Array.toList |> C
        }
    elif dto.ContainsKey "D" then
        result {
            let! dData = dto |> getValue "D" // might fail
            let! name = dData |> nameDtoToDomain // might also fail
            return name |> D
        }
    else
        // all other cases
        let msg = sprintf "No union case recognized"
        Error msg
```

### Generics

多数情况下，doamin type 是 generic type ，如果 serialization library 支持 generic type ，你就可以创建 generic type 的 DTO 。

例如，Result type 是 generic type ，所以可以被转换成 generic type 的 ResultDto ：
```rust
type ResultDto<'OkData,'ErrorData when 'OkData : null and 'ErrorData: null> = {
    IsError : bool // replaces "Tag" field
    OkData : 'OkData
    ErrorData : 'ErrorData
}
```
注意，'OkData and 'ErrorData 这两个 generic type 必须是 nullable 的，因为反序列化时它们有可能会有缺失的值。

如果 serialization library 不支持 generic type ，你就必须为每种 concrete type 创建一个 type 。这可能很乏味，但是你会发现在实践中，很少有 generic type 需要被序列化。

例如，以下是来自 order-placing workflow 中的 Result type ，使用 concrete type 将其转换成 DTO ，而不是使用 generic type 做转换。
```rust
/// PlaceOrderEventDto and PlaceOrderErrorDto is concrete type
type PlaceOrderResultDto = {
    IsError : bool
    OkData : PlaceOrderEventDto[]  
    ErrorData : PlaceOrderErrorDto
}
```