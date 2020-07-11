## Working with Relational Databases

relational database 的模型完全不同于大多数代码，传统上这是造成很多麻烦的原因——所谓的 object 和 database 之间的 “impedance mismatch” ( 阻抗失调 )。

以 functional programming 的原则开发的数据模型往往与 relational database 更兼容，这主要是因为 functional model 不会混合数据和行为，因此记录的保存和检索更加简单。

但是，仍然有一些问题需要解决。让我们比较一下 relational database model 和 functional model 。

首先，好消息是 relational database 中的 table 与 functional model 中的 record list 能够很好地对应。database 中面向集合的操作 ( SELECT, WHERE ) 类似于 functional programming 中面向列表的操作 ( map, filter )。

因此，我们的策略是使用前一章中的序列化技术 来设计 可以直接映射到 table 的 record type 。比如，假设有一个 domain type ：
```rust
type CustomerId = CustomerId of int
type String50 = String50 of string
type Birthdate = Birthdate of DateTime

type Customer = {
    CustomerId : CustomerId
    Name : String50
    Birthdate : Birthdate option
}
```
然后设计与之对应的 table 也很简单：
```sql
CREATE TABLE Customer (
    CustomerId int NOT NULL,
    Name NVARCHAR(50) NOT NULL,
    Birthdate DATETIME NULL,
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
)
```

坏消息是 relational database 只存储 primitive type ，如 string 或 int ，这意味着不得不将设计的很好的 domain type 做一下 unwrap ，如 ProductCode 或 OrderId 。

更糟糕的是，relational table 不能很好地映射到 choice type 。这也是我们将要深入的研究的问题。

### Mapping Choice Types to Tables

在 relational database 中如何建模 choice type ？ 如果将 choice type 想象成一种单级继承的层次结构，然后就可以借用一些用于将对象层次结构映射到关系模型的方法。<sup>5</sup>

两种常用的映射 choice type 的方式：
* 所有的 case 保存在相同的 table 中。
* 每个 case 都有自己的 table 。

例如，假设有一个 Contact type ，它包含一个 choice type ，然后需要将它保存到 database 中：
```rust
type Contact = {
    ContactId : ContactId
    Info : ContactInfo
}

and ContactInfo =
| Email of EmailAddress
| Phone of PhoneNumber

and EmailAddress = EmailAddress of string
and PhoneNumber = PhoneNumber of string
and ContactId = ContactId of int
```

第一种方式 (“所有的 case 保存在相同的 table 中”) 和之前讨论过的类似 (参见第11章 Choice types 这一节)。使用一个 table 来存储所有的 case ，这又意味着， (a) 需要一个或多个的 flag 表示那个 case 被使用了，(b) 使用 NULLable column 保存 值是 Some 的那些 case 。
```sql
CREATE TABLE ContactInfo (
    -- shared data
    ContactId int NOT NULL,
    -- case flags
    IsEmail bit NOT NULL,
    IsPhone bit NOT NULL,
    -- data for the "Email" case
    EmailAddress NVARCHAR(100), -- Nullable
    -- data for the "Phone" case
    PhoneNumber NVARCHAR(25), -- Nullable
    -- primary key constraint
    CONSTRAINT PK_ContactInfo PRIMARY KEY (ContactId)
)
```
我们为每个 case 使用了一个 bit 字段，而不是单个 Tag VARCHAR 字段 ，因为这样更紧凑，更容易索引。

