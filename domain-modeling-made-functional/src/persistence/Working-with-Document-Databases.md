## Working with Document Databases

前面我们讨论了 persistence 的一些一般原则。现在我们调转方向，深入到一些实现示例中，从所谓的 “ document database” 开始，该 database 用于存储 JSON 或 XML 格式的 semi-structured ( 半结构化 ) 数据。

持久化到 document database 非常简单。使用前面讨论过的技术 ( 参见第11章 [Serialization]() 这一节 ) 将 domain object 转换成 DTO ，再将 DTO 序列化成 JSON string ( 或 XML ) ，然后使用存储系统的 API 保存或加载这些数据。

例如，如果我们使用 Azure blob storage 来保存 PersonDto object ，我们可以这样设置：
```rust
open Microsoft.WindowsAzure
open Microsoft.WindowsAzure.Storage
open Microsoft.WindowsAzure.Storage.Blob

let connString = "... Azure connection string ..."
let storageAccount = CloudStorageAccount.Parse(connString)
let blobClient = storageAccount.CreateCloudBlobClient()

let container = blobClient.GetContainerReference("Person");
container.CreateIfNotExists()
```

然后用几行代码就可以将 DTO 保存为 blob：
```rust
type PersonDto = {
    PersonId : int
    ...
}

let savePersonDtoToBlob personDto =
    let blobId = sprintf "Person%i" personDto.PersonId
    let blob = container.GetBlockBlobReference(blobId)
    let json = Json.serialize personDto
    blob.UploadText(json)
```

好了，这就是全部。同样，可以使用前一章中的 反序列化技术 来加载数据 。