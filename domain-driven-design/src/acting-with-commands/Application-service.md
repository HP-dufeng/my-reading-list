## Application service

实际上，我们的 application service 与 command handler 非常相似。经典的 application service 公开一些具有多个参数的方法，如下面所示：
```csharp
public interface IPaymentApplicationService
{
    Guid Authorize(
        string creditCardNumber,
        int expiryYear,
        int expiryMonth,
        int cvcCode,
        int camount);

    void Capture(Guid authorizationId);
}
```

使用这种 interface 很好，但是它不能很好地组合（composition） 。将这样的 application service 添加到有 日志记录、重试策略等的 pipeline 中并不容易。要创建一个 pipeline ，我们需要所有的处理程序都有兼容的参数，但是 IPaymentApplicationService 的这些方法不允许我们这样做。pipeline 中的其他调用必须具有相同的一组参数，一旦我们想要向任何方法添加更多的参数，我们注定要在组成 pipeline 的多个 *class* 中进行大量更改。这不是我们想做的事情。

或者，我们可以有一个实现了多个 IHandle<T> 的 application service *class*。这是可行的，但是每个命令都需要一个单独的引导代码，尽管我们正在向管道中添加相同的元素：
```csharp
services.AddScoped<IHandleCommand<V1.Create>>(c =>
    new RetryingCommandHandler<V1.Create>(
        new CreateClassifiedAdHandler(c.GetService<RavenDbEntityStore>())));

services.AddScoped<IHandleCommand<V1.Create>>(c =>
    new RetryingCommandHandler<V1.Rename>(
        new RenameClassifiedAdHandler(c.GetService<RavenDbEntityStore>())));

// more handlers need to be added with the same composition
```

或者，我们可以泛化我们的 application service 来处理任何类型的命令，并再次使用 c# 7 的高级模式匹配特性（就像我们在事件处理中所做的那样）。在本例中，application service 看起来像这样：
```csharp
public interface IApplicationService
{
    Task Handle(object command);
}
```

我们以前用于 pipeline 的所有的 filter，如 retry filter 或 logging filter，都可以实现这个简单的 interface 。因为这些 class 不需要知道命令的内容，所以一切都会正常工作。我们的 service 将会是这样的：
```csharp
using System;
using System.Threading.Tasks;
using Marketplace.Framework;

using static Marketplace.Contracts.ClassifiedAds;

namespace Marketplace.Api
{
    public class ClassifiedAdsApplicationService : IApplicationService
    {
        public async Task Handle(object command)
        {
            switch (command)
            {
                case V1.Create cmd:
                    // we need to create a new Classified Ad here
                    break;
                default:
                    throw new InvalidOperationException(
                        $"Command type {command.GetType().FullName} is unknown");
            }
        }
    }
}

```

通过像这样实现的 application service，我们将有一个单独的依赖来处理所有的 API 调用，并且我们保持大门敞开以便将来可以组合一个更复杂的命令处理 pipeline ，就像我们能够处理单个命令一样。

当然，这里的折衷是一个 *class* 处理了多个命令，有些人会认为它违反了 SRP。与此同时，这个 *class* 是高内聚的，我们将在本章后面看到更多，那时我们将充分处理几个命令并从边缘层（web controller）调用我们的 application service。

现在让我们添加更多命令并相应地扩展 application service  和 web controller 。

首先，回顾一下，我们可以在实体上执行那些操作：
1. Set the title
2. Update the text
3. Update the price
4. Request to publish

可以添加四个命令来执行这些操作，根据 EventStorming 会议上讨论的，可以预期这是我们的用户想要做的。

命令的实现，如下所示：
```csharp
using System;

namespace Marketplace.Contracts
{
    public static class ClassifiedAds
    {
        public static class V1
        {
            public class Create
            {
                public Guid Id { get; set; }

                public Guid OwnerId { get; set; }
            }

            public class SetTitle
            {
                public Guid Id { get; set; }

                public string Title { get; set; }
            }

            public class UpdateText
            {
                public Guid Id { get; set; }

                public string Text { get; set; }
            }

            public class UpdatePrice
            {
                public Guid Id { get; set; }

                public decimal Price { get; set; }

                public string Currency { get; set; }
            }

            public class RequestToPublish
            {
                public Guid Id { get; set; }
            }
        }
    }
}

```

每个命令都需要有它将要操作的实体的ID。其他属性是特定于命令的。

