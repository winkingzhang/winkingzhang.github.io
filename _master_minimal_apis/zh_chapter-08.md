---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-08
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---


## 添加身份认证和授权

任何类型的应用程序都必须处理**身份认证**和**授权**问题。通常，这些术语会被交替使用，但实际上它们涉及不同的场景。
在本章中，我们将解释身份认证和授权之间的区别，并展示如何将这些功能添加到 Minimal API 项目中。

身份认证可以通过多种不同方式执行：使用本地账户与外部登录提供程序（如 Microsoft、Google、Facebook 和 Twitter）；
使用 Azure Active Directory 和 Azure B2C；以及使用身份认证服务器（如 Identity Server 和 Okta）。
此外，我们可能还需要处理诸如双因素身份认证和刷新令牌等要求。
然而，在本章中，我们将重点关注身份认证和授权的一般方面，并了解如何在 Minimal API 项目中实现它们，以便对该主题有一个总体的理解。
所提供的信息和示例将展示如何有效地处理身份认证和授权，并根据我们的要求自定义其行为。

本章将涵盖以下主题：

* 介绍身份认证和授权
* 保护 Minimal API
* 处理授权 - 角色和策略

## 技术要求

要遵循本章中的示例，你需要创建一个 ASP.NET Core 6.0 Web API 应用程序。
请参考 [第 2 章 “探索最小 API 及其优势”](/books/master-minimal-apis/chapter-02) 中的技术要求部分，了解如何创建。

如果你使用控制台、shell 或 Bash 终端创建 API，请记住将工作目录更改为当前章节编号（Chapter08）。

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，地址为：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter08](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter08)

## 介绍身份认证和授权

如前所述，身份认证和授权这两个术语经常被交替使用，但它们代表不同的安全功能。
身份认证是认证用户是否是他们声称的人的过程，而授权是授予已认证用户执行某些操作的权限的任务。
因此，授权必须始终在身份认证之后进行。

让我们以机场的安全为例：首先，你出示身份证以认证你的身份；
然后，在登机口，你出示登机牌以获得登机和进入飞机的授权。

在 ASP.NET Core 中，身份认证和授权由相应的中间件处理，并且在 Minimal API 和基于控制器的项目中以相同的方式工作。
它们允许根据用户身份、角色、策略等限制对端点的访问，我们将在接下来的部分中详细介绍。

