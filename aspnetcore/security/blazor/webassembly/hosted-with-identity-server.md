---
title: 使用标识服务器保护 ASP.NET Core Blazor WebAssembly 托管应用
author: guardrex
description: 使用[IdentityServer](https://identityserver.io/)后端在 Visual Studio 中创建具有身份验证的新 Blazor 托管应用
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 03/16/2020
no-loc:
- Blazor
- SignalR
uid: security/blazor/webassembly/hosted-with-identity-server
ms.openlocfilehash: 6c7942a827d88a620e6f295af3f523c23f4b3890
ms.sourcegitcommit: 91dc1dd3d055b4c7d7298420927b3fd161067c64
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/24/2020
ms.locfileid: "80219046"
---
# <a name="secure-an-aspnet-core-opno-locblazor-webassembly-hosted-app-with-identity-server"></a>使用标识服务器保护 ASP.NET Core Blazor WebAssembly 托管应用

作者： [Javier Calvarro 使用](https://github.com/javiercn)和[Luke Latham](https://github.com/guardrex)

[!INCLUDE[](~/includes/blazorwasm-preview-notice.md)]

[!INCLUDE[](~/includes/blazorwasm-3.2-template-article-notice.md)]

若要在 Visual Studio 中创建新的 Blazor 托管应用，使用[IdentityServer](https://identityserver.io/)对用户和 API 调用进行身份验证：

1. 使用 Visual Studio 创建新的 **Blazor WebAssembly**应用。 有关详细信息，请参阅 <xref:blazor/get-started>。
1. 在 "**新建 Blazor 应用**" 对话框中，在 "**身份验证**" 部分中选择 "**更改**"。
1. 选择后跟 **"确定"** 的**单个用户帐户**。
1. 选中 "**高级**" 部分中的 " **ASP.NET Core 托管**" 复选框。
1. 选择“创建”按钮。

若要在命令外壳中创建应用，请执行以下命令：

```dotnetcli
dotnet new blazorwasm -au Individual -ho
```

若要指定输出位置（如果它不存在，则创建一个项目文件夹），请在命令中包含 output 选项，其中包含一个路径（例如 `-o BlazorSample`）。 文件夹名称还会成为项目名称的一部分。

## <a name="server-app-configuration"></a>服务器应用配置

以下各节介绍了在包括身份验证支持时对项目的添加。

### <a name="startup-class"></a>Startup 类

`Startup` 类具有以下附加项：

* 在 `Startup.ConfigureServices` 中：

  * 具有默认 UI 的标识：

    ```csharp
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlite(Configuration.GetConnectionString("DefaultConnection")));

    services.AddDefaultIdentity<ApplicationUser>()
        .AddDefaultUI(UIFramework.Bootstrap4)
        .AddEntityFrameworkStores<ApplicationDbContext>();
    ```

  * 使用附加的 <xref:Microsoft.Extensions.DependencyInjection.IdentityServerBuilderConfigurationExtensions.AddApiAuthorization%2A> 帮助器方法，可在 IdentityServer 上设置某些默认的 ASP.NET Core 约定：

    ```csharp
    services.AddIdentityServer()
        .AddApiAuthorization<ApplicationUser, ApplicationDbContext>();
    ```

  * 使用附加的 <xref:Microsoft.AspNetCore.Authentication.AuthenticationBuilderExtensions.AddIdentityServerJwt%2A> 帮助器方法进行身份验证，该方法将应用程序配置为验证 IdentityServer 生成的 JWT 令牌：

    ```csharp
    services.AddAuthentication()
        .AddIdentityServerJwt();
    ```

* 在 `Startup.Configure` 中：

  * 负责验证请求凭据并在请求上下文上设置用户的身份验证中间件：

    ```csharp
    app.UseAuthentication();
    ```

  * 公开 Open ID Connect （OIDC）终结点的 IdentityServer 中间件：

    ```csharp
    app.UseIdentityServer();
    ```

### <a name="addapiauthorization"></a>AddApiAuthorization

<xref:Microsoft.Extensions.DependencyInjection.IdentityServerBuilderConfigurationExtensions.AddApiAuthorization%2A> helper 方法为 ASP.NET Core 情况配置[IdentityServer](https://identityserver.io/) 。 IdentityServer 是一个功能强大且可扩展的框架，用于处理应用安全问题。 在最常见的情况下，IdentityServer 会造成不必要的复杂性。 因此，我们考虑到了一个很好的起点。 身份验证需要更改后，IdentityServer 的全部功能仍可用于自定义身份验证，以满足应用程序的要求。

### <a name="addidentityserverjwt"></a>AddIdentityServerJwt

<xref:Microsoft.AspNetCore.Authentication.AuthenticationBuilderExtensions.AddIdentityServerJwt%2A> helper 方法为应用配置策略方案，作为默认身份验证处理程序。 此策略配置为允许标识处理路由到标识 URL 空间中任何子路径的所有请求 `/Identity`。 <xref:Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerHandler> 处理所有其他请求。 此外，此方法：

* 使用 `{APPLICATION NAME}API`的默认范围向 IdentityServer 注册 `{APPLICATION NAME}API` API 资源。
* 将 JWT 持有者令牌中间件配置为验证 IdentityServer 为应用程序颁发的令牌。

### <a name="weatherforecastcontroller"></a>WeatherForecastController

在 `WeatherForecastController` （*控制器/WeatherForecastController*）中， [`[Authorize]`](xref:Microsoft.AspNetCore.Authorization.AuthorizeAttribute)属性应用于类。 属性指示用户必须根据默认策略进行授权才能访问资源。 默认授权策略配置为使用默认身份验证方案，该方案由 <xref:Microsoft.AspNetCore.Authentication.AuthenticationBuilderExtensions.AddIdentityServerJwt%2A> 设置为之前提到的策略方案。 Helper 方法将 <xref:Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerHandler> 配置为应用程序请求的默认处理程序。

### <a name="applicationdbcontext"></a>ApplicationDbContext

在 `ApplicationDbContext` （*Data/ApplicationDbContext*）中，同一 <xref:Microsoft.EntityFrameworkCore.DbContext> 用于标识，但它扩展 <xref:Microsoft.AspNetCore.ApiAuthorization.IdentityServer.ApiAuthorizationDbContext%601> 以包括 IdentityServer 的架构。 <xref:Microsoft.AspNetCore.ApiAuthorization.IdentityServer.ApiAuthorizationDbContext%601> 派生自 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext>。

若要完全控制数据库架构，请从某个可用的标识 <xref:Microsoft.EntityFrameworkCore.DbContext> 类继承，并通过在 `OnModelCreating` 方法中调用 `builder.ConfigurePersistedGrantContext(_operationalStoreOptions.Value)`，将上下文配置为包含标识架构。

### <a name="oidcconfigurationcontroller"></a>OidcConfigurationController

在 `OidcConfigurationController` （controller */OidcConfigurationController*）中，客户端终结点配置为提供 OIDC 参数。

### <a name="app-settings-files"></a>应用设置文件

在项目根目录下的应用设置文件（*appsettings*）中，`IdentityServer` 部分介绍了已配置的客户端的列表。 在下面的示例中，有一个客户端。 客户端名称对应于应用名称，并按约定映射到 OAuth `ClientId` 参数。 配置文件指示正在配置的应用类型。 此配置文件可在内部使用，以促进简化服务器配置过程的约定。 <!-- There are several profiles available, as explained in the [Application profiles](#application-profiles) section. -->

```json
"IdentityServer": {
  "Clients": {
    "BlazorApplicationWithAuthentication.Client": {
      "Profile": "IdentityServerSPA"
    }
  }
}
```

开发环境中的应用设置文件（*appsettings）。开发*）在项目根目录下，`IdentityServer` 部分介绍了用于对令牌进行签名的密钥。 <!-- When deploying to production, a key needs to be provisioned and deployed alongside the app, as explained in the [Deploy to production](#deploy-to-production) section. -->

```json
"IdentityServer": {
  "Key": {
    "Type": "Development"
  }
}
```

## <a name="client-app-configuration"></a>客户端应用配置

### <a name="authentication-package"></a>身份验证包

创建应用以使用单个用户帐户（`Individual`）时，应用会自动在应用的项目文件中接收 `Microsoft.AspNetCore.Components.WebAssembly.Authentication` 包的包引用。 包提供一组基元，可帮助应用对用户进行身份验证，并获取令牌以调用受保护的 Api。

如果向应用程序中添加身份验证，请将包手动添加到应用的项目文件中：

```xml
<PackageReference 
    Include="Microsoft.AspNetCore.Components.WebAssembly.Authentication" 
    Version="{VERSION}" />
```

将前面的包引用中的 `{VERSION}` 替换为 <xref:blazor/get-started> 一文中所示 `Microsoft.AspNetCore.Blazor.Templates` 包的版本。

### <a name="api-authorization-support"></a>API 授权支持

通过 `Microsoft.AspNetCore.Components.WebAssembly.Authentication` 包内提供的扩展方法，将对用户进行身份验证的支持插入到服务容器中。 此方法设置应用与现有授权系统交互所需的所有服务。

```csharp
builder.Services.AddApiAuthorization();
```

默认情况下，它按约定从 `_configuration/{client-id}`加载应用的配置。 按照约定，将客户端 ID 设置为应用的程序集名称。 可以通过使用选项调用重载，将此 URL 更改为指向不同的终结点。

### <a name="index-page"></a>索引页面

[!INCLUDE[](~/includes/blazor-security/index-page-authentication.md)]

### <a name="app-component"></a>应用组件

[!INCLUDE[](~/includes/blazor-security/app-component.md)]

### <a name="redirecttologin-component"></a>RedirectToLogin 组件

[!INCLUDE[](~/includes/blazor-security/redirecttologin-component.md)]

### <a name="logindisplay-component"></a>LoginDisplay 组件

`LoginDisplay` 组件（*shared/LoginDisplay*）在 `MainLayout` 组件（*shared/MainLayout*）中呈现并管理以下行为：

* 对于经过身份验证的用户：
  * 显示当前用户名。
  * 提供指向 ASP.NET Core 标识中的用户配置文件页的链接。
  * 提供用于注销应用的按钮。
* 对于匿名用户：
  * 提供注册的选项。
  * 提供用于登录的选项。

```razor
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Components.WebAssembly.Authentication
@inject NavigationManager Navigation
@inject SignOutSessionStateManager SignOutManager

<AuthorizeView>
    <Authorized>
        <a href="authentication/profile">Hello, @context.User.Identity.Name!</a>
        <button class="nav-link btn btn-link" @onclick="BeginSignOut">
            Log out
        </button>
    </Authorized>
    <NotAuthorized>
        <a href="authentication/register">Register</a>
        <a href="authentication/login">Log in</a>
    </NotAuthorized>
</AuthorizeView>

@code {
    private async Task BeginSignOut(MouseEventArgs args)
    {
        await SignOutManager.SetSignOutState();
        Navigation.NavigateTo("authentication/logout");
    }
}
```

### <a name="authentication-component"></a>身份验证组件

[!INCLUDE[](~/includes/blazor-security/authentication-component.md)]

### <a name="fetchdata-component"></a>FetchData 组件

[!INCLUDE[](~/includes/blazor-security/fetchdata-component.md)]

## <a name="run-the-app"></a>运行应用

从服务器项目运行应用。 使用 Visual Studio 时，请在**解决方案资源管理器**中选择服务器项目，并在工具栏中选择 "**运行**" 按钮，或从 "**调试**" 菜单启动应用程序。

[!INCLUDE[](~/includes/blazor-security/troubleshoot.md)]