其次，我们可以扩展边缘层，将这些命令作为 HTTP 请求的参数。对 API 的修改如下：
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
        private readonly ClassifiedAdsApplicationService _applicationService;

        public ClassifiedAdsCommandsApi(
            ClassifiedAdsApplicationService applicationService) => 
                _applicationService = applicationService;

        [HttpPost]
        public async Task<IActionResult> Post(V1.Create request)
        {
            await _applicationService.Handle(request);

            return Ok();
        }

        [Route("name")]
        [HttpPut]
        public async Task<IActionResult> Put(V1.SetTitle request)
        {
            await _applicationService.Handle(request);

            return Ok();
        }

        [Route("text")]
        [HttpPut]
        public async Task<IActionResult> Put(V1.UpdateText request)
        {
            await _applicationService.Handle(request);

            return Ok();
        }

        [Route("price")]
        [HttpPut]
        public async Task<IActionResult> Put(V1.UpdatePrice request)
        {
            await _applicationService.Handle(request);

            return Ok();
        }

        [Route("publish")]
        [HttpPut]
        public async Task<IActionResult> Put(V1.RequestToPublish request)
        {
            await _applicationService.Handle(request);
            
            return Ok();
        }
    }
}
```

您可能预测到此代码在生产环境中运行时会出现一些问题。前面的边缘层代码还违反了一个重要原则，即 API controller 只能向命令处理程序发送有效的命令。在我们的代码中，没有执行这样的校验。别担心；我们将回到 API 代码并解决其中一些问题。现在，让我们集中讨论最基本的部分。

如您所见，我们的 application service 需要处理5个命令。新的代码如下：
```csharp
using System;
using System.Threading.Tasks;
using Marketplace.Contracts;
using Marketplace.Domain;
using Marketplace.Framework;

using static Marketplace.Contracts.ClassifiedAds;

namespace Marketplace.Api
{
    public class ClassifiedAdsApplicationService : IApplicationService
    {
        private readonly IEntityStore _store;

        private ICurrencyLookup _currencyLookup;

        public ClassifiedAdsApplicationService(
            IEntityStore store,
            ICurrencyLookup currencyLookup
        )
        {
            _store = store;
            _currencyLookup = currencyLookup;
        }

        public async Task Handle(object command)
        {
            ClassifiedAd classifiedAd;
            switch (command)
            {
                case V1.Create cmd:
                    if (await _store.Exists<ClassifiedAd>(cmd.Id.ToString()))
                        throw new InvalidOperationException(
                            $"Entity with id {cmd.Id} already exists");
                    classifiedAd = 
                        new ClassifiedAd(
                            new ClassifiedAdId(cmd.Id), 
                            new UserId(cmd.OwnerId)
                        );
                    await _store.Save(classifiedAd);
                    break;
                case V1.SetTitle cmd:
                    classifiedAd = await _store.Load<ClassifiedAd>(cmd.Id.ToString());
                    if (classifiedAd == null)
                        throw new InvalidOperationException(
                            $"Entity with id {cmd.Id} cannot be found");
                    classifiedAd.SetTitle(ClassifiedAdTitle.FromString(cmd.Title));
                    await _store.Save(classifiedAd);
                    break;
                case V1.UpdateText cmd:
                    classifiedAd = await _store.Load<ClassifiedAd>(cmd.Id.ToString());
                    if (classifiedAd == null)
                        throw new InvalidOperationException(
                            $"Entity with id {cmd.Id} cannot be found");
                    classifiedAd.UpdateText(ClassifiedAdText.FromString(cmd.Text));
                    await _store.Save(classifiedAd);
                    break;
                case V1.UpdatePrice cmd:
                    classifiedAd = await _store.Load<ClassifiedAd>(cmd.Id.ToString());
                    if (classifiedAd == null)
                        throw new InvalidOperationException(
                            $"Entity with id {cmd.Id} cannot be found");
                    classifiedAd.UpdatePrice(Price
                            .FromDecimal(cmd.Price, cmd.Currency, _currencyLookup));
                    await _store.Save(classifiedAd);
                    break;
                case V1.RequestToPublish cmd:
                    classifiedAd = await _store.Load<ClassifiedAd>(cmd.Id.ToString());
                    if (classifiedAd == null)
                        throw new InvalidOperationException(
                            $"Entity with id {cmd.Id} cannot be found");
                    classifiedAd.RequestToPublish();
                    await _store.Save(classifiedAd);
                    break;
                default:
                    throw new InvalidOperationException(
                        $"Command type {command.GetType().FullName} is unknown");
            }
        }
    }
}

```

这里，我们再次使用 IEntityStore 。这个 interface 非常简单：
```csharp
using System.Threading.Tasks;