你可以在官方文档中找到关于 ASP.NET Core 身份认证和授权的精彩概述，地址分别为
[https://docs.microsoft.com/aspnet/core/security/authentication](https://docs.microsoft.com/aspnet/core/security/authentication) 和
[https://docs.microsoft.com/aspnet/core/security/authorization](https://docs.microsoft.com/aspnet/core/security/authorization)。

保护 Minimal API

保护 Minimal API 意味着正确设置身份认证和授权。在现代应用程序中，采用了多种类型的身份认证解决方案。
在 Web 应用程序中，我们通常使用 `cookies`，而在处理 Web API 时，
我们使用诸如 API 密钥、基本身份认证和 **JSON Web Token（JWT）** 等方法。
JWT 是最常用的，在本章的其余部分，我们将重点关注此解决方案。

> 注意：
> 
> 了解 JWT 是什么以及如何使用它们的一个好的起点可在 [https://jwt.io/introduction](https://jwt.io/introduction) 找到。

要启用基于 JWT 的身份认证和授权，首先要做的是使用以下方法之一
将 `Microsoft.AspNetCore.Authentication.JwtBearer` NuGet 包添加到我们的项目中：

* 选项 1：如果你使用 Visual Studio 2022，右键单击项目并选择 “**管理 NuGet 包**” 命令以打开包管理器 GUI，
然后搜索 `Microsoft.AspNetCore.Authentication.JwtBearer` 并点击 “安装”。
* 选项 2：打开包管理器控制台（如果你在 Visual Studio 2022 内），或者打开你的控制台、shell 或 Bash 终端，
转到你的项目目录，并执行以下命令：<br/>
`dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer`

现在，我们需要将身份认证和授权服务添加到服务提供程序中，以便通过依赖注入使用它们：
```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddJwtBearer();
builder.Services.AddAuthorization();
```

这是为 ASP.NET Core 项目添加 JWT 身份认证和授权支持所需的最少代码。
它还不是一个真正可行的解决方案，因为它缺少实际的配置，但它足以认证端点保护的工作原理。

在 `AddAuthentication ()` 方法中，我们指定要使用承载身份认证方案。
这是一种 HTTP 身份认证方案，涉及安全令牌，实际上称为承载令牌。
这些令牌必须在 `Authorization` HTTP 头中以 `Authorization: Bearer <token>` 的格式发送。
然后，我们调用 `AddJwtBearer ()` 告诉 ASP.NET Core 它必须期望 JWT 格式的承载令牌。
正如我们稍后将看到的，承载令牌是服务器响应登录请求生成的编码字符串。
之后，我们使用 `AddAuthorization ()` 也添加授权服务。

现在，我们需要将身份认证和授权中间件插入到管道中，以便 ASP.NET Core 被指示检查令牌并应用所有授权规则：

```csharp
var app = builder.Build();
//...
app.UseAuthentication();
app.UseAuthorization();
//...
app.Run();
```

> 重要注意事项：
> 
> 我们已经说过授权必须在身份认证之后进行。这意味着身份认证中间件必须首先出现；否则，安全性将无法按预期工作。

最后，我们可以使用 Authorize 属性或 `RequireAuthorization ()` 方法保护我们的端点：

```csharp
app.MapGet("/api/attribute-protected", [Authorize] () =>
    "This endpoint is protected by attribute.")
app.MapGet("/api/method-protected", () => 
    "This endpoint is protected by method.")
    .RequireAuthorization();
```

> 注意：
> 
> 能够直接在 lambda 表达式上指定属性（如前面示例中的第一个端点）是 **C# 10** 的一个新功能。

如果我们现在尝试使用 Swagger 调用这些方法中的每一个，我们将得到一个 `401 Unauthorized` 的响应，如下所示：

![Figure 8.1 – Unauthorized response in Swagger](/assets/images/master-minimal-apis/Figure_8.1_B17902.jpg)

注意消息中包含一个头，指示预期的身份认证方案是 `Bearer`，正如我们在代码中声明的那样。

所以，现在我们知道如何限制对已认证用户的端点访问。但我们的工作还没有完成：
我们需要生成一个 JWT 承载令牌，认证它，并找到一种方法将这样的令牌传递给 Swagger，以便我们可以测试我们受保护的端点。

### 生成 JWT 承载令牌

我们已经说过，JWT 承载令牌是服务器响应登录请求生成的。
ASP.NET Core 提供了我们创建它所需的所有 API，所以让我们看看如何执行此任务。

首先要做的是定义登录请求端点，以使用用户名和密码对用户进行身份认证：

```csharp
app.MapPost("/api/auth/login", (LoginRequest request) =>
{
    if (request.Username == "marco" && request.Password == "P@$$w0rd")
    {
        // 生成 JWT 承载令牌...
    }
    return Results.BadRequest();
});
```

为了简单起见，在前面的示例中，我们使用了硬编码的值，但在实际应用程序中，
我们将使用例如 **ASP.NET Core Identity**，这是 ASP.NET Core 中负责用户管理的部分。
有关此主题的更多信息可在官方文档中找到，地址为：
[https://docs.microsoft.com/aspnet/core/security/authentication/identity](https://docs.microsoft.com/aspnet/core/security/authentication/identity)

在典型的登录工作流程中，如果凭据无效，我们将向客户端返回 `400 Bad Request` 响应。
如果用户名和密码正确，我们可以有效地生成一个 JWT 承载令牌，使用 ASP.NET Core 中的类：

```csharp
var claims = new List<Claim>()
{
    new(ClaimTypes.Name, request.Username)
};
var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("mysecuritystring"));
var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);
var jwtSecurityToken = new JwtSecurityToken(
    issuer: "https://www.packtpub.com",
    audience: "Minimal APIs Client",
    claims: claims,
    expires: DateTime.UtcNow.AddHours(1),
    signingCredentials: credentials);
var accessToken = new JwtSecurityTokenHandler().WriteToken(jwtSecurityToken);
return Results.Ok(new { AccessToken = accessToken });
```

JWT 承载令牌的创建涉及许多不同的概念，但通过前面的代码示例，我们将重点关注基本概念。
这种承载令牌包含允许认证用户身份的信息，以及描述用户属性的其他声明。
这些属性称为 **声明(claims)**，表示为字符串键值对。
在前面的代码中，我们创建了一个包含单个声明的列表，其中包含用户名。
我们可以根据需要添加尽可能多的声明，并且也可以有相同名称的声明。
在接下来的部分中，我们将看到如何使用声明，例如执行授权。

接下来，在前面的代码中，我们定义了签名凭证（`SigningCredentials`）来签署 JWT 承载令牌。
签名取决于实际的令牌内容，并用于检查令牌是否未被篡改。
实际上，如果我们更改令牌中的任何内容，例如声明值，签名将相应地改变。
由于签署承载令牌的密钥仅由服务器知道，第三方不可能修改令牌并保持其有效性。
在前面的代码中，我们使用了 `SymmetricSecurityKey`，它从不与客户端共享。

我们使用一个短字符串创建了凭证，但唯一的要求是密钥至少应为 32 字节或 16 个字符长。
在.NET 中，字符串是 Unicode，因此每个字符占用 2 字节。我们还需要设置凭证将用于签署令牌的算法。
为此，我们指定了 **基于哈希的消息认证码（HMAC）** 和哈希函数 `SHA256`，指定了 `SecurityAlgorithms.HmacSha256` 值。
这种算法在这种情况下是一个相当常见的选择。

> 注意：
> 
> 你可以在 
> [https://docs.microsoft.com/dotnet/api/system.security.cryptography.hmacsha256#remarks](https://docs.microsoft.com/dotnet/api/system.security.cryptography.hmacsha256#remarks) 
> 找到关于 HMAC 和 SHA256 哈希函数的更多信息。

到目前为止，在前面的代码中，我们终于拥有了创建令牌所需的所有信息，所以我们可以实例化一个 `JwtSecurityToken` 对象。
这个类可以使用许多参数来构建令牌，但为了简单起见，我们只指定了一个工作示例所需的最少参数：

* Issuer：一个字符串（通常是一个 URI），标识创建令牌的实体的名称。
* Audience：JWT 旨在针对的接收者，即谁可以使用令牌。
* 声明列表。
* 令牌的过期时间（以 UTC 表示）。
* 签名凭证。

> 提示：
> 
> 在前面的代码示例中，用于构建令牌的值是硬编码的，但在实际应用程序中，
> 我们应该将它们放在一个外部源中，例如 `appsettings.json` 配置文件中。

你可以在 
[https://docs.microsoft.com/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytoken](https://docs.microsoft.com/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytoken)
找到关于创建令牌的更多信息。

在完成前面的所有步骤后，我们可以创建 `JwtSecurityTokenHandler`，它负责实际生成承载令牌并将其返回给调用者，响应为 `200 OK`。

所以，现在我们可以尝试在 Swagger 中使用登录端点。
在插入正确的用户名和密码并点击 “**Execute**” 按钮后，我们将得到以下响应：

![Figure 8.2 – The JWT bearer as a result of the login request in Swagger](/assets/images/master-minimal-apis/Figure_8.2_B17902.jpg)

我们可以复制令牌值并将其插入到 [https://jwt.ms](https://jwt.ms) 网站的 URL 中，以查看它包含的内容。
我们将得到类似以下的内容：

```
{
  "alg": "HS256",
  "typ": "JWT"
}.{
  "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name": "marco",
  "exp": 1644431527,
  "iss": "https://www.packtpub.com",
  "aud": "Minimal APIs Client"
}.[Signature]
```

特别是，我们看到了已配置的声明：
* name：已登录用户的名称。
* exp：令牌的过期时间，表示为 Unix 纪元。
* iss：令牌的发行者。
* aud：令牌的接收者（受众）。

这是原始视图，但我们可以切换到 “Claims” 选项卡以查看所有声明的解码列表，并在可用时查看其含义的描述。

有一个重要的点需要注意：默认情况下，JWT 承载令牌未加密（它只是一个 Base64 编码的字符串），所以任何人都可以读取其内容。
令牌的安全性不取决于其无法被解码，而在于它已被签名。
即使令牌的内容是清晰的，也不可能修改它，因为在这种情况下，签名（使用仅服务器知道的密钥）将变得无效。

所以，重要的是不要在令牌中插入敏感数据；声明如用户名、用户 ID 和角色通常是可以的，但例如，我们不应该插入与隐私相关的信息。
举一个故意夸张的例子，我们绝不能在令牌中插入信用卡号码！
无论如何，请记住，即使 Microsoft 对于 Azure Active Directory 也使用 JWT，没有加密，所以我们可以信任这个安全系统。

总之，我们已经描述了如何获得一个有效的 JWT。
接下来的步骤是将令牌传递给我们受保护的端点，并指示我们的 Minimal API 如何认证它。

### 认证 JWT 承载令牌

在创建 JWT 承载令牌后，我们需要在每个 HTTP 请求中，在 `Authorization` HTTP 头内传递它，
以便 ASP.NET Core 可以认证其有效性并允许我们调用受保护的端点。
所以，我们必须完成前面描述的 `AddJwtBearer ()` 方法的调用，并指定认证承载令牌的规则：

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("mysecuritystring")),
            ValidIssuer = "https://www.packtpub.com",
            ValidAudience = "Minimal APIs Client"
        };
    });
```

在前面的代码中，我们添加了一个 `lambda` 表达式，其中我们定义了 `TokenValidationParameter` 对象，该对象包含令牌认证规则。
首先，我们检查发行者签名密钥，即令牌的签名，如在 “生成 JWT 承载令牌” 部分中所示，以认证 JWT 未被篡改。
用于签署令牌的安全字符串是执行此检查所需的，所以我们指定与我们在登录请求中插入的相同值（`mysecuritystring`）。
然后，我们指定令牌的有效发行者和受众的值。如果令牌是由不同的发行者发出的，或者是针对不同的受众的，则认证失败。

这是一个重要的安全检查；我们应该确保承载令牌是由我们期望的人发行的，并且是针对我们想要的受众的。

> 提示：
> 
> 如前所述，我们应该将用于处理令牌的信息放在一个外部源中，
> 以便我们可以在令牌生成和认证期间引用正确的值，避免硬编码它们或写入它们的值两次。

我们不需要指定我们也希望认证令牌过期，因为此检查是自动启用的。
在认证时间时应用一个时钟偏差，以补偿时钟时间的微小差异或处理客户端请求与服务器处理它的瞬间之间的延迟。
默认值是 5 分钟，这意味着过期的令牌在其实际过期后的 5 分钟时间范围内被视为有效。
我们可以减少时钟偏差，或禁用它，使用 `TokenValidationParameter` 类的 `ClockSkew` 属性。

现在，Minimal API 拥有检查承载令牌有效性所需的所有信息。
为了测试一切是否按预期工作，我们需要一种方法来告诉 Swagger 如何在请求中发送令牌，如下一节所示。

### 添加 JWT 支持到 Swagger

我们已经说过，承载令牌在请求的 `Authorization` HTTP 头中发送。
如果我们想使用 Swagger 认证身份认证系统并测试我们受保护的端点，我们需要更新配置，以便它能够在请求中包含此头。

要执行此任务，需要在 `AddSwaggerGen ()` 方法中添加一些代码：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddSwaggerGen(options =>
{
    options.AddSecurityDefinition(JwtBearerDefaults.AuthenticationScheme, 
        new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.ApiKey,
            In = ParameterLocation.Header,
            Name = HeaderNames.Authorization,
            Description = "Insert the token with the 'Bearer' prefix"
        });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = JwtBearerDefaults.AuthenticationScheme
                }
            },
            Array.Empty<string>()
        }
    });
});
```

在前面的代码中，我们定义了 Swagger 如何处理身份认证。
使用 `AddSecurityDefinition ()` 方法，我们描述了我们的 API 是如何受保护的；
我们使用一个 API 键，即承载令牌，在名为 `Authorization` 的头中。
然后，使用 `AddSecurityRequirement ()`，我们指定我们的端点有一个安全要求，这意味着必须为每个请求发送安全信息。

在添加前面的代码后，如果我们现在运行我们的应用程序，Swagger UI 将包含一些新内容。

![Figure 8.3 – Swagger showing the authentication features](/assets/images/master-minimal-apis/Figure_8.3_B17902.jpg)

点击 “**Authorize**” 按钮或端点右侧的任何挂锁图标，将出现以下窗口，允许我们插入承载令牌：

![Figure 8.4 – The window that allows setting the bearer token](/assets/images/master-minimal-apis/Figure_8.4_B17902.jpg)

最后要做的是在 “**Value**” 文本框中插入令牌并点击 “**Authorize**” 确认。
从现在开始，指定的承载令牌将与使用 Swagger 发出的每个请求一起发送。

我们终于完成了向 Minimal API 添加身份认证支持所需的所有步骤。
现在，是时候认证一切是否按预期工作了。
在下一节中，我们将进行一些测试。

### 测试身份认证

如前所述，如果我们调用一个受保护的端点，我们会得到一个 `401 Unauthorized` 的响应。
为了认证令牌身份认证是否工作，让我们调用登录端点以获取一个令牌。
之后，点击 Swagger 中的 “**Authorize**” 按钮并插入获得的令牌，记住 `Bearer<space>` 前缀。
现在，我们将得到一个 `200 OK` 响应，这意味着我们能够正确调用需要身份认证的端点。
我们也可以尝试更改令牌中的一个字符，再次得到 `401 Unauthorized` 的响应，
因为在这种情况下，签名将不是预期的，如前所述。同样，如果令牌形式上有效但已过期，我们将得到一个 **401** 响应。

由于我们已经定义了只有经过身份认证的用户才能访问的端点，一个常见的需求是在相应的路由处理程序中访问用户信息。
在 [第 2 章 “探索 Minimal API 及其优势”](/books/master-minimal-apis/chapter-02) 中，
我们展示了 Minimal API 提供了一个特殊绑定，直接提供一个 `ClaimsPrincipal` 对象来表示已登录用户：

```csharp
app.MapGet("/api/me", [Authorize] (ClaimsPrincipal user) =>
{
    // 这里可以获取用户信息并返回，例如返回用户名
    return $"User: {user.Identity.Name}"; 
});
```

路由处理程序的用户参数会自动填充用户信息。
在这个示例中，我们只是获取用户名，而用户名又是从令牌声明中读取的，但该对象公开了许多属性，允许我们处理身份认证数据。
你可以参考官方文档 
[https://docs.microsoft.com/dotnet/api/system.security.claims.claimsprincipal.identity](https://docs.microsoft.com/dotnet/api/system.security.claims.claimsprincipal.identity)
获取更多详细信息。

这就结束了我们对身份认证的概述。在下一节中，我们将看看如何处理授权。

## 处理授权 - 角色和策略

在身份认证之后，紧接着就是授权步骤，它授予已认证用户执行某些操作的权限。
Minimal API 提供了与基于控制器的项目相同的授权功能，基于角色和策略的概念。

当创建一个身份时，它可能属于一个或多个角色。
例如，一个用户可以属于管理员角色，而另一个用户可以同时属于用户和利益相关者两个角色。

通常，每个用户只能执行其角色允许的操作。角色只是在身份认证时插入到 JWT 承载令牌中的声明。
正如我们马上会看到的，ASP.NET Core 提供了内置支持来认证用户是否属于某个角色。

虽然基于角色的授权涵盖了许多场景，但在某些情况下，这种安全方式是不够的，因为我们需要应用更具体的规则来检查用户是否有权执行某些活动。
在这种情况下，我们可以创建自定义策略，允许我们指定更详细的授权要求，甚至完全根据我们的算法定义授权逻辑。

在接下来的部分中，我们将看到如何在我们的 API 中管理基于角色和基于策略的授权，
以便我们能够涵盖所有我们的要求，即允许只有具有特定角色或声明的用户访问某些端点，或者基于我们的自定义逻辑。

### 处理基于角色的授权

如前所述，角色是声明。这意味着它们必须像其他任何声明一样在身份认证时插入到 JWT 承载令牌中：

```csharp
app.MapPost("/api/auth/login", (LoginRequest request) =>
{
    if (request.Username == "marco" && request.Password == "P@$$w0rd")
    {
        var claims = new List<Claim>()
        {
            new(ClaimTypes.Name, request.Username),
            new(ClaimTypes.Role, "Administrator"),
            new(ClaimTypes.Role, "User")
        };
        //...
    }
})
```

在这个示例中，我们静态地添加了两个名为 `ClaimTypes.Role` 的声明：`Administrator` 和 `User`。
如前所述，在实际应用中，这些值通常来自一个完整的用户管理系统，例如使用 ASP.NET Core Identity 构建的系统。

与所有其他声明一样，角色被插入到 JWT 承载令牌中。
如果现在我们尝试调用登录端点，我们会注意到令牌更长了，因为它包含了很多信息，
我们可以再次使用 [https://jwt.ms](https://jwt.ms) 网站来认证这一点：

```
{
    "alg": "HS256",
    "typ": "JWT"
}.{
    "http://schemas.xmlsoap.org/ws/2005/05/identity/cla
    "http://schemas.microsoft.com/ws/2008/06/identity/c
    "Administrator",
    "User"
],
    "exp": 1644755166,
    "iss": "https://www.packtpub.com",
    "aud": "Minimal APIs Client"
}.[Signature]
```

为了限制只有属于特定角色的用户才能访问某个端点，
我们需要将这个角色作为参数指定在 `Authorize` 属性或 `RequireAuthorization ()` 方法中：

```csharp
app.MapGet("/api/admin-attribute-protected", [Authorize(Roles = "Administrator")] () => { })
app.MapGet("/api/admin-method-protected", () => { })
    .RequireAuthorization(new AuthorizeAttribute { Roles = "Administrator" });
```

通过这种方式，只有被分配了 `Administrator` 角色的用户才能访问这些端点。
我们也可以指定多个角色，用逗号分隔：用户只要拥有至少一个指定的角色就会被授权。

> 重要注意事项：
> 
> 角色名称是区分大小写的。

现在假设我们有以下端点：

```csharp
app.MapGet("/api/stackeholder-protected", [Authorize(Roles = "Stakeholder")] () => { })
```

这个方法只能由被分配了 `Stakeholder` 角色的用户调用。 然而，在我们的示例中，这个角色并没有被分配。
所以，如果我们使用之前的承载令牌并尝试调用这个端点，当然会得到一个错误。
但在这种情况下，它不会是 `401 Unauthorized`，而是 `403 Forbidden`。
我们看到这种行为是因为用户实际上已经通过身份认证（意味着令牌是有效的，所以不是 **401** 错误），
但他们没有执行该方法的授权，所以访问被禁止。换句话说，身份认证错误和授权错误会导致不同的 HTTP 状态码。

还有另一个重要的场景涉及角色。有时，我们根本不需要限制端点访问，而是需要根据特定用户角色调整处理程序的行为，
例如在检索仅特定类型的信息时。在这种情况下，我们可以使用 `ClaimsPrincipal` 对象上的 `IsInRole ()` 方法：

```csharp
app.MapGet("/api/role-check", [Authorize] (ClaimsPrincipal user) =>
{
    if (user.IsInRole("Administrator"))
    {
        return "User is an Administrator";
    }
    return "This is a normal user";
});
```

在这个端点中，我们只使用 `Authorize` 属性来检查用户是否已通过身份认证。
然后，在路由处理程序中，我们检查用户是否具有 `Administrator` 角色。
如果是，我们只返回一个消息，但我们可以想象管理员可以检索所有可用信息，而普通用户只能根据信息本身的值获取一个子集。

正如我们所看到的，通过基于角色的授权，我们可以在我们的端点中执行不同类型的授权检查，以涵盖许多场景。
然而，这种方法不能处理所有情况。如果角色不够用，我们需要使用基于策略的授权，我们将在下一节中讨论。

### 应用基于策略的授权

策略是一种更通用的定义授权规则的方式。
基于角色的授权可以被视为一种涉及角色检查的特定策略授权。
我们通常在需要处理更复杂场景时使用策略。

这种类型的授权需要两个步骤：
1. 定义一个带有规则集的策略
2. 将某个策略应用到端点

策略是在 `AddAuthorization ()` 方法的上下文中添加的，我们在前面的 “保护 Minimal API” 部分中看到了这个方法。
每个策略都有一个唯一的名称，用于稍后引用它，以及一组规则，通常以流畅的方式描述。

我们可以在角色授权不足时使用策略。假设承载令牌还包含用户所属租户的 ID：

```csharp
var claims = new List<Claim>()
{
    //...
    new("tenant-id", "42")
};
```

同样，在实际场景中，这个值可能来自一个存储用户属性的数据库。假设我们只想允许属于特定租户的用户访问某个端点。
由于 `tenant-id` 是一个自定义声明，ASP.NET Core 不知道如何使用它来执行授权。
所以，我们不能使用前面展示的解决方案。我们需要定义一个带有相应规则的自定义策略：

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Tenant42", policy =>
    {
        policy.RequireClaim("tenant-id", "42");
    });
});
```

在前面的代码中，我们创建了一个名为 `Tenant42` 的策略，它要求令牌包含 `tenant-id` 声明且值为 `42`。
策略变量是 `AuthorizationPolicyBuilder` 的一个实例，它公开了一些方法，允许我们流畅地指定授权规则；
我们可以指定一个策略要求某些用户、角色和声明得到满足。
我们也可以在同一个策略中链接多个要求，例如写 `policy.RequireRole (“Administrator”).RequireClaim (“tenant-id”)`。
完整的方法列表可以在文档页面 
[https://docs.microsoft.com/dotnet/api/microsoft.aspnetcore.authorization.authorizationpolicybuilder](https://docs.microsoft.com/dotnet/api/microsoft.aspnetcore.authorization.authorizationpolicybuilder) 
上找到。

然后，在我们想要保护的方法中，我们必须像往常一样使用 `Authorize` 属性或 `RequireAuthorization ()` 方法指定策略名称：

```csharp
app.MapGet("/api/policy-attribute-protected", [Authorize(Policy = "Tenant42")] () => { })
app.MapGet("/api/policy-method-protected", () => { })
    .RequireAuthorization("Tenant42");
```

如果我们尝试使用不包含 `tenant-id` 声明或其值不是 `42` 的令牌执行这些端点，
我们会得到一个 `403 Forbidden` 结果，就像在角色检查中发生的那样。

有些场景中，声明一个允许的角色和声明列表是不够的：例如，我们可能需要执行更复杂的检查或根据动态参数认证授权。
在这些情况下，我们可以使用所谓的策略要求，它包括一组授权规则，我们可以为其提供自定义认证逻辑。

要采用这种解决方案，我们需要两个对象：
1. 一个要求类，它实现 `IAuthorizationRequirement` 接口并定义我们想要管理的要求
2. 一个处理程序类，它继承自 `AuthorizationHandler` 并包含认证要求的逻辑

假设我们不想让不属于 `Administrator` 角色的用户在维护时间窗口内访问某些端点。
这是一个完全有效的授权规则，但我们不能使用到目前为止看到的解决方案来实现它。
这个规则涉及一个考虑当前时间的条件，所以策略不能是静态定义的。

所以，我们首先创建一个自定义要求：

```csharp
public class MaintenanceTimeRequirement : IAuthorizationRequirement
{
    public TimeOnly StartTime { get; init; }
    public TimeOnly EndTime { get; init; }
}
```

这个要求包含维护窗口的开始和结束时间。在这个时间段内，我们只想让管理员能够操作。

> 注意：
> 
> `TimeOnly` 是 **C# 10** 中引入的一种新的数据类型，它允许我们只存储一天中的时间（而不是日期）。
> 更多信息可以在 [https://docs.microsoft.com/dotnet/api/system.timeonly](https://docs.microsoft.com/dotnet/api/system.timeonly) 找到。

注意 `IAuthorizationRequirement` 接口只是一个占位符。
它不包含任何要实现的方法或属性；它只是用来识别这个类是一个要求。
换句话说，如果我们不需要为要求提供任何额外信息，我们可以创建一个实现 `IAuthorizationRequirement` 但实际上没有内容的类。

这个要求必须得到执行，所以我们需要创建相应的处理程序：

```csharp
public class MaintenanceTimeAuthorizationHandler 
    : AuthorizationHandler<MaintenanceTimeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        MaintenanceTimeRequirement requirement)
    {
        var isAuthorized = true;
        if (!context.User.IsInRole("Administrator"))
        {
            var time = TimeOnly.FromDateTime(DateTime.Now);
            if (time >= requirement.StartTime 
                && time <= requirement.EndTime)
            {
                isAuthorized = false;
            }
        }
        if (isAuthorized)
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}
```

我们的处理程序继承自 `AuthorizationHandler<MaintenanceTimeRequirement>`，
所以我们需要重写 `HandleRequirementAsync ()` 方法来认证要求，
使用 `AuthorizationHandlerContext` 参数，它包含对当前用户的引用。
如开头所述，如果用户没有被分配 `Administrator` 角色，我们检查当前时间是否在维护窗口内。
如果是，用户没有访问权限。

最后，如果 `isAuthorized` 变量为 `true`，这意味着授权可以被授予，
所以我们在 `context` 对象上调用 `Succeed ()` 方法，传递我们想要认证的要求。
否则，我们不在 `context` 上调用任何方法，这意味着要求没有得到认证。

我们还没有完成实现自定义策略。我们仍然需要定义策略并在服务提供器中注册处理程序：

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("TimedAccessPolicy", policy =>
    {
        policy.Requirements.Add(new MaintenanceTimeRequirement
        {
            StartTime = new TimeOnly(0, 0, 0),
            EndTime = new TimeOnly(4, 0, 0)
        });
    });
});
builder.Services.AddScoped<IAuthorizationHandler, 
    MaintenanceTimeAuthorizationHandler>();
```

在前面的代码中，我们定义了一个从午夜到凌晨 4 点的维护时间窗口。
然后，我们将处理程序注册为 `IAuthorizationHandler` 接口的一个实现，
而 `IAuthorizationHandler` 接口又由 `AuthorizationHandler` 类实现。

现在我们已经一切就绪，我们可以将策略应用到我们的端点：

```csharp
app.MapGet("/api/custom-policy-protected", 
    [Authorize(Policy = "TimedAccessPolicy")] () => { })
```

当我们尝试访问这个端点时，ASP.NET Core 会检查相应的策略，发现它包含一个要求，
并扫描所有 `IAuhorizationHandler` 接口的注册，看看是否有一个能够处理这个要求。
然后，处理程序将被调用，结果将用于确定用户是否有访问路由的权利。
如果策略没有得到认证，我们会得到一个 `403 Forbidden` 响应。

我们已经展示了策略是多么强大，但还有更多。
我们还可以使用它们来定义自动应用于所有端点的全局规则，使用默认和回退策略的概念，我们将在下一节中看到。

### 使用默认和回退策略

默认和回退策略在我们想要定义必须自动应用的全局规则时很有用。
实际上，当我们使用 `Authorize` 属性或 `RequireAuthorization ()` 方法而没有任何其他参数时，
我们隐式地引用了 ASP.NET Core 定义的默认策略，该策略设置为要求一个已认证的用户。

如果我们想要使用不同的默认条件，我们只需要重新定义 `DefaultPolicy` 属性，它在 `AddAuthorization ()` 方法的上下文中可用：

```csharp
builder.Services.AddAuthorization(options =>
{
    var policy = new AuthorizationPolicyBuilder()
     .RequireAuthenticatedUser()
     .RequireClaim("tenant-id")
     .Build();
    options.DefaultPolicy = policy; 
});
```

我们使用 `AuthorizationPolicyBuilder` 定义所有的安全要求，然后将其设置为默认策略。
这样，即使我们在 `Authorize` 属性或 `RequireAuthorization ()` 方法中没有指定自定义策略，
系统也会总是认证用户是否已认证，并且承载令牌是否包含 `tenant-id` 声明。
当然，我们可以通过在授权属性或方法中指定角色或策略名称来覆盖这个默认行为。

另一方面，回退策略是在端点没有授权信息时应用的策略。它在我们想要自动保护所有我们的端点时很有用，
即使我们忘记指定 `Authorize` 属性或者只是不想为每个处理程序重复这个属性。
让我们试着通过以下代码来理解这一点：

```csharp
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = options.DefaultPolicy;
});
```

在前面的代码中，`FallbackPolicy` 变得等于 `DefaultPolicy`。
我们已经说过默认策略要求用户已认证，所以这个代码的结果是现在所有的端点都自动需要身份认证，即使我们没有显式地保护它们。

这是一个在我们的大多数端点都有受限访问时通常采用的解决方案。
我们不再需要指定 `Authorize` 属性或使用 `RequireAuthorization ()` 方法。
换句话说，现在我们所有的端点都默认受到保护。

如果我们决定使用这种方法，但有一些端点需要公共访问，比如登录端点，每个人都应该能够调用它，
我们可以使用 `AllowAnonymous` 属性或 `AllowAnonymous ()` 方法：

```csharp
app.MapPost("/api/auth/login", [AllowAnonymous] (LoginRequest request) =>
{
    //...
})
// 或者
app.MapPost("/api/auth/login", (LoginRequest request) =>
{
    request.AllowAnonymous();
})
```

正如其名称所示，前面的代码将绕过端点的所有授权检查，包括默认和回退授权策略。
要深入了解基于策略的身份认证，我们可以参考官方文档：<br/>
[https://docs.microsoft.com/aspnet/core/security/authorization/policies](https://docs.microsoft.com/aspnet/core/security/authorization/policies)

## 总结

了解身份认证和授权在 Minimal API 中的工作方式对于开发安全应用程序至关重要。
使用 JWT 承载身份认证、角色和策略，我们甚至可以定义复杂的授权场景，能够使用标准和自定义规则。

在本章中，我们介绍了使服务安全的基本概念，但还有更多内容需要讨论，特别是关于 ASP.NET Core Identity：
一个支持登录功能并允许管理用户、密码、个人资料数据、角色、声明等的 API。
我们可以通过查看官方文档进一步研究这个主题，
官方文档可在 [https://docs.microsoft.com/aspnet/core/security/authentication/identity](https://docs.microsoft.com/aspnet/core/security/authentication/identity) 找到。

在下一章中，我们将看到如何为我们的 Minimal API 添加多语言支持，以及如何正确处理与不同日期格式、时区等一起工作的应用程序。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
