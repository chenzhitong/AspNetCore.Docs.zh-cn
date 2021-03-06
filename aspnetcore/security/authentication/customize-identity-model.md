---
title: ASP.NET Core 中的标识模型自定义
author: ajcvickers
description: 本文介绍如何为 ASP.NET Core 标识自定义的基础的实体框架核心数据模型。
ms.author: avickers
ms.date: 07/01/2019
uid: security/authentication/customize_identity_model
ms.openlocfilehash: f549fdff4a416b5fadcb2b1078b051bbab8e402e
ms.sourcegitcommit: 9a129f5f3e31cc449742b164d5004894bfca90aa
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/06/2020
ms.locfileid: "78651456"
---
# <a name="identity-model-customization-in-aspnet-core"></a>ASP.NET Core 中的标识模型自定义

作者： [Arthur Vickers](https://github.com/ajcvickers)

ASP.NET Core 标识提供了一个框架，用于管理和存储 ASP.NET Core 应用中的用户帐户。 选择**单个用户帐户**作为身份验证机制时，会将标识添加到你的项目中。 默认情况下，标识使用实体框架（EF）核心数据模型。 本文介绍如何自定义标识模型。

## <a name="identity-and-ef-core-migrations"></a>标识和 EF Core 迁移

在检查模型之前，了解身份如何与[EF Core 迁移](/ef/core/managing-schemas/migrations/)来创建和更新数据库，这一点很有用。 在顶级，此过程如下：

1. [在代码中](/ef/core/modeling/)定义或更新数据模型。
1. 添加迁移，将此模型转换为可应用于数据库的更改。
1. 检查迁移是否正确表示你的意图。
1. 应用迁移以更新数据库，使其与模型保持同步。
1. 重复步骤1到4，进一步优化模型并使数据库保持同步。

使用以下方法之一来添加和应用迁移：

* 如果使用 Visual Studio，则为**包管理器控制台**（PMC）窗口。 有关详细信息，请参阅[EF CORE PMC 工具](/ef/core/miscellaneous/cli/powershell)。
* 使用命令行时的 .NET Core CLI。 有关详细信息，请参阅[EF Core .net 命令行工具](/ef/core/miscellaneous/cli/dotnet)。
* 当应用运行时，单击 "错误" 页上的 "**应用迁移**" 按钮。

ASP.NET Core 具有一个开发时错误页面处理程序。 在运行应用程序时，处理程序可以应用迁移。 生产应用通常从迁移生成 SQL 脚本，并将数据库更改作为受控应用和数据库部署的一部分进行部署。

当创建使用标识的新应用时，上面的步骤1和2已经完成。 也就是说，初始数据模型已存在，初始迁移已添加到该项目中。 初始迁移仍需应用于数据库。 可以通过以下方法之一来应用初始迁移：

* 在 PMC 中运行 `Update-Database`。
* 在命令行界面中运行 `dotnet ef database update`。
* 运行应用时，单击 "错误" 页上的 "**应用迁移**" 按钮。

对模型进行更改时重复前面的步骤。

## <a name="the-identity-model"></a>标识模型

### <a name="entity-types"></a>实体类型

标识模型包含以下实体类型。

|实体类型|说明                                                  |
|-----------|-------------------------------------------------------------|
|`User`     |表示用户。                                         |
|`Role`     |表示角色。                                           |
|`UserClaim`|表示用户拥有的声明。                    |
|`UserToken`|表示用户的身份验证令牌。               |
|`UserLogin`|将用户与登录名相关联。                              |
|`RoleClaim`|表示向角色内的所有用户授予的声明。|
|`UserRole` |关联用户和角色的联接实体。               |

### <a name="entity-type-relationships"></a>实体类型关系

[实体类型](#entity-types)通过以下方式彼此相关：

* 每个 `User` 可以有多个 `UserClaims`。
* 每个 `User` 可以有多个 `UserLogins`。
* 每个 `User` 可以有多个 `UserTokens`。
* 每个 `Role` 可以有多个关联的 `RoleClaims`。
* 每个 `User` 可以有多个关联的 `Roles`，每个 `Role` 可以与多个 `Users`关联。 这是一个多对多关系，需要数据库中的联接表。 联接表由 `UserRole` 实体表示。

### <a name="default-model-configuration"></a>默认模型配置

标识定义了许多继承自[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)的*上下文类*来配置和使用模型。 此配置是使用上下文类的[OnModelCreating](/dotnet/api/microsoft.entityframeworkcore.dbcontext.onmodelcreating)方法中的[EF CORE Code First 熟知 API](/ef/core/modeling/)完成的。 默认配置为：

```csharp
builder.Entity<TUser>(b =>
{
    // Primary key
    b.HasKey(u => u.Id);

    // Indexes for "normalized" username and email, to allow efficient lookups
    b.HasIndex(u => u.NormalizedUserName).HasName("UserNameIndex").IsUnique();
    b.HasIndex(u => u.NormalizedEmail).HasName("EmailIndex");

    // Maps to the AspNetUsers table
    b.ToTable("AspNetUsers");

    // A concurrency token for use with the optimistic concurrency checking
    b.Property(u => u.ConcurrencyStamp).IsConcurrencyToken();

    // Limit the size of columns to use efficient database types
    b.Property(u => u.UserName).HasMaxLength(256);
    b.Property(u => u.NormalizedUserName).HasMaxLength(256);
    b.Property(u => u.Email).HasMaxLength(256);
    b.Property(u => u.NormalizedEmail).HasMaxLength(256);

    // The relationships between User and other entity types
    // Note that these relationships are configured with no navigation properties

    // Each User can have many UserClaims
    b.HasMany<TUserClaim>().WithOne().HasForeignKey(uc => uc.UserId).IsRequired();

    // Each User can have many UserLogins
    b.HasMany<TUserLogin>().WithOne().HasForeignKey(ul => ul.UserId).IsRequired();

    // Each User can have many UserTokens
    b.HasMany<TUserToken>().WithOne().HasForeignKey(ut => ut.UserId).IsRequired();

    // Each User can have many entries in the UserRole join table
    b.HasMany<TUserRole>().WithOne().HasForeignKey(ur => ur.UserId).IsRequired();
});

builder.Entity<TUserClaim>(b =>
{
    // Primary key
    b.HasKey(uc => uc.Id);

    // Maps to the AspNetUserClaims table
    b.ToTable("AspNetUserClaims");
});

builder.Entity<TUserLogin>(b =>
{
    // Composite primary key consisting of the LoginProvider and the key to use
    // with that provider
    b.HasKey(l => new { l.LoginProvider, l.ProviderKey });

    // Limit the size of the composite key columns due to common DB restrictions
    b.Property(l => l.LoginProvider).HasMaxLength(128);
    b.Property(l => l.ProviderKey).HasMaxLength(128);

    // Maps to the AspNetUserLogins table
    b.ToTable("AspNetUserLogins");
});

builder.Entity<TUserToken>(b =>
{
    // Composite primary key consisting of the UserId, LoginProvider and Name
    b.HasKey(t => new { t.UserId, t.LoginProvider, t.Name });

    // Limit the size of the composite key columns due to common DB restrictions
    b.Property(t => t.LoginProvider).HasMaxLength(maxKeyLength);
    b.Property(t => t.Name).HasMaxLength(maxKeyLength);

    // Maps to the AspNetUserTokens table
    b.ToTable("AspNetUserTokens");
});

builder.Entity<TRole>(b =>
{
    // Primary key
    b.HasKey(r => r.Id);

    // Index for "normalized" role name to allow efficient lookups
    b.HasIndex(r => r.NormalizedName).HasName("RoleNameIndex").IsUnique();

    // Maps to the AspNetRoles table
    b.ToTable("AspNetRoles");

    // A concurrency token for use with the optimistic concurrency checking
    b.Property(r => r.ConcurrencyStamp).IsConcurrencyToken();

    // Limit the size of columns to use efficient database types
    b.Property(u => u.Name).HasMaxLength(256);
    b.Property(u => u.NormalizedName).HasMaxLength(256);

    // The relationships between Role and other entity types
    // Note that these relationships are configured with no navigation properties

    // Each Role can have many entries in the UserRole join table
    b.HasMany<TUserRole>().WithOne().HasForeignKey(ur => ur.RoleId).IsRequired();

    // Each Role can have many associated RoleClaims
    b.HasMany<TRoleClaim>().WithOne().HasForeignKey(rc => rc.RoleId).IsRequired();
});

builder.Entity<TRoleClaim>(b =>
{
    // Primary key
    b.HasKey(rc => rc.Id);

    // Maps to the AspNetRoleClaims table
    b.ToTable("AspNetRoleClaims");
});

builder.Entity<TUserRole>(b =>
{
    // Primary key
    b.HasKey(r => new { r.UserId, r.RoleId });

    // Maps to the AspNetUserRoles table
    b.ToTable("AspNetUserRoles");
});
```

### <a name="model-generic-types"></a>模型泛型类型

标识为上面列出的每个实体类型定义默认的[公共语言运行时](/dotnet/standard/glossary#clr)（CLR）类型。 这些类型都带有*标识*：

* `IdentityUser`
* `IdentityRole`
* `IdentityUserClaim`
* `IdentityUserToken`
* `IdentityUserLogin`
* `IdentityRoleClaim`
* `IdentityUserRole`

可以将类型用作应用自己的类型的基类，而不是直接使用这些类型。 标识定义的 `DbContext` 类是泛型类，因此，不同的 CLR 类型可用于模型中的一个或多个实体类型。 这些泛型类型还允许更改 `User` 主键（PK）数据类型。

将标识与支持角色一起使用时，应使用 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext> 类。 例如：

```csharp
// Uses all the built-in Identity types
// Uses `string` as the key type
public class IdentityDbContext
    : IdentityDbContext<IdentityUser, IdentityRole, string>
{
}

// Uses the built-in Identity types except with a custom User type
// Uses `string` as the key type
public class IdentityDbContext<TUser>
    : IdentityDbContext<TUser, IdentityRole, string>
        where TUser : IdentityUser
{
}

// Uses the built-in Identity types except with custom User and Role types
// The key type is defined by TKey
public class IdentityDbContext<TUser, TRole, TKey> : IdentityDbContext<
    TUser, TRole, TKey, IdentityUserClaim<TKey>, IdentityUserRole<TKey>,
    IdentityUserLogin<TKey>, IdentityRoleClaim<TKey>, IdentityUserToken<TKey>>
        where TUser : IdentityUser<TKey>
        where TRole : IdentityRole<TKey>
        where TKey : IEquatable<TKey>
{
}

// No built-in Identity types are used; all are specified by generic arguments
// The key type is defined by TKey
public abstract class IdentityDbContext<
    TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>
    : IdentityUserContext<TUser, TKey, TUserClaim, TUserLogin, TUserToken>
         where TUser : IdentityUser<TKey>
         where TRole : IdentityRole<TKey>
         where TKey : IEquatable<TKey>
         where TUserClaim : IdentityUserClaim<TKey>
         where TUserRole : IdentityUserRole<TKey>
         where TUserLogin : IdentityUserLogin<TKey>
         where TRoleClaim : IdentityRoleClaim<TKey>
         where TUserToken : IdentityUserToken<TKey>
```

也可以使用不带角色的标识（仅限声明），在这种情况下，应使用 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityUserContext%601> 类：

```csharp
// Uses the built-in non-role Identity types except with a custom User type
// Uses `string` as the key type
public class IdentityUserContext<TUser>
    : IdentityUserContext<TUser, string>
        where TUser : IdentityUser
{
}

// Uses the built-in non-role Identity types except with a custom User type
// The key type is defined by TKey
public class IdentityUserContext<TUser, TKey> : IdentityUserContext<
    TUser, TKey, IdentityUserClaim<TKey>, IdentityUserLogin<TKey>,
    IdentityUserToken<TKey>>
        where TUser : IdentityUser<TKey>
        where TKey : IEquatable<TKey>
{
}

// No built-in Identity types are used; all are specified by generic arguments, with no roles
// The key type is defined by TKey
public abstract class IdentityUserContext<
    TUser, TKey, TUserClaim, TUserLogin, TUserToken> : DbContext
        where TUser : IdentityUser<TKey>
        where TKey : IEquatable<TKey>
        where TUserClaim : IdentityUserClaim<TKey>
        where TUserLogin : IdentityUserLogin<TKey>
        where TUserToken : IdentityUserToken<TKey>
{
}
```

## <a name="customize-the-model"></a>自定义模型

模型自定义的起点是派生自适当的上下文类型。 请参阅[模型泛型类型](#model-generic-types)部分。 此上下文类型通常称为 `ApplicationDbContext`，由 ASP.NET Core 模板创建。

上下文用于通过两种方式配置模型：

* 为泛型类型参数提供实体和键类型。
* 重写 `OnModelCreating` 修改这些类型的映射。

重写 `OnModelCreating`时，应首先调用 `base.OnModelCreating`;接下来应调用重写配置。 EF Core 通常具有配置的最后一个 wins 策略。 例如，如果先使用一个表名称调用实体类型的 `ToTable` 方法，然后使用不同的表名称再次调用该方法，则使用第二个调用中的表名。

### <a name="custom-user-data"></a>自定义用户数据

<!--
set projNam=WebApp1
dotnet new webapp -o %projNam%
cd %projNam%
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design 
dotnet aspnet-codegenerator identity  -dc ApplicationDbContext --useDefaultUI 
dotnet ef migrations add CreateIdentitySchema
dotnet ef database update
 -->

[自定义用户数据](xref:security/authentication/add-user-data)是通过从 `IdentityUser`继承来支持的。 建议将此类型命名 `ApplicationUser`：

```csharp
public class ApplicationUser : IdentityUser
{
    public string CustomTag { get; set; }
}
```

使用 `ApplicationUser` 类型作为上下文的泛型参数：

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
    }
}
```

不需要重写 `ApplicationDbContext` 类中的 `OnModelCreating`。 EF Core 按约定映射 `CustomTag` 属性。 但是，需要更新数据库以创建新的 `CustomTag` 列。 若要创建该列，请添加迁移，然后根据[标识和 EF Core 迁移](#identity-and-ef-core-migrations)中所述更新数据库。

更新*Pages/Shared/_LoginPartial* ，并将 `IdentityUser` 替换为 `ApplicationUser`：

```cshtml
@using Microsoft.AspNetCore.Identity
@using WebApp1.Areas.Identity.Data
@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager
```

更新*Areas/Identity/IdentityHostingStartup*或 `Startup.ConfigureServices`，并将 `IdentityUser` 替换为 `ApplicationUser`。

```csharp
services.AddDefaultIdentity<ApplicationUser>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultUI();
```

在 ASP.NET Core 2.1 或更高版本中，标识作为 Razor 类库提供。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用 <xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果使用标识 scaffolder 将标识文件添加到项目中，请删除对 `AddDefaultUI`的调用。 有关详细信息，请参阅：

* [基架标识](xref:security/authentication/scaffold-identity)
* [向标识添加、下载和删除自定义用户数据](xref:security/authentication/add-user-data)

### <a name="change-the-primary-key-type"></a>更改主键类型

在创建数据库后，对 PK 列的数据类型的更改在许多数据库系统上都有问题。 更改 PK 通常涉及删除并重新创建表。 因此，在创建数据库时，应在初始迁移中指定密钥类型。

按照以下步骤更改 PK 类型：

1. 如果数据库是在 PK 更改之前创建的，请运行 `Drop-Database` （PMC）或 `dotnet ef database drop` （.NET Core CLI）以删除它。
2. 确认删除数据库后，删除带有 `Remove-Migration` （PMC）或 `dotnet ef migrations remove` （.NET Core CLI）的初始迁移。
3. 更新 `ApplicationDbContext` 类，使其从 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext%603>派生。 为 `TKey`指定新的密钥类型。 例如，若要使用 `Guid` 密钥类型：

    ```csharp
    public class ApplicationDbContext
        : IdentityDbContext<IdentityUser<Guid>, IdentityRole<Guid>, Guid>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
    ```

    ::: moniker range=">= aspnetcore-2.0"

    在前面的代码中，必须指定泛型类 <xref:Microsoft.AspNetCore.Identity.IdentityUser%601> 和 <xref:Microsoft.AspNetCore.Identity.IdentityRole%601> 才能使用新的密钥类型。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    在前面的代码中，必须指定泛型类 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityUser%601> 和 <xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityRole%601> 才能使用新的密钥类型。

    ::: moniker-end

    若要使用一般用户，必须更新 `Startup.ConfigureServices`：

    ::: moniker range=">= aspnetcore-2.1"

    ```csharp
    services.AddDefaultIdentity<IdentityUser<Guid>>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    ```csharp
    services.AddIdentity<IdentityUser<Guid>, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    ```csharp
    services.AddIdentity<IdentityUser<Guid>, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext, Guid>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

4. 如果正在使用自定义 `ApplicationUser` 类，请将类更新为从 `IdentityUser`继承。 例如：

    ::: moniker range="<= aspnetcore-1.1"

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Models/ApplicationUser.cs?name=snippet_ApplicationUser&highlight=4)]

    ::: moniker-end

    ::: moniker range=">= aspnetcore-2.0"

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationUser.cs?name=snippet_ApplicationUser&highlight=4)]

    ::: moniker-end

    更新 `ApplicationDbContext` 以引用自定义 `ApplicationUser` 类：

    ```csharp
    public class ApplicationDbContext
        : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
    ```

    在 `Startup.ConfigureServices`中添加标识服务时注册自定义数据库上下文类：

    ::: moniker range=">= aspnetcore-2.1"

    ```csharp
    services.AddDefaultIdentity<ApplicationUser>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultUI()
            .AddDefaultTokenProviders();
    ```

    通过分析[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象来推断主键的数据类型。

    在 ASP.NET Core 2.1 或更高版本中，标识作为 Razor 类库提供。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用 <xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果使用标识 scaffolder 将标识文件添加到项目中，请删除对 `AddDefaultUI`的调用。

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    ```csharp
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    通过分析[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象来推断主键的数据类型。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    ```csharp
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext, Guid>()
            .AddDefaultTokenProviders();
    ```

    <xref:Microsoft.Extensions.DependencyInjection.IdentityEntityFrameworkBuilderExtensions.AddEntityFrameworkStores*> 方法接受指示主键的数据类型的 `TKey` 类型。

    ::: moniker-end

5. 如果正在使用自定义 `ApplicationRole` 类，请将类更新为从 `IdentityRole<TKey>`继承。 例如：

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationRole.cs?name=snippet_ApplicationRole&highlight=4)]

    更新 `ApplicationDbContext` 以引用自定义 `ApplicationRole` 类。 例如，下面的类引用自定义 `ApplicationUser` 和自定义 `ApplicationRole`：

    ::: moniker range=">= aspnetcore-2.1"

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    在 `Startup.ConfigureServices`中添加标识服务时注册自定义数据库上下文类：

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=13-16)]

    通过分析[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象来推断主键的数据类型。

    在 ASP.NET Core 2.1 或更高版本中，标识作为 Razor 类库提供。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用 <xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果使用标识 scaffolder 将标识文件添加到项目中，请删除对 `AddDefaultUI`的调用。

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    [!code-csharp[](customize-identity-model/samples/2.0/RazorPagesSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    在 `Startup.ConfigureServices`中添加标识服务时注册自定义数据库上下文类：

    [!code-csharp[](customize-identity-model/samples/2.0/RazorPagesSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=7-9)]

    通过分析[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象来推断主键的数据类型。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    在 `Startup.ConfigureServices`中添加标识服务时注册自定义数据库上下文类：

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=7-9)]

    <xref:Microsoft.Extensions.DependencyInjection.IdentityEntityFrameworkBuilderExtensions.AddEntityFrameworkStores*> 方法接受指示主键的数据类型的 `TKey` 类型。

    ::: moniker-end

### <a name="add-navigation-properties"></a>添加导航属性

更改关系的模型配置可能比进行其他更改更难。 必须小心替换现有关系，而不是创建新的其他关系。 特别是，更改的关系必须指定与现有关系相同的外键（FK）属性。 例如，默认情况下，`Users` 和 `UserClaims` 之间的关系按如下方式指定：

```csharp
builder.Entity<TUser>(b =>
{
    // Each User can have many UserClaims
    b.HasMany<TUserClaim>()
     .WithOne()
     .HasForeignKey(uc => uc.UserId)
     .IsRequired();
});
```

此关系的 FK 作为 `UserClaim.UserId` 属性指定。 无需参数即可调用 `HasMany` 和 `WithOne` 来创建不带导航属性的关系。

向 `ApplicationUser` 添加一个导航属性，该属性允许从用户引用关联的 `UserClaims`：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
}
```

`IdentityUserClaim<TKey>` 的 `TKey` 是为用户的 PK 指定的类型。 在这种情况下，`TKey` `string`，因为正在使用默认值。 它**不**是 `UserClaim` 实体类型的 PK 类型。

由于导航属性存在，因此必须在 `OnModelCreating`中进行配置：

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();
        });
    }
}
```

请注意，关系的配置与之前完全相同，只是在对的调用中指定的导航属性 `HasMany`。

导航属性仅存在于 EF 模型中，而不存在于数据库中。 由于关系的 FK 未更改，这种类型的模型更改不需要更新数据库。 这可以通过在更改后添加迁移来进行检查。 `Up` 和 `Down` 方法为空。

### <a name="add-all-user-navigation-properties"></a>添加所有用户导航属性

以下示例使用上述部分作为指导，为用户上的所有关系配置单向导航属性：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
    public virtual ICollection<IdentityUserLogin<string>> Logins { get; set; }
    public virtual ICollection<IdentityUserToken<string>> Tokens { get; set; }
    public virtual ICollection<IdentityUserRole<string>> UserRoles { get; set; }
}
```

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne()
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne()
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne()
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });
    }
}
```

### <a name="add-user-and-role-navigation-properties"></a>添加用户和角色导航属性

以下示例使用上述部分作为指导，为用户和角色上的所有关系配置导航属性：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
    public virtual ICollection<IdentityUserLogin<string>> Logins { get; set; }
    public virtual ICollection<IdentityUserToken<string>> Tokens { get; set; }
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationRole : IdentityRole
{
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationUserRole : IdentityUserRole<string>
{
    public virtual ApplicationUser User { get; set; }
    public virtual ApplicationRole Role { get; set; }
}
```

