---
title: 托管和部署 ASP.NET Core Blazor
author: guardrex
description: 了解如何托管和部署 Blazor 应用。
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 03/11/2020
no-loc:
- Blazor
- SignalR
uid: host-and-deploy/blazor/index
ms.openlocfilehash: ddf70da29a82d462422c1bdf74ff45b92bb10b56
ms.sourcegitcommit: 5bdc54162d7dea8d9fa54ac3055678db23586af1
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/17/2020
ms.locfileid: "79434260"
---
# <a name="host-and-deploy-aspnet-core-blazor"></a>托管和部署 ASP.NET Core Blazor

作者：[Luke Latham](https://github.com/guardrex)、[Rainer Stropek](https://www.timecockpit.com) 和 [Daniel Roth](https://github.com/danroth27)

[!INCLUDE[](~/includes/blazorwasm-preview-notice.md)]

## <a name="publish-the-app"></a>发布应用

发布应用，用于发布配置中的部署。

# <a name="visual-studio"></a>[Visual Studio](#tab/visual-studio)

1. 从导航栏中选择“生成”   > “发布{应用程序}”  。
1. 选择“发布目标”  。 若要在本地发布，请选择“文件夹”  。
1. 接受“选择文件夹”  字段中的默认位置，或指定其他位置。 选择“发布”  按钮。

# <a name="net-core-cli"></a>[.NET Core CLI](#tab/netcore-cli)

使用 [dotnet publish](/dotnet/core/tools/dotnet-publish) 命令发布具有发布配置的应用：

```dotnetcli
dotnet publish -c Release
```

---

创建部署资产前，发布应用将触发项目依赖项的[还原](/dotnet/core/tools/dotnet-restore)并[生成](/dotnet/core/tools/dotnet-build)生成该项目。 在生成过程期间，将删除未使用的方法和程序集，以减少应用下载大小并缩短加载时间。

发布位置：

* Blazor WebAssembly
  * 独立 &ndash; 应用将发布到 /bin/Release/{TARGET FRAMEWORK}/publish/wwwroot  文件夹。 若要将应用部署为静态站点，请将 wwwroot 文件夹的内容复制到静态站点主机  。
  * 托管 &ndash; 客户端 Blazor WebAssembly 应用将发布到服务器应用的 /bin/Release/{TARGET FRAMEWORK}/publish/wwwroot 文件夹，以及服务器应用的任何其他静态 Web 资产  。 将 publish 文件夹的内容部署到主机  。
* Blazor 服务器 &ndash; 应用将发布到 /bin/Release/{TARGET FRAMEWORK}/publish 文件夹  。 将 publish 文件夹的内容部署到主机  。

文件夹中的资产将部署到 Web 服务器。 部署可能是手动或自动化过程，具体取决于使用的开发工具。

## <a name="app-base-path"></a>应用基路径

应用基路径是应用的根 URL 路径。  请考虑使用下列 ASP.NET Core 应用和 Blazor 子应用：

* ASP.NET Core 应用名为 `MyApp`：
  * 该应用实际驻留在 d:/MyApp 中  。
  * 在 `https://www.contoso.com/{MYAPP RESOURCE}` 收到请求。
* 名为 `CoolApp` 的 Blazor 应用是 `MyApp` 的子应用：
  * 子应用实际驻留在 d:/MyApp/CoolApp 中  。
  * 在 `https://www.contoso.com/CoolApp/{COOLAPP RESOURCE}` 收到请求。

如果不为 `CoolApp` 指定其他配置，此方案中的子应用将不知道其在服务器上的位置。 例如，不知道它驻留在相对 URL 路径 `/CoolApp/` 上，应用就无法构造其资源的正确相对 URL。

要为 Blazor 应用的基路径 `https://www.contoso.com/CoolApp/` 提供配置，请将 `<base>` 标记的 `href` 属性设置为 Pages/_Host.cshtml 文件（Blazor 服务器）或 wwwroot/index.html 文件 (Blazor WebAssembly) 中的相对根路径   ：

```html
<base href="/CoolApp/">
```

Blazor 服务器应用还会在应用的请求管道 `Startup.Configure` 中调用 <xref:Microsoft.AspNetCore.Builder.UsePathBaseExtensions.UsePathBase*>，设置服务器端基路径：

```csharp
app.UsePathBase("/CoolApp");
```

提供相对 URL 路径，不再根目录中的元件可以构造相对于应用根路径的 URL。 位于目录结构不同级别的组件可生成指向整个应用其他位置的资源链接。 应用基路径也用于拦截所选超链接，其中链接的 `href` 目标位于应用基路径 URI 空间中。 Blazor 路由器负责处理内部导航。

在许多托管方案中，应用的相对 URL 路径为应用的根目录。 在这些情况下，应用的相对 URL 基路径为正斜杠 (`<base href="/" />`)，它是 Blazor 应用的默认配置。 在其他托管方案中（例如 GitHub 页和 IIS 子应用），应用基路径必须设置为应用的服务器相对 URL 路径。

要设置应用的基路径，请更新 Pages/_Host.cshtml 文件（Blazor 服务器）或 wwwroot/index.html 文件 (Blazor WebAssembly) 的 `<head>` 标记元素中的 `<base>` 标记   。 将 `href` 属性值设置为 `/{RELATIVE URL PATH}/`（需要尾部反斜杠），其中 `{RELATIVE URL PATH}` 是应用完整相对 URL 路径。

对于具有非根相对 URL 路径的 Blazor WebAssembly 应用（例如 `<base href="/CoolApp/">`），应用在本地运行时无法查找其资源  。 要在本地开发和测试过程中解决此问题，可提供 path base 参数，用于匹配运行时 `<base>` 标记的 `href` 值  。 不要包含尾部反斜杠。 在本地运行应用时，若要传递路径基础参数，请使用 `--pathbase` 选项从应用的目录执行 `dotnet run` 命令：

```dotnetcli
dotnet run --pathbase=/{RELATIVE URL PATH (no trailing slash)}
```

对于具有相对 URL 路径 `/CoolApp/` (`<base href="/CoolApp/">`) 的 Blazor WebAssembly 应用，命令如下：

```dotnetcli
dotnet run --pathbase=/CoolApp
```

Blazor WebAssembly 应用在`http://localhost:port/CoolApp` 本地响应。

## <a name="deployment"></a>部署

有关部署指南，请参阅以下主题：

* <xref:host-and-deploy/blazor/webassembly>
* <xref:host-and-deploy/blazor/server>