第二种方式 (“每个 case 都有自己的 table”) 意思是 除了 main table 之外，还需要两个 child table ，每个 case 有一个自己的 child table 。所有的 table 共享 primary key 。main table 保存 id 和 一些 flag ，这些 flag 表示那些 case 被使用了。child table 保存和 case 关联的数据。为了增加复杂性，我们在 database 中使用了更好的约束 (例如 child table 中的 NOT NULL column )。
``` sql
-- Main table
CREATE TABLE ContactInfo (
    -- shared data
    ContactId int NOT NULL,
    -- case flags
    IsEmail bit NOT NULL,
    IsPhone bit NOT NULL,
    CONSTRAINT PK_ContactInfo PRIMARY KEY (ContactId)
)

-- Child table for "Email" case
CREATE TABLE ContactEmail (
    ContactId int NOT NULL,
    -- case specific data
    EmailAddress NVARCHAR(100) NOT NULL,
    CONSTRAINT PK_ContactEmail PRIMARY KEY (ContactId)
)

-- Child table for "Phone" case
CREATE TABLE ContactPhone (
    ContactId int NOT NULL,
    -- case specific data
    PhoneNumber NVARCHAR(25) NOT NULL,
    CONSTRAINT PK_ContactPhone PRIMARY KEY (ContactId)
)
```

这种 “multi-table” 的方式更适合用于： 和 case 关联的数据非常大且几乎没有共同点，对于其它的情况，则默认使用 “one-table” 的方式。

### Reading from a Relational Database

在 F# 中，我们倾向于不使用 object-relational mapper ( ORM )，而是直接使用原始的 SQL 命令。最方便的方式是使用 F# SQL type provider 。有很多这样的 provider ——对于本例，我们将使用 FSharp.Data.SqlClient type provider 。<sup>6</sup> 

使用 type provider 而不是典型的 运行时库 的好处在于，SQL type provider 将在编译时创建与 SQL 查询或 SQL 命令相匹配的类型。如果 SQL 不正确，将会得到一个编译时错误，而不是一个运行时错误。如果 SQL 是正确的，那么 provider 会生成于 SQL 代码的输出完全匹配的 F# record type 。

假如要根据 CustomerId 查询 Customer 。下面是使用 type provider 的实现方式。

首先，定义一个在编译时需要用到的 connection string 。
```rust
open FSharp.Data

[< Literal>]
let CompileTimeConnectionString =
    @"Data Source=(localdb)\MsSqlLocalDb; Initial Catalog=DomainModelingExample;"
```

然后将查询定义成 type ，命名为 ReadOneCustomer ，如下：
```rust
type ReadOneCustomer = SqlCommandProvider<"""
    SELECT CustomerId, Name, Birthdate
    FROM Customer
    WHERE CustomerId = @customerId
    """, CompileTimeConnectionString>
```
在编译时，type provider 将在 database 上运行该查询，并生成表示该查询的 type 。这和 SqlMetal<sup>7</sup> 或 EdmGenerator<sup>8</sup> 相似，除了，不生成单独的文件—— type 在原处被创建。稍后，在生产环境，使用这种 type 时，我们提供一个与编译时不同的 connection string ( “production” connection string )。

接下来，和前一章中的序列化示例一样，创建一个 toDomain function 。这个 function 校验 database 中的字段，然后使用 result expression 进行封装。

也就是说，我们将 database 视为不可信的数据源，需要像校验任何其他数据源一样进行验证，因此toDomain function 需要返回 Result<Customer,_> ，而不是普通的 Customer 。代码如下：
```rust
let toDomain (dbRecord:ReadOneCustomer.Record) : Result<Customer,_> =
    result {
        let! customerId =
            dbRecord.CustomerId
            |> CustomerId.create
        let! name =
            dbRecord.Name
            |> String50.create "Name"
        let! birthdate =
            dbRecord.Birthdate
            |> Result.bindOption Birthdate.create

        let customer = {
            CustomerId = customerId
            Name = name
            Birthdate = birthdate
        }

        return customer
    }
```

在前一章中见过类似的代码 (参见第11章 [Serialization]() ) 。不过，增加了一点功能。database 中的 Birthdate column 是 nullable 的，因此，type provider 生成与之对应的 dbRecord.Birthdate field 为 Option type 。但是 Birthdate.create function 并不接受 Option 。为了解决这个问题，创建一个名为 bindOption 的 helper function ，以使 “switch” function 可以处理 Option 。
```rust
let bindOption f xOpt =
    match xOpt with
    | Some x -> f x |> Result.map Some
    | None -> Ok None
```