```csharp
public class ApplicationDbContext
    : IdentityDbContext<
        ApplicationUser, ApplicationRole, string,
        IdentityUserClaim<string>, ApplicationUserRole, IdentityUserLogin<string>,
        IdentityRoleClaim<string>, IdentityUserToken<string>>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne()
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne()
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.User)
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });

        modelBuilder.Entity<ApplicationRole>(b =>
        {
            // Each Role can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.Role)
                .HasForeignKey(ur => ur.RoleId)
                .IsRequired();
        });

    }
}
```

说明：

* 此示例还包括 `UserRole` 联接实体，该实体是从用户到角色导航多对多关系所必需的。
* 请记住更改导航属性的类型，以反映现在正在使用 `ApplicationXxx` 类型，而不是 `IdentityXxx` 类型。
* 请记得使用一般 `ApplicationContext` 定义中的 `ApplicationXxx`。

### <a name="add-all-navigation-properties"></a>添加所有导航属性

以下示例使用上述部分作为指导，为所有实体类型上的所有关系配置导航属性：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<ApplicationUserClaim> Claims { get; set; }
    public virtual ICollection<ApplicationUserLogin> Logins { get; set; }
    public virtual ICollection<ApplicationUserToken> Tokens { get; set; }
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationRole : IdentityRole
{
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
    public virtual ICollection<ApplicationRoleClaim> RoleClaims { get; set; }
}

