---
title: ASP.NET Core Blazor 依赖关系注入
author: guardrex
description: 了解 Blazor 应用如何将服务注入组件。
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 02/20/2020
no-loc:
- Blazor
- SignalR
uid: blazor/dependency-injection
ms.openlocfilehash: 4cdde9ee8c9fd9adf00894a067d32965b180e5ec
ms.sourcegitcommit: 9a129f5f3e31cc449742b164d5004894bfca90aa
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/06/2020
ms.locfileid: "78646692"
---
# <a name="aspnet-core-blazor-dependency-injection"></a>ASP.NET Core Blazor 依赖关系注入

作者：[Rainer Stropek](https://www.timecockpit.com) 和 [Mike Rousos](https://github.com/mjrousos)

[!INCLUDE[](~/includes/blazorwasm-preview-notice.md)]

Blazor 支持[依赖关系注入 (DI)](xref:fundamentals/dependency-injection)。 应用可通过将内置服务注入组件来使用这些服务。 应用还可定义和注册自定义服务，并通过 DI 使其在整个应用中可用。

DI 是一种技术，它用于访问配置在中心位置的服务。 该技术可在 Blazor 应用中用于以下方面：

* 跨多个组件共享服务类的单个实例，称为“单一实例”服务  。
* 通过使用引用抽象将组件与具体服务类分离。 例如，请考虑用接口 `IDataAccess` 来访问应用中数据。 该接口由具体的 `DataAccess` 类实现，并在应用的服务容器中注册为服务。 当组件使用 DI 接收 `IDataAccess` 实现时，组件不与具体类型耦合。 可交换实现，目的可能是为了单元测试中的模拟实现。

## <a name="default-services"></a>默认服务

默认服务会自动添加到应用的服务集合中。

| 服务 | 生存期 | 描述 |
| ------- | -------- | ----------- |
| <xref:System.Net.Http.HttpClient> | Singleton | 提供用于发送 HTTP 请求以及从 URI 标识的资源接收 HTTP 响应的方法。<br><br>Blazor WebAssembly 应用中 `HttpClient` 的实例使用浏览器在后台处理 HTTP 流量。<br><br>默认情况下，Blazor 服务器应用不包含配置为服务的 `HttpClient`。 向 Blazor 服务器应用提供 `HttpClient`。<br><br>有关详细信息，请参阅 <xref:blazor/call-web-api>。 |
| `IJSRuntime` | Singleton (Blazor WebAssembly)<br>Scoped（Blazor 服务器） | 表示在其中调度 JavaScript 调用的 JavaScript 运行时实例。 有关详细信息，请参阅 <xref:blazor/call-javascript-from-dotnet>。 |
| `NavigationManager` | Singleton (Blazor WebAssembly)<br>Scoped（Blazor 服务器） | 包含用于处理 URI 和导航状态的帮助程序。 有关详细信息，请参阅 [URI 和导航状态帮助程序](xref:blazor/routing#uri-and-navigation-state-helpers)。 |

自定义服务提供程序不会自动提供表中列出的默认服务。 如果你使用自定义服务提供程序且需要表中所示的任何服务，请将所需服务添加到新的服务提供程序。

## <a name="add-services-to-an-app"></a>向应用添加服务

### <a name="blazor-webassembly"></a>Blazor WebAssembly

在 Program.cs 的 `Main` 方法中配置应用服务集合的服务  。 在下例中，为 `IMyDependency` 注册了 `MyDependency` 实现：

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var builder = WebAssemblyHostBuilder.CreateDefault(args);
        builder.Services.AddSingleton<IMyDependency, MyDependency>();
        builder.RootComponents.Add<App>("app");

        await builder.Build().RunAsync();
    }
}
```

生成主机后，可在呈现任何组件之前从根 DI 范围访问服务。 这对于在呈现内容之前运行初始化逻辑而言很有用：

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var builder = WebAssemblyHostBuilder.CreateDefault(args);
        builder.Services.AddSingleton<WeatherService>();
        builder.RootComponents.Add<App>("app");

        var host = builder.Build();

        var weatherService = host.Services.GetRequiredService<WeatherService>();
        await weatherService.InitializeWeatherAsync();

        await host.RunAsync();
    }
}
```