编写自定义的 toDomain function ，使用 Result type ，类似这样的代码，增加了一点点复杂性，但是一旦完成这些代码，就可以确保永远不会出现未处理的错误。

换句话说，如果非常确信 database 永远不会包含错误的数据，或者即使存在错误的数据，如果我们乐意见到程序 panic ，那么就可以为无效的数据抛出 exception 。对于这种情况，可以创建一个 panicOnError helper function ( 它将 error Result 转换为 exception ) ，这反过来意味着toDomain function 的输出是一个普通的 Customer ，而不是一个包装了 Customer 的 Result 。代码看起来如下：
```rust
let toDomain (dbRecord:ReadOneCustomer.Record) : Customer =
    let customerId =
        dbRecord.CustomerId
        |> CustomerId.create
        |> panicOnError "CustomerId"

    let name =
        dbRecord.Name
        |> String50.create "Name"
        |> panicOnError "Name"

    let birthdate =
        dbRecord.Birthdate
        |> Result.bindOption Birthdate.create
        |> panicOnError "Birthdate"

    // return the customer
    {CustomerId = customerId; Name = name; Birthdate = birthdate}
```
panicOnError helper function 的定义如下：
```rust
exception DatabaseError of string

let panicOnError columnName result =
    match result with
    | Ok x -> x
    | Error err ->
        let msg = sprintf "%s: %A" columnName err
        raise (DatabaseError msg)
```

无论哪种方式，一旦有了 toDomain function ，就可以编写代码来读取 database ，并返回一个 domain type 。比如，下面的 readOneCustomer function 执行了一个 ReadOneCustomer query ，然后将查询的结果转会成 domain type 并返回：
```rust
type DbReadError =
    | InvalidRecord of string
    | MissingRecord of string

let readOneCustomer (productionConnection:SqlConnection) (CustomerId customerId) =
    // create the command by instantiating the type we defined earlier
    use cmd = new ReadOneCustomer(productionConnection)

    // execute the command
    let records = cmd.Execute(customerId = customerId) |> Seq.toList

    // handle the possible cases
    match records with
    // none found
    | [] ->
        let msg = sprintf "Not found. CustomerId=%A" customerId
        Error (MissingRecord msg) // return a Result
    // exactly one found
    | [dbCustomer] ->
        dbCustomer
        |> toDomain
        |> Result.mapError InvalidRecord
    // more than one found?
    | _ ->
        let msg = sprintf "Multiple records found for CustomerId=%A" customerId
        raise (DatabaseError msg)
```

首先，请注意，现在显式地传递一个 SqlConnection ，将它用作 “production” connection 。

接下来，这里有三种可能会出现的 case ——没有查询到数据，一条数据，多条数据。这时需要决定哪些 case 应该作为 domain 的一部分处理，哪些 case 是不应该发生的 ( 并且可以作为 panic 处理 )。在本例中，我们说可能会出现查询不到数据的情况 ，所以将这种 case 视为 Result.Error ，而查询到多个记录的情况将被视为 panic 。

处理这些不同的 case 似乎要做很多工作，但是这样做的好处是，可以明确地对可能出现的错误做出决策( 并且这也是代码即文档的体现 )，而不是假设一切正常，然后在将来的某个时刻出现 NullReferenceException 。

