## A Complete Serialization Example

下面用一个例子来演示 domain object 和 JSON 之间的转换。假设要持久化一个 domain type ，将其命名为 Person ：
```
module Domain = // our domain-driven types
    /// constrained to be not null and at most 50 chars
    type String50 = String50 of string

    /// constrained to be bigger than 1/1/1900 and less than today's date
    type Birthdate = Birthdate of DateTime

    /// Domain type
    type Person = {
        First: String50
        Last: String50
        Birthdate : Birthdate
    }
```

String50 和 Birthdate 不能被直接序列化，所以首先创建一个与之对应的 DTO type —— Dto.Person ( 位于 Dto module 中的 Person ) ，这个 dto 的所有字段的类型都是 primitive 的：
```
/// A module to group all the DTO-related
/// types and functions.
module Dto =
    type Person = {
        First: string
        Last: string
        Birthdate : DateTime
    }
```

接下来，需要两个 function ，“toDomain” 和 “fromDomain” 。这些 function 和 DTO type 相关，和 domain type 无关，这是因为 domain 不应该关心 DTO 的实现细节。因此将它们放入到 Dto module 中的命名为 Person 的子 module 中 ：
```
module Dto =
    module Person =
        let fromDomain (person:Domain.Person) :Dto.Person =
        ...
        let toDomain (dto:Dto.Person) :Result<Domain.Person,string> =
        ...
```
为了保持一致性，这里使用了成对的 fromDomain 和 toDomain 。

我们从 fromDomain function 开始，它将 domain type 转换成 DTO 。这个 function 总是会成功 ( 不需要 Result )，因为一个复杂的 domain type 总是可以转换成一个 DTO ，而不会产生 error 。
```
let fromDomain (person:Domain.Person) :Dto.Person =
    // get the primitive values from the domain object
    let first = person.First |> String50.value
    let last = person.Last |> String50.value
    let birthdate = person.Birthdate |> Birthdate.value

    // combine the components to create the DTO
    {First = first; Last = last; Birthdate = birthdate}
```

反之，toDomain function 将 DTO 转换成 domain type ，因为有各种的校验和约束可能会出错，所以 toDomain function 返回的是 Result<Person, string> ，而不是一个单纯的 Person 。
```
let toDomain (dto:Dto.Person) :Result<Domain.Person,string> =
    result {
        // get each (validated) simple type from the DTO as a success or failure
        let! first = dto.First |> String50.create "First"
        let! last = dto.Last |> String50.create "Last"
        let! birthdate = dto.Birthdate |> Birthdate.create

        // combine the components to create the domain object
        return {
            First = first
            Last = last
            Birthdate = birthdate
        }
    }
```
这里使用了一个 result computation expression 来处理有可能会出现的 error ，因为 String50 和  Birthdate 从它们各自的 create function 中返回的是 Result 。

例如，String50.create 的实现可能会使用到第6章 Integrity of Simple Values 这一节所讨论的方式。代码如下。注意，这里把 field name 作为输入参数，因此我们会获得更有用的 error 信息：
```
let create fieldName str : Result<String50,string> =
    if String.IsNullOrEmpty(str) then
        Error (fieldName + " must be non-empty")
    elif str.Length > 50 then
        Error (fieldName + " must be less that 50 chars")
    else
        Ok (String50 str)
```

### Wrapping the JSON Serializer

类似序列化 JSON 或 XML 这样的功能，我们将使用第三方的 library 。然而，library 里的 API 可能不是 functional 友好的。因此需要将这些 API 包装一下，以使其匹配 pipeline 中的其它 function 。这里是一个包装 Newtonsoft.json library 的例子：
```
module Json =
    open Newtonsoft.Json

    let serialize obj =
        JsonConvert.SerializeObject obj

    let deserialize<'a> str =
        try
            JsonConvert.DeserializeObject<'a> str
            |> Result.Ok
        with
        // catch all exceptions and convert to Result
        | ex -> Result.Error ex
```
以上创建了一个 Json module ，以便将调整后的版本放入其中，这样我们就可以以 Json.serialize 和 Json.deserialize 的形式调用这些 function 。

### A Complete Serialization Pipeline