namespace Marketplace.Framework
{
    public interface IEntityStore
    {
        /// <summary>
        /// Loads an entity by id
        /// </summary>
        Task<T> Load<T>(string entityId) where T : Entity;

        /// <summary>
        /// Persists an entity
        /// </summary>
        Task Save<T>(T entity) where T : Entity;

        /// <summary>
        /// Check if entity with a given id already exists
        /// <typeparam name="T">Entity type</typeparam>
        Task<bool> Exists<T>(string entityId);
    }
}

```

我们将为不同的持久化类型实现此 interface 。

如您所见，处理 Create 命令看起来与处理所有其他命令不同。这是很自然的，因为当我们创建一个新实体时，我们需要确保它还不存在。当我们处理现有实体上的操作时，它的工作方式是相反的。在这种情况下，我们需要确保实体存在，否则，我们就不能执行操作，并且必须抛出异常。

另一件值得一提的事情是，application service 负责将基本类型（如 string 或 decimal） 转换为值对象。边缘层始终使用不依赖于领域模型的可序列化类型。然而，application service 与领域有关；它需要告诉我们的领域模型要做什么，并且由于领域模型更喜欢将数据作为值对象接收，因此 application service 要负责转换。

用于处理现有实体的命令的代码看起来与处理现有实体的更新非常相似。事实上，只有我们调用实体方法的那一行是不同的。因此，我们可以通过使用简单的泛型方法并使用 switch 表达式替换 switch 模式匹配操作符来简化 Handle 方法：
```csharp

using System;
using System.Threading.Tasks;
using Marketplace.Domain;
using Marketplace.Framework;
using static Marketplace.Contracts.ClassifiedAds;
namespace Marketplace.Api
{
    public class ClassifiedAdsApplicationService : IApplicationService
    {
        private readonly IClassifiedAdRepository _repository;
        private readonly ICurrencyLookup _currencyLookup;

        public ClassifiedAdsApplicationService(
            IClassifiedAdRepository repository,
            ICurrencyLookup currencyLookup)
        {
            _repository = repository;
            _currencyLookup = currencyLookup;
        }

        public Task Handle(object command) => 
        command switch
        {
            V1.Create cmd => 
                HandleCreate(cmd),
            V1.SetTitle cmd => 
                HandleUpdate(
                    cmd.Id,
                    c => c.SetTitle(ClassifiedAdTitle.FromString(cmd.Title))
                ),
            V1.UpdateText cmd => 
                HandleUpdate(
                    cmd.Id,
                    c => c.UpdateText(ClassifiedAdText.FromString(cmd.Text))
                ),
            V1.UpdatePrice cmd => 
                HandleUpdate(
                    cmd.Id,
                    c => c.UpdatePrice(Price.FromDecimal(
                        cmd.Price,
                        cmd.Currency,
                        _currencyLookup
                    ))
                ),
            V1.RequestToPublish cmd => 
                HandleUpdate(
                    cmd.Id,
                    c => c.RequestToPublish()
                ),
            _ => Task.CompletedTask
        };

        private async Task HandleCreate(V1.Create cmd)
        {
            if (await _repository.Exists(cmd.Id.ToString()))
                throw new InvalidOperationException(
                    $"Entity with id {cmd.Id} already exists");

            var classifiedAd = 
                new ClassifiedAd(
                    new ClassifiedAdId(cmd.Id),
                    new UserId(cmd.OwnerId)
                );

            await _repository.Save(classifiedAd);
        }

        private async Task HandleUpdate(
            Guid classifiedAdId,
            Action<ClassifiedAd> operation)
        {
            var classifiedAd = 
                await _repository.Load(classifiedAdId.ToString());

            if (classifiedAd == null)
                throw new InvalidOperationException(
                    $"Entity with id {classifiedAdId} cannot be found");

            operation(classifiedAd);

            await _repository.Save(classifiedAd);
        }
    }
}
```

从 application service 的代码可以清楚地看出，application service 本身在应用程序*边缘层*和领域模型之间扮演着重要的中介角色。*边缘层*可以是任何与外部世界沟通的东西。我们的 *边缘层* 使用了 HTTP API 作为示例，但它也可以是消息传递接口或完全不同的接口。*边缘层*的重要需求是能够接受命令、校验它们，并使用 application service 来处理这些命令。

当我们处理一个命令时，无论我们使用多个命令处理程序还是单个 application service，操作的顺序通常非常相似。命令处理程序需要从实体存储中获取一个持久的实体，调用领域模型来完成这项工作，然后持久化更改。在我们的示例中，我们只调用了实体的一个方法，但情况并非总是如此。当我们在下一章讨论一致性和事务边界时，我们将对此进行更深入的讨论。