当然，也可以遵循 “参数化所有的东西” 的原则来创建一个通用的 function ，如下面的  convertSingleDbRecord ，其中的 table name ，id ，record 和 toDomain 都作为参数传入：
```rust
let convertSingleDbRecord tableName idValue records toDomain =
    match records with
    // none found
    | [] ->
        let msg = sprintf "Not found. Table=%s Id=%A" tableName idValue
        Error msg   // return a Result

    // exactly one found
    | [dbRecord] ->
        dbRecord
        |> toDomain
        |> Ok       // return a Result

    // more than one found?
    | _ ->
        let msg = sprintf "Multiple records found. Table=%s Id=%A" tableName idValue
        raise (DatabaseError msg)
```
然后，readOneCustomer 减少到只有几行了：
```rust
let readOneCustomer (productionConnection:SqlConnection) (CustomerId customerId) =
    use cmd = new ReadOneCustomer(productionConnection)
    let tableName = "Customer"

    let records = cmd.Execute(customerId = customerId) |> Seq.toList
    convertSingleDbRecord tableName customerId records toDomain
```

### Reading Choice Types from a Relational Database

可以使用同样的方式读取 choice type ，尽管稍微有点复杂。假设，是以 “one-table” 的方式存储的数据。想要使用 ContactId 读取单个 ContactInfo 。像之前一样将查询定义为 type ，如下所示：
```rust
type ReadOneContact = SqlCommandProvider<"""
    SELECT ContactId,IsEmail,IsPhone,EmailAddress,PhoneNumber
    FROM ContactInfo
    WHERE ContactId = @contactId
    """, CompileTimeConnectionString>
```
接下来，创建一个 toDomain function 。这个 function 检查 database 中的 flag ( isEmail ) 
，查看要创建 ContactInfo 的哪个 case ，然后使用子 result expressions 为每个 case 组装数据(好极了，可组合性!)
```rust
let toDomain (dbRecord:ReadOneContact.Record) : Result<Contact,_> =
    result {
        let! contactId =
            dbRecord.ContactId
            |> ContactId.create
        let! contactInfo =
            if dbRecord.IsEmail then
                result {
                    // get the primitive string which should not be NULL
                    let! emailAddressString =
                        dbRecord.EmailAddress
                        |> Result.ofOption "Email expected to be non null"
                    // create the EmailAddress simple type
                    let! emailAddress =
                        emailAddressString |> EmailAddress.create
                    // lift to the Email case of Contact Info
                    return (Email emailAddress)
                }
            else
                result {
                    // get the primitive string which should not be NULL
                    let! phoneNumberString =
                        dbRecord.PhoneNumber
                        |> Result.ofOption "PhoneNumber expected to be non null"
                    // create the PhoneNumber simple type
                    let! phoneNumber =
                        phoneNumberString |> PhoneNumber.create
                    // lift to the PhoneNumber case of Contact Info
                    return (Phone phoneNumber)
                }

        let contact = {
            ContactId = contactId
            Info = contactInfo
        }
        
        return contact
    }
```
例如，在 Email case 中，database 中的 EmailAddress column 是 nullable 的，所以 type provider 生成 dbRecord.EmailAddress 为 Option type ，因此，首先必须使用 Result.ofOption 将 Option 转换成 Result ( 以防丢失 ) ，然后创建 EmailAddress type ，然后将其提升为 ContactInfo 的 Email case 。

这比之前的 Customer 的例子更复杂，但我们有信心，这样的代码不会出现不可预期的错误。

顺便提一下，如果你好奇 Result.ofOption function 到底长什么样， 如下所示：
```rust
module Result =
    /// Convert an Option into a Result
    let ofOption errorValue opt =
        match opt with
        | Some v -> Ok v
        | None -> Error errorValue
```

与前面一样，一旦有了 toDomain function ，就可以将其与前面创建的 convertSingleDbRecord helper function 结合起来。
```rust
let readOneContact (productionConnection:SqlConnection) (ContactId contactId) =
    use cmd = new ReadOneContact(productionConnection)
    let tableName = "ContactInfo"

    let records = cmd.Execute(contactId = contactId) |> Seq.toList
    convertSingleDbRecord tableName contactId records toDomain
```

可以看到，创建 toDomain function 是比较困难的部分。一旦完成，实际的操作 database 代码就相对简单了。