public class ApplicationUserRole : IdentityUserRole<string>
{
    public virtual ApplicationUser User { get; set; }
    public virtual ApplicationRole Role { get; set; }
}

public class ApplicationUserClaim : IdentityUserClaim<string>
{
    public virtual ApplicationUser User { get; set; }
}

public class ApplicationUserLogin : IdentityUserLogin<string>
{
    public virtual ApplicationUser User { get; set; }
}

public class ApplicationRoleClaim : IdentityRoleClaim<string>
{
    public virtual ApplicationRole Role { get; set; }
}

public class ApplicationUserToken : IdentityUserToken<string>
{
    public virtual ApplicationUser User { get; set; }
}
```

```csharp
public class ApplicationDbContext
    : IdentityDbContext<
        ApplicationUser, ApplicationRole, string,
        ApplicationUserClaim, ApplicationUserRole, ApplicationUserLogin,
        ApplicationRoleClaim, ApplicationUserToken>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne(e => e.User)
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne(e => e.User)
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne(e => e.User)
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.User)
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });

        modelBuilder.Entity<ApplicationRole>(b =>
        {
            // Each Role can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.Role)
                .HasForeignKey(ur => ur.RoleId)
                .IsRequired();

            // Each Role can have many associated RoleClaims
            b.HasMany(e => e.RoleClaims)
                .WithOne(e => e.Role)
                .HasForeignKey(rc => rc.RoleId)
                .IsRequired();
        });
    }
}
```

### <a name="use-composite-keys"></a>使用组合键

前面几节演示了如何更改标识模型中使用的密钥类型。 不支持或建议将标识密钥模型更改为使用组合键。 使用带有标识的组合键涉及更改标识管理器代码与模型交互的方式。 此自定义超出了本文档的范围。

### <a name="change-tablecolumn-names-and-facets"></a>更改表/列名称和方面

若要更改表和列的名称，请调用 `base.OnModelCreating`。 然后，添加配置以覆盖任何默认值。 例如，若要更改所有标识表的名称，请执行以下操作：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.ToTable("MyUsers");
    });

    modelBuilder.Entity<IdentityUserClaim<string>>(b =>
    {
        b.ToTable("MyUserClaims");
    });

    modelBuilder.Entity<IdentityUserLogin<string>>(b =>
    {
        b.ToTable("MyUserLogins");
    });

    modelBuilder.Entity<IdentityUserToken<string>>(b =>
    {
        b.ToTable("MyUserTokens");
    });

    modelBuilder.Entity<IdentityRole>(b =>
    {
        b.ToTable("MyRoles");
    });

    modelBuilder.Entity<IdentityRoleClaim<string>>(b =>
    {
        b.ToTable("MyRoleClaims");
    });

    modelBuilder.Entity<IdentityUserRole<string>>(b =>
    {
        b.ToTable("MyUserRoles");
    });
}
```

