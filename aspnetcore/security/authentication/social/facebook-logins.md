---
title: 在 ASP.NET Core Facebook 外部登录安装程序
author: rick-anderson
description: 包含代码示例的教程演示如何将 Facebook 帐户用户身份验证集成到现有 ASP.NET Core 应用。
ms.author: riande
ms.custom: seoapril2019, mvc, seodec18
ms.date: 03/19/2020
monikerRange: '>= aspnetcore-3.0'
uid: security/authentication/facebook-logins
ms.openlocfilehash: bb26a27f026e744c7d4925aa2281bf0625fff8a2
ms.sourcegitcommit: 9b6e7f421c243963d5e419bdcfc5c4bde71499aa
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/21/2020
ms.locfileid: "79989783"
---
# <a name="facebook-external-login-setup-in-aspnet-core"></a>在 ASP.NET Core Facebook 外部登录安装程序

作者：[Valeriy Novytskyy](https://github.com/01binary) 和 [Rick Anderson](https://twitter.com/RickAndMSFT)

本教程中的代码示例演示如何使用户能够使用在[前一页](xref:security/authentication/social/index)上创建的示例 ASP.NET Core 3.0 项目登录其 Facebook 帐户。 首先，我们要按照[官方步骤](https://developers.facebook.com)创建 FACEBOOK 应用 ID。

## <a name="create-the-app-in-facebook"></a>在 Facebook 中创建应用

* 将[AspNetCore](https://www.nuget.org/packages/Microsoft.AspNetCore.Authentication.Facebook) NuGet 包添加到项目。

* 导航到[Facebook 开发人员应用](https://developers.facebook.com/apps/)页面并登录。 如果还没有 Facebook 帐户，请使用登录页上的 "**注册 Facebook** " 链接创建一个。  获得 Facebook 帐户后，请按照说明注册为 Facebook 开发人员。

* 从 "**我的应用**" 菜单中，选择 "**创建应用**" 以创建新的应用 ID。

   ![Microsoft Edge 中打开的 Facebook 开发人员门户](index/_static/FBMyApps.png)

* 填写表单，然后点击 "**创建应用 ID** " 按钮。

  ![创建新的应用 ID 窗体](index/_static/FBNewAppId.png)

* 在 "新建应用" 卡中，选择 "**添加产品**"。  在**Facebook 登录**卡上，单击 "**设置**" 

  ![产品安装程序页](index/_static/FBProductSetup.png)

* **快速入门**向导会启动，并**选择一个平台**作为第一页。 现在，通过单击左下方菜单中的 " **FaceBook 登录** **设置**" 链接，绕过向导：

  ![跳过快速入门](index/_static/FBSkipQuickStart.png)

* 将显示 "**客户端 OAuth 设置**" 页：

  ![客户端 OAuth 设置页](index/_static/FBOAuthSetup.png)

* 输入包含 */signin-facebook*的开发 URI，并将其追加到 "**有效的 OAuth 重定向 uri** " 字段中（例如： `https://localhost:44320/signin-facebook`）。 稍后在本教程中配置的 Facebook 身份验证将自动处理 */signin-facebook*路由中的请求以实现 OAuth 流。

> [!NOTE]
> URI */signin-facebook*设置为 facebook 身份验证提供程序的默认回调。 通过[FacebookOptions](/dotnet/api/microsoft.aspnetcore.authentication.facebook.facebookoptions)类的继承的[RemoteAuthenticationOptions. CallbackPath](/dotnet/api/microsoft.aspnetcore.authentication.remoteauthenticationoptions.callbackpath)属性配置 Facebook 身份验证中间件时，可以更改默认的回叫 URI。

* 单击“保存更改”。

* 单击左侧导航栏中的 "**设置**" > **基本**"链接。

  在此页上，记下你的 `App ID` 和 `App Secret`。 你将添加到 ASP.NET Core 应用程序下一节中：

* 部署站点时，需要重新访问**Facebook 登录**设置页面并注册新的公共 URI。

## <a name="store-the-facebook-app-id-and-secret"></a>存储 Facebook 应用 ID 和机密

用[机密管理器](xref:security/app-secrets)存储敏感设置，如 FACEBOOK 应用 ID 和机密值。 对于本示例，请使用以下步骤：

1. 按照[启用密钥存储](xref:security/app-secrets#enable-secret-storage)中的说明初始化密钥存储的项目。
1. 将敏感设置存储在本地密钥存储中，并将机密密钥 `Authentication:Facebook:AppId` 和 `Authentication:Facebook:AppSecret`：

    ```dotnetcli
    dotnet user-secrets set "Authentication:Facebook:AppId" "<app-id>"
    dotnet user-secrets set "Authentication:Facebook:AppSecret" "<app-secret>"
    ```

[!INCLUDE[](~/includes/environmentVarableColon.md)]

## <a name="configure-facebook-authentication"></a>配置 Facebook 身份验证

在*Startup.cs*文件的 `ConfigureServices` 方法中添加 Facebook 服务：

```csharp
services.AddAuthentication().AddFacebook(facebookOptions =>
{
    facebookOptions.AppId = Configuration["Authentication:Facebook:AppId"];
    facebookOptions.AppSecret = Configuration["Authentication:Facebook:AppSecret"];
});
```

[!INCLUDE [default settings configuration](includes/default-settings.md)]

[!INCLUDE[](includes/chain-auth-providers.md)]

有关 Facebook 身份验证支持的配置选项的详细信息，请参阅[FacebookOptions](/dotnet/api/microsoft.aspnetcore.builder.facebookoptions) API 参考。 配置选项可用于：

* 请求有关用户的不同信息。
* 添加查询字符串参数以自定义登录体验。

## <a name="sign-in-with-facebook"></a>使用 Facebook 登录

运行应用程序，并单击 "**登录"** 。 请参阅使用 Facebook 登录的选项。

![Web 应用程序： 用户未经过身份验证](index/_static/DoneFacebook.png)

单击**facebook**时，会重定向到 facebook 进行身份验证：

![Facebook 身份验证页面](index/_static/FBLogin.png)

Facebook 身份验证请求的默认公共配置文件和电子邮件地址：

![Facebook 身份验证页许可屏幕](index/_static/FBLoginDone.png)

输入你的 Facebook 凭据后你重定向回你的站点，你可以设置你的电子邮件。

现在已在使用 Facebook 凭据登录：

![Web 应用程序： 用户通过身份验证](index/_static/Done.png)

[!INCLUDE[Forward request information when behind a proxy or load balancer section](includes/forwarded-headers-middleware.md)]

## <a name="troubleshooting"></a>故障排除

* **仅 ASP.NET Core 2.x：** 如果未通过在 `ConfigureServices`中调用 `services.AddIdentity` 来配置标识，则尝试进行身份验证将导致*ArgumentException：必须提供 "SignInScheme" 选项*。 在本教程中使用的项目模板可确保，此操作。
* 如果尚未通过应用初始迁移来创建站点数据库，则在处理请求错误时，将会出现*数据库操作失败*的情况。 点击 "**应用迁移**" 以创建数据库，然后单击 "刷新" 以继续出现错误。

## <a name="next-steps"></a>后续步骤

* 本文介绍了您如何可以使用 Facebook 进行验证。 您可以遵循类似的方法向[前一页](xref:security/authentication/social/index)上列出的其他提供程序进行身份验证。

* 将网站发布到 Azure web 应用后，应重置 Facebook 开发人员门户中的 `AppSecret`。

* 将 `Authentication:Facebook:AppId` 和 `Authentication:Facebook:AppSecret` 设置为 Azure 门户中的应用程序设置。 配置系统设置以从环境变量读取密钥。