有了这些 DTO-to-domain 的转换 function ，就可以做如下的实现 ：
```
/// Serialize a Person into a JSON string
let jsonFromDomain (person:Domain.Person) =
    person
    |> Dto.Person.fromDomain
    |> Json.serialize
```
测试一下，结果是我们期望的 JSON string ：
```
// input to test with
let person : Domain.Person = {
    First = String50 "Alex"
    Last = String50 "Adams"
    Birthdate = Birthdate (DateTime(1980,1,1))
}

// use the serialization pipeline
jsonFromDomain person

// The output is
// "{"First":"Alex","Last":"Adams","Birthdate":"1980-01-01T00:00:00"}"
```

组合 serialization function 很简单，因为它不带有任何 Result ，但是组合 deserialization function 有点麻烦，因为 Json.deserialize 和 PersonDto.fromDomain 都有可能返回 Result 。解决方案是使用 Result.mapError 将潜在的错误转换成一个公共的 choice type ，然后使用 result expression 隐藏 error ：
```
type DtoError =
| ValidationError of string
| DeserializationException of exn

/// Deserialize a JSON string into a Person
let jsonToDomain jsonString :Result<Domain.Person,DtoError> =
    result {
        let! deserializedValue =
            jsonString
            |> Json.deserialize
            |> Result.mapError DeserializationException

        let! domainValue =
            deserializedValue
            |> Dto.Person.toDomain
            |> Result.mapError ValidationError

        return domainValue
    }
```
对其进行测试，没有 error 的情况：
```
// JSON string to test with
let jsonPerson = """{
    "First": "Alex",
    "Last": "Adams",
    "Birthdate": "1980-01-01T00:00:00"
}"""

// call the deserialization pipeline
jsonToDomain jsonPerson |> printfn "%A"

// The output is:
//  Ok {First = String50 "Alex";
//      Last = String50 "Adams";
//      Birthdate = Birthdate 01/01/1980 00:00:00;}
```
可以看到结果是 Ok ，Person 这个 domain object 也创建成功了。

现在调整一下 JSON string ，使其具有 error ——  name 为空 和 错误的 date ：
```
let jsonPersonWithErrors = """{
    "First": "",
    "Last": "Adams",
    "Birthdate": "1776-01-01T00:00:00"
}"""

// call the deserialization pipeline
jsonToDomain jsonPersonWithErrors |> printfn "%A"

// The output is:
//  Error (ValidationError [
//      "First must be non-empty"
//      ])
```
可以看到结果是一个 ValidationError 。在真实的应用程序中，你可以将其记录到日志，并将 error 返回给调用者。为了返回所有的 error ，可以参考 第10章 Composing in Parallel With Applicatives 这一节的内容。

反序列化过程中处理 exception 的另一种方式是 什么也不做，就只是让 反序列化 的代码抛出 exception 。选择哪种方式取决于你是 希望将反序列化错误作为预期情况处理，还是作为 “panic” ( 崩溃整个 pipeline ) 来处理 。反过来，这又取决于你的 API 的公开程度，你对调用者的信任度，以及你希望向调用者提供多少关于这类错误的信息。

### Working with Other Serializers

上面的代码使用 Newtonsoft.Json 作为序列化器。你也可以使用其它的，但是你也许想要给 PersonDto type 添加 attribute 。比如，要使用 DataContractSerializer ( 用于 XML ) 或 DataContractJsonSerializer ( 用于JSON ) 序列化一个 type 时，必须用 DataContractAttribute 和 DataMemberAttribute 装饰你的 DTO type ：
```
module Dto =
    [<DataContract>]
    type Person = {
        [<field: DataMember>]
        First: string

        [<field: DataMember>]
        Last: string

        [<field: DataMember>]
        Birthdate : DateTime
    }
```
这显示了让 serialization type 与 domain type 分离的另一个好处—— domain type 不会被这样的复杂的 attribute 所污染。一如既往，最好将 domain 的 与基础设施分离开来。

关于序列化器的另一个很有用的 attribute 是 CLIMutableAttribute ，它会产生一个 ( 隐藏的 ) 无参数构造函数，通常使用反射的序列化器需要它。

如果，你确定只会和其它的 F# 组件工作，那么你可以使用针对 F# 专门的序列化器 ( FsPickler 或 Chiron ) 。如果你使用这些专门的序列化器，所有的 bounded context 都必须使用相同的编程语言，而这就导致了 bounded context 之间的耦合。