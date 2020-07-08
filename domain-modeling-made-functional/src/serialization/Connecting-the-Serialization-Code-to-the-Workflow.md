## Connecting the Serialization Code to the Workflow

serialization 的过程仅仅只是作为组件被添加到 workflow pipeline 中：deserialization 步骤在 workflow 的开头，serialization 在 workflow 的最后。

例如，假设我们有一个 workflow 像下面这样 ( 这里忽略 error handling 和 Result ) ：
```rust
type MyInputType = ...
type MyOutputType = ...

type Workflow = MyInputType -> MyOutputType
```

deserialization 的 function signature 如下：
```rust
type JsonString = string
type MyInputDto = ...

type DeserialiseInputDto = JsonString -> MyInputDto
type InputDtoToDomain = MyInputDto -> MyInputType
```

serialization 步骤，如下：
```rust
type MyOutputDto = ...

type OutputDtoFromDomain = MyOutputType -> MyOutputDto
type SerialiseOutputDto = MyOutputDto -> JsonString
```

很明显，所有这些 function 都可以链接在一个 pipeline 中，就像这样：
```rust
let workflowWithSerialization jsonString =
    jsonString
    |> deserialiseInputDto      // JSON to DTO
    |> inputDtoToDomain         // DTO to domain object
    |> workflow                 // the core workflow in the domain
    |> outputDtoFromDomain      // Domain object to DTO
    |> serialiseOutputDto       // DTO to JSON
    // final output is another JsonString
```

然后这个 workflowWithSerialization function 将会公开给基础设施。输入和输出都是 jsonString 或其它类似的，因此，基础设施和 domain 是彼此独立的。

当然，实践时并没有那么简单!我们需要处理 error ， async ，等等。但这演示了基本的概念。

### DTOs as a Contract Between Bounded Contexts

我们会处理由别的 bounded contexts 所触发的 command ，我门的 workflow 发出的 event 会成为 别的 bounded context 的输入。这些 event 和 command 便成了 bounded context 必须实现的契约。当然这是一个宽松的契约，因为我们要避免 bounded context 之间的紧耦合。尽管如此，对于这些 event 和 command 的序列化格式 (dto) ，也应该小心的更改。这意味着你应该始终完全控制序列化格式，而不应该把这交由某个第三方的 library 自动完成！