这些示例使用默认的标识类型。 如果使用 `ApplicationUser`之类的应用程序类型，请配置该类型而不是默认类型。

下面的示例将更改某些列名：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.Property(e => e.Email).HasColumnName("EMail");
    });

    modelBuilder.Entity<IdentityUserClaim<string>>(b =>
    {
        b.Property(e => e.ClaimType).HasColumnName("CType");
        b.Property(e => e.ClaimValue).HasColumnName("CValue");
    });
}
```

某些类型的数据库列可以配置某些*方面*（例如，允许的最大 `string` 长度）。 下面的示例设置了模型中多个 `string` 属性的列最大长度：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.Property(u => u.UserName).HasMaxLength(128);
        b.Property(u => u.NormalizedUserName).HasMaxLength(128);
        b.Property(u => u.Email).HasMaxLength(128);
        b.Property(u => u.NormalizedEmail).HasMaxLength(128);
    });

    modelBuilder.Entity<IdentityUserToken<string>>(b =>
    {
        b.Property(t => t.LoginProvider).HasMaxLength(128);
        b.Property(t => t.Name).HasMaxLength(128);
    });
}
```

### <a name="map-to-a-different-schema"></a>映射到其他架构

架构在数据库提供程序中的行为可能有所不同。 对于 SQL Server，默认设置是在*dbo*架构中创建所有表。 可在其他架构中创建这些表。 例如：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.HasDefaultSchema("notdbo");
}
```

::: moniker range=">= aspnetcore-2.1"

### <a name="lazy-loading"></a>延迟加载

在本部分中，将添加对标识模型中的延迟加载代理的支持。 延迟加载非常有用，因为它允许使用导航属性，而无需首先确保它们已加载。

可以通过多种方式使实体类型适用于延迟加载，如[EF Core 文档](/ef/core/querying/related-data#lazy-loading)中所述。 为简单起见，请使用延迟加载代理，这需要：

* [Microsoft.entityframeworkcore](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Proxies/)包的安装。
* 调用[AddDbContext\<TContext >](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext)中的 <xref:Microsoft.EntityFrameworkCore.ProxiesExtensions.UseLazyLoadingProxies*>。
* 具有 `public virtual` 导航属性的公共实体类型。

下面的示例演示如何在 `Startup.ConfigureServices`中调用 `UseLazyLoadingProxies`：

```csharp
services
    .AddDbContext<ApplicationDbContext>(
        b => b.UseSqlServer(connectionString)
              .UseLazyLoadingProxies())
    .AddDefaultIdentity<ApplicationUser>()
    .AddEntityFrameworkStores<ApplicationDbContext>();
```

有关将导航属性添加到实体类型的指导，请参阅前面的示例。

## <a name="additional-resources"></a>其他资源

* <xref:security/authentication/scaffold-identity>

::: moniker-end