你可能会想：哇，这不是要做很多工作吗？ 难道就不能使用 Entity Framework 或 NHibernate 之类的框架来自动完成这些映射吗？ 答案是否定的，如果你想确保 domain 的完整性，那么就不能这样做。前面提到的 ORM 不能校验 email address 和 order quantity ，不能处理嵌套的 choice type ，等等。是的，编写这种处理 database 的代码是很乏味的，但是这个过程是机械而直接的，而且不是编写应用程序最困难的部分!


### Writing to a Relational Database

对于 relational database 来说，写入与读取遵循相同的模式：将 domain object 转换成 DTO ，然后执行 insert 或 update 命令 。

实现 insert 的最简单方法是，让 SQL type provider 生成一个 mutable type ，这个 type 可以表示 table 的结构的，然后我们只需设置这个 type 的 field 。

这是一个示例。首先，使用 type provider 为所有 table 生成对应的 type ：
```rust
type Db = SqlProgrammabilityProvider<CompileTimeConnectionString>
```

然后，定义一个 writeContact function ，它接受一个 Contact ，并设置 database 中对应的 Contact 的所有 field ：
```rust
let writeContact (productionConnection:SqlConnection) (contact:Contact) =
    // extract the primitive data from the domain object
    let contactId = contact.ContactId |> ContactId.value

    let isEmail,isPhone,emailAddressOpt,phoneNumberOpt =
        match contact.Info with
        | Email emailAddress->
            let emailAddressString = emailAddress |> EmailAddress.value
            true,false,Some emailAddressString,None
        | Phone phoneNumber ->
            let phoneNumberString = phoneNumber |> PhoneNumber.value
            false,true,None,Some phoneNumberString

    // create a new row
    let contactInfoTable = new Db.dbo.Tables.ContactInfo()
    let newRow = contactInfoTable.NewRow()
    newRow.ContactId <- contactId
    newRow.IsEmail <- isEmail
    newRow.IsPhone <- isPhone
    // use optional types to map to NULL in the database
    newRow.EmailAddress <- emailAddressOpt
    newRow.PhoneNumber <- phoneNumberOpt

    // add to table
    contactInfoTable.Rows.Add newRow

    // push changes to the database
    let recordsAffected = contactInfoTable.Update(productionConnection)
    recordsAffected
```

另一种方式是使用 SQL 语句，使用这种方式你可以做更多的控制。比如，添加一条新的 Contact 数据，首先定义一个 type ，用它描述一条 SQL INSERT 语句：
```rust
type InsertContact = SqlCommandProvider<"""
    INSERT INTO ContactInfo
    VALUES (@ContactId,@IsEmail,@IsPhone,@EmailAddress,@PhoneNumber)
    """, CompileTimeConnectionString>
```

然后，定义一个 writeContact function，它接受一个 Contact ，从 choice type 中提取出 primitive 值，然后执行命令。
```rust
let writeContact (productionConnection:SqlConnection) (contact:Contact) =
    // extract the primitive data from the domain object
    let contactId = contact.ContactId |> ContactId.value
    let isEmail,isPhone,emailAddress,phoneNumber =
        match contact.Info with
        | Email emailAddress->
            let emailAddressString = emailAddress |> EmailAddress.value
            true,false,emailAddressString,null
        | Phone phoneNumber ->
            let phoneNumberString = phoneNumber |> PhoneNumber.value
            false,true,null,phoneNumberString

    // write to the DB
    use cmd = new InsertContact(productionConnection)
    cmd.Execute(contactId,isEmail,isPhone,emailAddress,phoneNumber)
```





---
5.  http://www.agiledata.org/essays/mappingObjects.html
6.  http://fsprojects.github.io/FSharp.Data.SqlClient/
7.  https://msdn.microsoft.com/en-us/library/bb386987.aspx
8.  https://msdn.microsoft.com/en-us/library/bb387165.aspx


