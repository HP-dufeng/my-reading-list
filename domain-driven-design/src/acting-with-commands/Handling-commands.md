## Handling commands

接下来，我们将学习更多关于命令处理和持久化的知识。另外，我们将讨论如何处理异常和校验传入的请求是否有效。

## The command handler pattern

在 CQRS 中有几种处理命令的方式。一种既定的模式是使用命令处理程序（command handler）。它不是 CQRS 特有的，但它被广泛使用，因为它非常适合。命令处理程序是一个 *class*，它有一个方法来处理单个命令类型。例如，我们可能有这样一个命令处理程序：
```csharp
public class CreateClassifiedAdHandler : 
    IHandleCommand<Contracts.ClassifiedAds.V1.Create>
{
    private readonly IEntityStore _store;

    public CreateClassifiedAdHandler(IEntityStore store) => _store = store;

    public Task Handle(Contracts.ClassifiedAds.V1.Create command)
    {
        var classifiedAd = new ClassifiedAd(
            new ClassifiedAdId(command.Id),
            new UserId(command.OwnerId));

        return _store.Save(classifiedAd);
    }
}
```

前面的命令处理程序使用了两个 interface ：
```csharp
public interface IHandleCommand<in T>
{
    Task Handle(T command);
}

public interface IEntityStore
{
    Task<T> Load<T>(string id);
    Task Save<T>(T entity);
}
```

请记住，IEntityStore 是经过简化的，并不是所有持久化方法都可以用这样的接口表示。

<blockquote><p>
<font size=5>👉</font>&nbsp;&nbsp;&nbsp;毫无疑问，我并没有试图在您的头脑中植入通用 repository 的想法。实际上，实体存储不是 repository pattern 的精确数学运算。如果 repository 的目的是模拟对象集合并隐藏持久化时，那么实体存储则完全相反。它不代表一个集合。它所做的正是它所告诉您的——持久化一个对象并将其检索回来。通用 repository 通常被认为是一种反模式，我不会对实体存储接口应用同样的反模式。
</p></blockquote>

然后在 API 中调用这个 命令处理程序 ：
```csharp

using System.Threading.Tasks;
using Marketplace.Contracts;
using Microsoft.AspNetCore.Mvc;
using static Marketplace.Contracts.ClassifiedAds;

namespace Marketplace.Api
{
    [Route("/ad")]
    public class ClassifiedAdsCommandsApi : Controller
    {
        private readonly IHandleCommand<V1.Create>_createAdCommandHandler;

        public ClassifiedAdsCommandsApi(
            IHandleCommand<V1.Create>
            createAdCommandHandler 
        ) =>
            _createAdCommandHandler = createAdCommandHandler;

        [HttpPost]
        public Task Post(V1.Create request) =>
            _createAdCommandHandler.Handle(request);
    }
}
```

然后注册这个 IHandleCommand ：

```csharp
services.AddSingleton<IEntityStore, RavenDbEntityStore>();

services.AddScoped<IHandleCommand<Contracts.ClassifiedAds.V1.Create>, CreateClassifiedAdHandler>();
```

<blockquote><p>
<font size=5>👉</font>&nbsp;&nbsp;&nbsp;在这里，我们注册了 RavenDbEntityStore class，它实现了 IEntityStore。我们不打算在这里展示实现的代码，但是由于 RavenDb 是文档数据库，这样的 class 实现起来可能很简单。
</p></blockquote>

到目前为止，所做的都非常简单，由于在 API 中使用了 IHandleCommand<T> ，所以我们可以做一些更有趣的事情。例如，我们可以创建一个用来重试的泛型的命令处理程序：
```csharp
public class RetryingCommandHandler<T> : IHandleCommand<T>
{
    static RetryPolicy _policy = 
        Policy
            .Handle<InvalidOperationException>()
            .Retry();

    private IHandleCommand<T> _next;

    public RetryingCommandHandler(IHandleCommand<T> next) => _next = next;

    public Task Handle(T command) =>
        _policy.ExecuteAsync(() => _next.Handle(command));
}
```

只需要将注册 service 的地方 更改为如下所示：
```csharp
services.AddScoped<IHandleCommand<V1.Create>>(c =>
    new RetryingCommandHandler<V1.Create>(
        new CreateClassifiedAdHandler(c.GetService<RavenDbEntityStore>())));
```

在这里，我们将实际的命令处理程序包装在泛型的重试处理程序中。因为它们都实现了相同的 interface ，所以我们可以组合这些 class 来构建管道。我们可以继续向管道中添加更多的元素，比如使用 circuit breaker 或 logger。

我们可以向命令 *class* 添加更多的属性（谨记 weak schema），但是我们可能希望更改的惟一处理程序是实际的命令处理程序。所有瞬态处理程序将保持不变，因为我们使用命令类型（复合类型）作为参数，因此接口定义本身不会改变。

命令处理程序模式非常好，并且它遵循 **单一职责原则(SRP)**。同时，API 中的每个 HTTP 方法都需要一个单独的命令处理程序作为依赖项。如果我们有2个或3个方法，这不是什么大问题，但我们可能有更多的方法。通过查看 EventStorming 会议的结果，可以预测 web API 中将有超过10个方法，以及足够数量的命令处理程序。命令处理程序需要实体存储作为依赖项，因为所有 API controller 都是按作用域实例化的，所以所有的命令处理程序以及它们的依赖项也将被实例化和注入。通过使用工厂委托而不是每个请求的依赖项，可以减轻大量依赖树的实例化，因此每个方法都能够实例化其处理程序：
```csharp
using System;
using System.Threading.Tasks;
using Marketplace.Contracts;
using Microsoft.AspNetCore.Mvc;
using static Marketplace.Contracts.ClassifiedAds;
namespace Marketplace.Api
{
    [Route("/ad")]
    public class ClassifiedAdsCommandsApi : Controller
    {
        private readonly Func<IHandleCommand<V1.Create>> _createAdCommandHandlerFactory;

        public ClassifiedAdsCommandsApi(Func<IHandleCommand<V1.Create>> createAdCommandHandlerFactory) => 
            _createAdCommandHandlerFactory = createAdCommandHandlerFactory;

        [HttpPost]
        public Task Post(V1.Create request) =>
            _createAdCommandHandlerFactory().Handle(request);
    }
}
```

这种方法需要更高级的注册，因为我们使用的不是实际类型，而是委托。另一个解决方案可能是使用 Lazy<IHandleCommand<T>> 作为依赖项。同样，这将需要更复杂的注册。注册问题可以通过使用另一个依赖注入容器来解决，比如 Autofac，它支持自动工厂委托和 Lazy<T> 的开箱即用。

在本书中，我们将不使用*命令处理程序模式（command handler pattern）*，而是使用 application service 来实现命令处理。这里已经实现了一个简单的 service，将在下一节中继续。命令处理程序的存在是为了更好地概述有用的模式，因为没有一种模式能够适合所有用例。