主机还会为应用提供中心配置实例。 在上述示例的基础上，天气服务的 URL 将从默认配置源（例如 appsettings.json）传递到 `InitializeWeatherAsync`  ：

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var builder = WebAssemblyHostBuilder.CreateDefault(args);
        builder.Services.AddSingleton<WeatherService>();
        builder.RootComponents.Add<App>("app");

        var host = builder.Build();

        var weatherService = host.Services.GetRequiredService<WeatherService>();
        await weatherService.InitializeWeatherAsync(
            host.Configuration["WeatherServiceUrl"]);

        await host.RunAsync();
    }
}
```

### <a name="blazor-server"></a>Blazor 服务器

创建新应用后，请检查 `Startup.ConfigureServices` 方法：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add custom services here
}
```

向 `ConfigureServices` 方法传递 <xref:Microsoft.Extensions.DependencyInjection.IServiceCollection>，它是服务描述符对象 (<xref:Microsoft.Extensions.DependencyInjection.ServiceDescriptor>) 的列表。 通过向服务集合提供服务描述符来添加服务。 下面的示例演示了 `IDataAccess` 接口的概念及其具体实现 `DataAccess`：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IDataAccess, DataAccess>();
}
```

### <a name="service-lifetime"></a>服务生存期

可使用下表中所示的生存期配置服务。

| 生存期 | 描述 |
| -------- | ----------- |
| <xref:Microsoft.Extensions.DependencyInjection.ServiceDescriptor.Scoped*> | Blazor WebAssembly 应用当前没有 DI 范围的概念。 已注册 `Scoped` 的服务的行为与 `Singleton` 服务类似。 但是，Blazor 服务器托管模型支持 `Scoped` 生存期。 在 Blazor 服务器应用中，Scoped 服务注册的范围为“连接”  。 因此，即使当前意图是在浏览器中运行客户端，对于范围应限定为当前用户的服务来说，首选使用 Scoped 服务。 |
| <xref:Microsoft.Extensions.DependencyInjection.ServiceDescriptor.Singleton*> | DI 创建服务的单个实例  。 需要 `Singleton` 服务的所有组件都会接收同一服务的实例。 |
| <xref:Microsoft.Extensions.DependencyInjection.ServiceDescriptor.Transient*> | 每当组件从服务容器获取 `Transient` 服务的实例时，它都会接收该服务的新实例  。 |

DI 系统基于 ASP.NET Core 中的 DI 系统。 有关详细信息，请参阅 <xref:fundamentals/dependency-injection>。

## <a name="request-a-service-in-a-component"></a>在组件中请求服务

将服务添加到服务集合后，使用 Razor 指令 [\@inject](xref:mvc/views/razor#inject) 将服务注入组件。 `@inject` 具有两个参数：

* Type &ndash; 要注入的服务类型。
* Property &ndash; 接收注入的应用服务的属性名称。 属性无需手动创建。 编译器会创建属性。

有关详细信息，请参阅 <xref:mvc/views/dependency-injection>。

使用多个 `@inject` 语句注入不同的服务。

下面的示例说明如何使用 `@inject`。 将实现 `Services.IDataAccess` 的服务注入组件的 `DataRepository` 属性中。 请注意代码是如何仅使用 `IDataAccess` 抽象的：

[!code-razor[](dependency-injection/samples_snapshot/3.x/CustomerList.razor?highlight=2-3,23)]

在内部，生成的属性 (`DataRepository`) 使用 `InjectAttribute` 特性。 通常，不直接使用此特性。 如果组件需要基类，而基类也需要注入的属性，则请手动添加 `InjectAttribute`：

```csharp
public class ComponentBase : IComponent
{
    // DI works even if using the InjectAttribute in a component's base class.
    [Inject]
    protected IDataAccess DataRepository { get; set; }
    ...
}
```

在从基类派生得到的组件中，不需要 `@inject` 指令。 基类的 `InjectAttribute` 就已足够：

```razor
@page "/demo"
@inherits ComponentBase

<h1>Demo Component</h1>
```

## <a name="use-di-in-services"></a>在服务中使用 DI

复杂的服务可能需要其他服务。 在前面的示例中，`DataAccess` 可能需要 `HttpClient` 默认服务。 `@inject`（或 `InjectAttribute`）不可用于服务。 必须改用构造函数注入  。 通过向服务的构造函数添加参数来添加所需服务。 当 DI 创建服务时，它会在构造函数中识别其所需的服务，并相应地提供这些服务。

```csharp
public class DataAccess : IDataAccess
{
    // The constructor receives an HttpClient via dependency
    // injection. HttpClient is a default service.
    public DataAccess(HttpClient client)
    {
        ...
    }
}
```

构造函数注入的先决条件：

* 必须存在一个构造函数，其参数可完全通过 DI 实现。 如果指定默认值，则允许使用 DI 未涵盖的其他参数。
* 适用的构造函数必须是公共函数  。
* 必须存在一个适用的构造函数。 如果出现歧义，DI 会引发异常。

## <a name="utility-base-component-classes-to-manage-a-di-scope"></a>用于管理 DI 范围的实用工具基组件类

在 ASP.NET Core 应用中，Scoped 服务的范围通常限定为当前请求。 请求完成后，DI 系统将处置所有 Scoped 或 Transient 服务。 在 Blazor 服务器应用中，请求范围会在客户端连接期间一直持续，这可能导致 Transient 和 Scoped 服务的生存期比预期要长得多。 在 Blazor WebAssembly 应用中，已注册 Scoped 生存期的服务被视为单一实例，因此它们的生存期比典型 ASP.NET Core 应用中的 Scoped 服务要长。

限制 Blazor 应用中服务生存期的一种方法是使用 `OwningComponentBase` 类型。 `OwningComponentBase` 是派生自 `ComponentBase` 的一种抽象类型，它会创建与组件生存期相对应的 DI 范围。 通过使用此范围，可使用具有 Scoped 生存期的 DI 服务，并使其生存期与组件的生存期一样长。 销毁组件时，也会处置组件的 Scoped 服务提供程序提供的服务。 这对以下服务很有用：

* 由于 Transient 生存期不适用而应在组件中重复使用的服务。
* 由于 Singleton 生存期不适用而不得跨组件共享的服务。

可使用下面两个版本的 `OwningComponentBase` 类型：

* `OwningComponentBase` 是 `ComponentBase` 类型的抽象、可释放子级，其具有 `IServiceProvider` 类型的受保护的 `ScopedServices` 属性。 此提供程序可用于解析范围限定为组件生存期的服务。

  使用 `@inject` 或 `InjectAttribute` (`[Inject]`) 注入到组件中的 DI 服务不是在组件的范围中创建的。 要使用组件的范围，必须使用 `ScopedServices.GetRequiredService` 或 `ScopedServices.GetService` 解析服务。 任何使用 `ScopedServices` 提供程序进行解析的服务都具有从同一范围提供的依赖关系。

  ```razor
  @page "/preferences"
  @using Microsoft.Extensions.DependencyInjection
  @inherits OwningComponentBase

  <h1>User (@UserService.Name)</h1>

  <ul>
      @foreach (var setting in SettingService.GetSettings())
      {
          <li>@setting.SettingName: @setting.SettingValue</li>
      }
  </ul>

  @code {
      private IUserService UserService { get; set; }
      private ISettingService SettingService { get; set; }

      protected override void OnInitialized()
      {
          UserService = ScopedServices.GetRequiredService<IUserService>();
          SettingService = ScopedServices.GetRequiredService<ISettingService>();
      }
  }
  ```

* `OwningComponentBase<T>` 派生自 `OwningComponentBase`，并添加了一个 `Service` 属性，该属性从 Scoped DI 提供程序返回 `T` 的实例。 当存在一项应用需要从使用组件范围的 DI 容器中获取的主服务时，不必使用 `IServiceProvider` 的实例即可通过此类型便捷地访问 Scoped 服务。 `ScopedServices` 属性可用，因此应用可获取其他类型的服务（如有必要）。

  ```razor
  @page "/users"
  @attribute [Authorize]
  @inherits OwningComponentBase<AppDbContext>

  <h1>Users (@Service.Users.Count())</h1>

  <ul>
      @foreach (var user in Service.Users)
      {
          <li>@user.UserName</li>
      }
  </ul>
  ```

## <a name="use-of-entity-framework-dbcontext-from-di"></a>使用来自 DI 的实体框架 DbContext

从 Web 应用中的 DI 检索的一种常见服务类型是实体框架 (EF) `DbContext` 对象。 默认情况下，使用 `IServiceCollection.AddDbContext` 注册 EF 服务会将 `DbContext` 添加为一项 Scoped 服务。 注册为 Scoped 服务可能会导致 Blazor 应用中出现问题，因为这会导致 `DbContext` 实例生存期较长且跨应用共享。 `DbContext` 不是线程安全的且不得同时使用。

根据应用的不同，使用 `OwningComponentBase` 将 `DbContext` 的范围限制为单个组件可能会解决此问题  。 如果组件不并行使用 `DbContext`，则从 `OwningComponentBase` 派生该组件并从 `ScopedServices` 检索 `DbContext` 就已足够，因为它可确保：

* 单独的组件不共享 `DbContext`。
* `DbContext` 的生存期与依赖它的组件的生存期一样长。

如果单个组件可能同时使用 `DbContext`（例如用户每次选择一个按钮），则即使使用 `OwningComponentBase` 也不能避免并发 EF 操作问题。 在这种情况下，请对每个逻辑 EF 操作使用不同的 `DbContext`。 请使用下述任一方法：

* 使用 `DbContextOptions<TContext>` 作为参数直接创建 `DbContext`，这可从 DI 进行检索且是线程安全的。

    ```razor
    @page "/example"
    @inject DbContextOptions<AppDbContext> DbContextOptions

    <ul>
        @foreach (var item in _data)
        {
            <li>@item</li>
        }
    </ul>

    <button @onclick="LoadData">Load Data</button>

    @code {
        private List<string> _data = new List<string>();

        private async Task LoadData()
        {
            _data = await GetAsync();
            StateHasChanged();
        }

        public async Task<List<string>> GetAsync()
        {
            using (var context = new AppDbContext(DbContextOptions))
            {
                return await context.Products.Select(p => p.Name).ToListAsync();
            }
        }
    }
    ```

* 在具有 Transient 生存期的服务容器中注册 `DbContext`：
  * 注册上下文时，请使用 `ServiceLifetime.Transient`。 `AddDbContext` 扩展方法采用两个 `ServiceLifetime` 类型的可选参数。 若要使用此方法，则只有 `contextLifetime` 参数需要设为 `ServiceLifetime.Transient`。 `optionsLifetime` 可保留其默认值 `ServiceLifetime.Scoped`。

    ```csharp
    services.AddDbContext<AppDbContext>(options =>
         options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")),
         ServiceLifetime.Transient);
    ```  

  * 可将暂时性 `DbContext` 正常注入（使用 `@inject`）到不会并行执行多个 EF 操作的组件。 可能同时执行多个 EF 操作的人员可使用 `IServiceProvider.GetRequiredService` 为每个并行操作请求单独的 `DbContext` 对象。

    ```razor
    @page "/example"
    @using Microsoft.Extensions.DependencyInjection
    @inject IServiceProvider ServiceProvider

    <ul>
        @foreach (var item in _data)
        {
            <li>@item</li>
        }
    </ul>

    <button @onclick="LoadData">Load Data</button>

    @code {
        private List<string> _data = new List<string>();

        private async Task LoadData()
        {
            _data = await GetAsync();
            StateHasChanged();
        }

        public async Task<List<string>> GetAsync()
        {
            using (var context = ServiceProvider.GetRequiredService<AppDbContext>())
            {
                return await context.Products.Select(p => p.Name).ToListAsync();
            }
        }
    }
    ```

## <a name="additional-resources"></a>其他资源

* <xref:fundamentals/dependency-injection>
* <xref:mvc/views/dependency-injection>
