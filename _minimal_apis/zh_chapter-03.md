---
title: "使用 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/minimal-apis/chapter-03
layout: single
classes: wide
sidebar:
  nav: "minimal_apis"
---


## 使用 Minimal APIs

在本章中，我们将尝试应用在早期版本的 .NET 中可用的一些高级开发技术。
我们将涉及四个相互独立的常见主题，涵盖生产力主题以及前端接口和配置管理的最佳实践。

每个开发人员迟早都会遇到我们在本章中描述的问题。
程序员需要为 API 编写文档，使 API 与 JavaScript 前端进行通信，处理错误并尝试修复它们，以及根据参数配置应用程序。

本章将涉及的主题如下：
- 探索 Swagger
- 支持 CORS
- 使用全局 API 设置
- 错误处理


## 技术要求

如前几章所述，需要有可用的 .NET 6 开发框架；还需要使用 .NET 工具来运行内存中的 Web 服务器。

为了验证 **跨源资源共享 (CORS)** 的功能，我们还需要有一个与我们托管 API 的地址不同的 HTTP 地址上的前端应用程序。

为了测试本章中提到的 CORS 示例，我们将利用内存中的 Web 服务器，它将允许我们托管一个简单的静态 HTML 页面。

为此，我们将使用 `LiveReloadServer` 来托管网页（HTML 和 JavaScript），您可以使用以下命令将其安装为 .NET 工具：
```bash
$ dotnet tool install -g LiveReloadServer
```

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，网址为：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6/tree/main/Chapter03](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6/tree/main/Chapter03).

## 探索 Swagger

Swagger 在很大程度上进入了 .NET 开发人员的生活； 它一直存在于多个版本的 Visual Studio 的项目架上。

Swagger 是一个基于 OpenAPI 规范的工具，允许你使用 Web 应用程序为 API 编写文档。根据官方文档
[https://oai.github.io/Documentation/introduction.xhtml](https://oai.github.io/Documentation/introduction.xhtml)：

> OpenAPI 规范允许描述通过 HTTP 或类似 HTTP 的协议可访问的远程 API。
> 
> API 定义了两个软件之间允许的交互，就像用户界面定义了用户与程序交互的方式一样。
> 
> API 由可能调用的方法（发出的请求）、它们的参数、返回值以及它们所需的任何数据格式（以及其他内容）组成。
> 这类似于用户与移动应用程序的交互仅限于应用程序用户界面中的按钮、滑块和文本框。 


### Visual Studio 项目模板中的 Swagger

我们了解到，在 .NET 世界中，我们所熟知的 Swagger 只不过是为所有公开基于 Web 的 API 的应用程序定义的一组规范：

![Figure_3.1 - Visual Studio scaffold](/assets/images/minimal-apis/Figure_3.1_B17902.jpg)

通过选择 " _启用 OpenAPI 支持_ "，Visual Studio 会添加一个名为 `Swashbuckle.AspNetCore` 的 NuGet 包，
并自动在 **Program.cs** 文件中对其进行配置。

我们展示了在新项目中添加的几行代码。通过这些少量信息，仅在开发环境中启用了一个 Web 应用程序，
这允许开发人员在不生成客户端或使用应用程序外部工具的情况下测试 API：

```csharp
var builder = WebApplication.CreateBuilder(args);

// below two lines setup Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    // below two lines enable Swagger
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Swagger 生成的可视化部分大大提高了生产力，并允许开发人员与将与应用程序交互的人员（无论是前端应用程序还是机器应用程序）共享信息。

> 注意
> 
> 再次提醒，强烈建议不要在生产环境中启用 Swagger，因为敏感信息可能会在网络或应用程序所在的网络上公开。

我们已经了解了如何将 Swagger 引入我们的 API 应用程序；此功能允许我们为 API 编写文档，并允许用户生成调用我们应用程序的客户端。
让我们看看有哪些选项可以快速将应用程序与使用 OpenAPI 描述的 API 进行接口。

### OpenAPI 生成器

使用 Swagger，尤其是使用 OpenAPI 标准，您可以自动生成客户端以连接到 Web 应用程序。 
可以为多种语言生成客户端，也可以为开发工具生成客户端。 我们知道编写客户端访问 Web API 是多么繁琐和重复。 
Open API Generator 帮助我们自动生成代码，检查 Swagger 和 OpenAPI 制作的 API 文档，并自动生成与 API 接口的代码。 
简单、容易，最重要的是，快速。

npm 包 `@openapitools/openapi-generator-cli` 是一个非常著名的 OpenAPI 生成器的工具包，您可以在 https://openapi-generator.tech/ 找到它。

使用此工具，您可以为各种编程语言生成客户端以及 JMeter 和 K6 等负载测试工具。

无需在您的计算机上安装该工具，只要从计算机可访问应用程序的 URL，可以使用 Docker 映像，如以下命令所述：

```bash
docker run --rm \
    -v ${PWD}:/local openapitools/openapi-generator-cli generate \
    -i /local/petstore.yaml \
    -g go \
    -o /local/out/go
```

该命令允许您使用挂载在 Docker 卷上的 `petstore.yaml` 文件中找到的 OpenAPI 定义以生成 Go 客户端。

现在，让我们详细了解如何在 .NET 6 项目中使用 Swagger，并应用到 `Minimal API` 中。

### Minimal APIs 中的 Swagger

在 ASP.NET Web API 中，如以下代码片段所示，我们看到一个方法记录在 C# 代码注释中，带有三重斜杠 (///)。

文档部分用于向 API 描述添加更多信息。 此外，`ProducesResponseType` 特性帮助 Swagger 识别客户端必须处理的可能代码作为方法调用的结果：

```csharp
/// <summary>
/// 创建一个联系人。
/// </summary>
/// <param name="contact"></param>
/// <returns>一个新创建的联系人</returns>
/// <response code="201">返回新创建的联系人</response>
/// <response code="400">如果联系人是 null</response>
[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create(Contact contactItem)
{
     _context.Contacts.Add(contactItem);
     await _context.SaveChangesAsync();
     return CreatedAtAction(nameof(Get), new { id = contactItem.Id }, contactItem);
}
```

除了单个方法上的注释外，Swagger 还由语言文档指导，为那些必须使用 API 应用程序的人提供进一步的信息。
方法参数的描述对于必须对接接口的人来说总是受欢迎的；不幸的是，在 Minimal API 中无法利用此功能。

让我们按顺序看看如何开始在单个方法上使用 Swagger：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() 
    { 
        Title = builder.Environment.ApplicationName,
        Version = "v1",
        Contact = new() 
        {
           Name = "PacktAuthor",
           Email = "authors@packtpub.com",
           Url = new Uri("https://www.packtpub.com/")
        },
        Description = "PacktPub Minimal API - Swagger",
        License = new Microsoft.OpenApi.Models.OpenApiLicense(),
        TermsOfService = new("https://www.packtpub.com/")
    });
});

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

在第一个示例中，我们配置了 Swagger 和一般的 Swagger 信息。我们包含了丰富 Swagger UI 的附加信息。
唯一必需的信息是标题，而版本、联系人、描述、许可证和服务条款是可选的。

`UseSwaggerUI()` 方法自动配置放置 UI 和使用 OpenAPI 格式描述 API 的 JSON 文件的位置。

这是现代桌面浏览器显示结果：
![Figure_3.2 - The Swagger UI](/assets/images/minimal-apis/Figure_3.2_B17902.jpg)

我们立即可以看到 OpenAPI 契约信息已经放在了 `/swagger/v1/swagger.json` 路径下。

联系信息已自动展示，但没有下面任何接口操作的文档，因为我们尚未开发任何接口操作。
API 应该有版本控制吗？是的，在右上角有个下拉框，我们可以选择具体的版本，已查看其可用的操作。

我们可以自定义 Swagger URL 并将 JSON 文档对应到新路径；
这里重要的是使用 `SwaggerEndpoint` 方法来重写配置，类似如下：
```csharp
app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", 
    $"{builder.Environment.ApplicationName} v1"));
```

接下来让我们添加具体业务逻辑的端点。

配置和定义 `RouteHandlerBuilder` 非常重要，因为它允许我们描述在代码中编写的端点的各种属性。

Swagger 的 UI 应当尽可能的丰富，因此，我们必须尽可能多的描述 Minimal API 所允许我们可以指定或者修改的内容。
但遗憾的是，并非所有 ASP.NET Web API 中的功能都可用。

#### Minimal APIs 中的版本控制

Minimal APIs 中的版本控制在当前框架中并不能处理；也就是说，Swagger 也不会在 UI 上显示 API 的多版本控制。
因此，我们观察到，当我们看到上图 3.2 所示的 “选择定义” 部分时，只有当前版本的 API 的一个条目可见。

#### Swagger 功能

我们刚刚了解到，并非所有功能都可以在 Swagger 中使用；现在让我们探索一下当前可用的方法。
为了描述端点可能的输出信息，我们需要调用附加在处理程序之后调用的一些函数，
例如我们现在要探讨的 `Produces` 或 `WithTags` 函数。

`Produces` 函数对所有返回给我们已知的客户端的端点的各种可能响应做装饰。
我们可以为操作添加名称（`OperationId`）；
此信息不会出现在 Swagger 屏幕中，但它将是客户端创建调用端点的方法的标识名称。 
`OperationId` 是处理程序提供的操作的唯一名称。

要从 API 描述中排除某个端点，您需要调用 `ExcludeFromDescription()`。
此函数很少使用，但在一些特定情况和场景非常有用，例如端点是底层或内部接口，但又不想暴露给前端开发人员。

最后，我们可以添加和标记各种端点并将它们分割以更好地进行客户端管理：

```csharp
app.MapGet("/sampleresponse", () =>
    {
        return Results.Ok(new ResponseData("My Response"));
    })
    .Produces<ResponseData>(StatusCodes.Status200OK)
    .WithTags("Sample")
    .WithName("SampleResponseOperation"); // operation ids to Open API

app.MapGet("/sampleresponseskipped", () =>
    {
        return Results.Ok(new ResponseData("My Response Skipped"));
    })
    .ExcludeFromDescription();

app.MapGet("/{id}", (int id) => Results.Ok(id));
app.MapPost("/", (ResponseData data) => Results.Ok(data))
   .Accepts<ResponseData>(MediaTypeNames.Application.Json);
```

如下这是 Swagger 的可视化结果；正如我之前预期的那样，标签和 `OperationId` 不在 Web 客户端显示：

![Figure_3.3 - Swagger UI methods](/assets/images/minimal-apis/Figure_3.3_B17902.jpg)

正因如此。从另一个角度来看，包含端点描述将是非常有用的。
当然实现起来非常简单：只需在方法中插入 C# 注释（只需在方法中插入三个斜杠 ///）。
Minimal API 没有像我们在基于 Web 的控制器中所惯用的方法，因为它们不受原生支持的。

Swagger 不仅仅是我们习惯看到的 GUI。
实际上，Swagger 还支持 OpenAPI 规范的 JSON 文件，较新版本为3.1.0。

在下面的代码片段中，我们展示了包含我们在 API 中插入的第一个端点的描述的部分。
我们可以推断出标签和操作 ID；这些信息将由那些与 API 交互的人员使用：

```json
"paths": {
    "/sampleresponse": {
        "get": {
            "tags": [
                "Sample"
            ],
            "operationId": "SampleResponseOperation",
            "responses": {
            "200": {
                "description": "Success",
                "content": {
                    "application/json": {
                        "schema": {
                            "$ref": "#/components/schemas/ResponseData"
                        }
                    }
                }
            }
        }
    }
},
```
在本节中，我们已经看到了如何配置 Swagger 以及目前还不支持的内容。  

在接下来的章节中，我们还将看到如何配置 OpenAPI，包括对 OpenID Connect 标准和通过 API 密钥进行身份验证。

在前面的 Swagger UI 代码片段中，Swagger 提供了涉及的对象的示意图，包括进入各个端点的入站和从它们出站的出站。

![Figure_3.4 - Input and output data schema](/assets/images/minimal-apis/Figure_3.4_B17902.jpg)

在第六章中，我们将学习如何处理这些对象以及如何验证和定义它们，探索验证和映射。

#### Swagger 操作过滤器

操作过滤器允许您为 Swagger 显示的所有操作添加行为。
在下面的示例中，我们将向您展示如何为特定调用添加一个 HTTP 头部，通过 `OperationId` 过滤它。

当您去定义一个操作过滤器时，您也可以基于路由、标签和 `OperationId` 设置过滤器：

```csharp
public class CorrelationIdOperationFilter : IOperationFilter
{
    private readonly IWebHostEnvironment environment;
    public CorrelationIdOperationFilter(IWebHostEnvironment environment)
    {
        this.environment = environment;
    }
    
    /// <summary>
    /// Apply header in parameter Swagger.
    /// We add default value in parameter for developer environment
    /// </summary>
    /// <param name="operation"></param>
    /// <param name="context"></param>
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (operation.Parameters == null)
        {
            operation.Parameters = new List<OpenApiParameter>();
        }
        if (operation.OperationId == "SampleResponseOperation")
        {
             operation.Parameters.Add(new OpenApiParameter
             {
                 Name = "x-correlation-id",
                 In = ParameterLocation.Header,
                 Required = false,
                 Schema = new OpenApiSchema 
                 {
                     Type = "String", 
                     Default = new OpenApiString("42") 
                 }
             });
        }
    }
}
```

要定义操作过滤器，必须实现 `IOperationFilter` 接口。

在构造函数中，您可以定义所有已在依赖注入引擎中注册的接口或对象。

然后，过滤器由一个名为 Apply 的方法组成，该方法提供两个对象：  

- **OpenApiOperation**：一个操作，我们可以在其中添加参数或检查当前调用的操作 ID
- **OperationFilterContext**：过滤器上下文，允许您读取 ApiDescription，您可以在其中找到当前端点的 URL

最后，要在 Swagger 中启用操作过滤器，我们需要在 SwaggerGen 方法中注册它。 
在此方法中，我们应添加过滤器，如下所示

```csharp
builder.Services.AddSwaggerGen(c =>
{
     // ... removed for brevity
     c.OperationFilter<CorrelationIdOperationFilter>();
});
```

以下是用户界面看到的结果；在端点中，只有对于特定的 `OperationId`，
我们将有一个新的必填头部，该头部带有一个默认参数，在开发中不需要手动插入：

![Figure 3.5 – API key section](/assets/images/minimal-apis/Figure_3.5_B17902.jpg)

此案例研究在我们需要设置 API key 且不希望在每次调用时都插入它时，对我们帮助很大。

#### 生产环境中的操作过滤器

由于在生产环境中不应启用 Swagger，因此过滤器及其默认值不会造成应用程序安全问题。

强烈建议您在生产环境中禁用 Swagger。

在本节中，我们了解了如何启用一个 UI 工具，该工具描述了 API 并允许我们对其进行测试。
在下一节中，我们将看到如何通过 CORS 启用单页应用程序（SPAs）与后端的调用。

## 启用 CORS

CORS（跨源资源共享）是一种安全机制，通过该机制，如果HTTP/S请求来自于与托管应用程序不同的域，则该请求会被阻止。
更多信息可以在 Microsoft 文档或 Mozilla 开发者站点上找到。

浏览器默认会阻止网页向除了提供该网页的域之外的任何域发起请求。
一个网页、单页应用程序（SPA）或服务器端网页可以向托管在不同源上的几个后端 API 发起 HTTP 请求。

这种限制被称为同源策略。同源策略防止恶意网站读取另一个网站的数据。
浏览器不会阻止 HTTP 请求，但会阻止响应数据。

因此，我们理解，就安全性而言，必须谨慎评估 CORS 资格。

最常见的场景是，SPA发布在与托管于 Minimal API 服务的不同 Web 地址上的 Web 服务器上。

![Figure 3.6 – SPA and minimal API](/assets/images/minimal-apis/Figure_3.6_B17902.jpg)

类似的场景是微服务之间需要相互通信，因为每个微服务将驻留在与其他服务不同的特定网络地址上。

![Figure 3.7 – Microservices and minimal APIs](/assets/images/minimal-apis/Figure_3.7_B17902.jpg)

在所有这些情况下，显然都会遇到 CORS 问题。

我们现在理解了可能发生 CORS 请求的情况。
现在，让我们看看正确的 HTTP 请求流程是什么，以及浏览器如何处理请求。

### 从 HTTP 请求看 CORS 流程

当调用离开浏览器前往与前端托管地址不同的地址时会发生什么？

HTTP 调用被执行并一直到达后端代码，后端代码正确执行。

但包含正确数据的响应被浏览器阻止。
这就是为什么当我们使用 Postman、Fiddler 或任何 HTTP 客户端执行调用时，响应能够正确到达我们的原因。

![Figure 3.8 – CORS flow](/assets/images/minimal-apis/Figure_3.8_B17902.jpg)

在下面的图中，我们可以看到浏览器首先使用 OPTIONS 方法进行调用，后端以 204 状态码正确响应：

![Figure 3.9 – First request for the CORS call (204 No Content result)](/assets/images/minimal-apis/Figure_3.9_B17902.jpg)

随后在浏览器发起的第二个调用是，这里就报错了；
在 `Referrer Policy` 信息里显示 `strict-origin-when-cross-origin`，以表示浏览器拒绝从服务器后端接收数据

![Figure 3.10 – Second request for the CORS call (blocked by the browser)](/assets/images/minimal-apis/Figure_3.10_B17902.jpg)

当 CORS 启用时，在对 OPTIONS 方法调用的响应中，会插入三个具有后端愿意遵守的特征的头：

![Figure 3.11 – Request for CORS call (with CORS enabled)](/assets/images/minimal-apis/Figure_3.11_B17902.jpg)

在这种情况下，我们可以看到添加了三个头，
它们定义了 `Access-Control-Allow-Headers`、`Access-Control-Allow-Methods` 和 `Access-Control-Allow-Origin`。

浏览器可以使用此信息接受或阻止对该 API 的响应。

### 设置 CORS 使用策略

在.NET 6 应用程序中，可以通过多种配置来激活 CORS。我们可以定义授权策略，在其中可以配置四个可用设置。
也可以通过添加扩展方法或注释来激活 CORS。

但让我们逐一展示。

`CorsPolicyBuilder` 类允许我们定义 CORS 接受策略中允许或不允许的内容。

因此，我们可以设置不同的方法，例如：

- `AllowAnyHeader`
- `AllowAnyMethod`
- `AllowAnyOrigin`
- `AllowCredentials`

前三种方法是描述性的，分别允许我们启用与 HTTP 调用的头、方法和源相关的任何设置，而 `AllowCredentials` 允许我们在认证凭证中包含 cookie。

#### CORS 策略建议

我们建议不要使用 `AllowAny` 方法，而是筛选出必要信息以提高安全性。
作为最佳实践，在启用 CORS 时，我们建议使用以下方法：

- `WithExposedHeaders`
- `WithHeaders`
- `WithOrigins`

为了模拟 CORS 场景，我们创建了一个带有三个不同按钮的简单前端应用程序。
每个按钮允许你测试 Minimal API 中 CORS 的一种可能配置。我们将在后续解释这些配置。

为了启用 CORS 场景，我们创建了一个可以在内存中的 Web 服务器上启动的单页应用程序。
我们使用了 `LiveReloadServer`，这是一个可以使用.NET CLI 安装的工具。
我们在本章开头提到过它，现在是时候使用它了。

安装后，需要使用以下命令启动 SPA：

```bash
$ livereloadserver "{BasePath}\Chapter03\2-CorsSample\Frontend"
```

这里，BasePath 是你要下载 GitHub 上示例的文件夹。

然后，就得启动应用程序后端，可以通过 Visual Studio、Visual Studio Code 或使用以下命令通过 .NET CLI 启动：

```bash
$ dotnet run .\Backend\CorsSample.csproj
```

我们已经了解了如何启动一个突出 CORS 问题的示例；
现在我们需要配置服务器以接受请求并告知浏览器它知道请求来自不同的源。

接下来，我们将讨论策略配置。我们将了解默认策略的特征以及如何创建自定义策略。

#### 配置默认策略

要配置单个启用 CORS 的策略，需要在 `Program.cs` 文件中定义行为并添加所需的配置。
让我们实现一个策略并将其定义为 `Default`。

然后，要为整个应用程序启用该策略，只需在定义处理程序之前添加 `app.UseCors();`：


```csharp
var builder = WebApplication.CreateBuilder(args);
var corsPolicy = new CorsPolicyBuilder("http://localhost:5200")
    .AllowAnyHeader()
    .AllowAnyMethod()
    .Build();
builder.Services.AddCors(c => c.AddDefaultPolicy(corsPolicy));

var app = builder.Build();
app.UseCors();
app.MapGet("/api/cors", () =>
{
    return Results.Ok(new { CorsResultJson = true });
});
app.Run();
```

#### 配置自定义策略

我们可以在应用程序中创建多个策略；每个策略可能有自己的配置，并且每个策略可能与一个或多个端点相关联。

在微服务的情况下，拥有多个策略有助于精确地分割来自不同源的访问。

要配置新策略，需要添加它并给它一个名称；这个名称将提供对策略的访问并允许它与端点相关联。

与前面的示例一样，自定义策略被分配给整个应用程序：

```csharp
var builder = WebApplication.CreateBuilder(args);
var corsPolicy = new CorsPolicyBuilder("http://localhost:5200")
    .AllowAnyHeader()
    .AllowAnyMethod()
    .Build();
builder.Services.AddCors(options => options.AddPolicy("MyCustomPolicy", corsPolicy));
var app = builder.Build();
app.UseCors("MyCustomPolicy");
app.MapGet("/api/cors", () =>
{
    return Results.Ok(new { CorsResultJson = true });
});
app.Run();
```

接下来，我们看看如何将单个策略应用于特定端点；为此，有两种方法可用。
第一种是通过 `IEndpointConventionBuilder` 接口的扩展方法。第二种方法是添加 `EnableCors` 注释，后跟要为该方法启用的策略名称。

### 使用扩展方法设置 CORS

需要使用 `RequireCors` 方法后跟策略名称。

通过这种方法，可以为一个端点启用一个或多个策略：

```csharp
app.MapGet("/api/cors/extension", () =>
{
    return Results.Ok(new { CorsResultJson = true });
})
.RequireCors("MyCustomPolicy");
```

### 使用注解设置 CORS

第二种方法是添加 EnableCors 注解，后跟要为该方法启用的策略名称：

```csharp
app.MapGet("/api/cors/annotation", [EnableCors("MyCustomPolicy")] () =>
{
   return Results.Ok(new { CorsResultJson = true });
});
```

对于控制器编程，很快就会发现不可能将策略应用于特定控制器的所有方法。
也不可能对控制器进行分组并启用策略。因此，必须将单个策略应用于方法或整个应用程序。

在本节中，我们了解了如何为托管在不同域上的应用程序配置浏览器保护。

在下一节中，我们将开始配置我们的应用程序。

## 使用全局 API 设置

我们刚刚定义了如何在 ASP.NET 应用程序中使用选项模式加载数据。在本节中，我们想描述如何配置应用程序并利用我们在上一节中看到的所有内容。

随着 **.NET Core** 的诞生，配置从 `Web.config` 文件转移到了 `appsettings.json` 文件。
配置也可以从其他来源读取，例如旧的 **.ini** 文件或位置文件等其他文件格式。

在 Minimal API 中，选项模式功能保持不变，但在接下来的几段中，我们将看到如何重用接口或 `appsettings.json` 文件结构。

### .NET 6 中的配置

.NET 提供的对象是 `IConfiguration`，它允许我们读取 `appsettings` 文件中的一些特定配置。

但是，如前所述，这个接口的作用远不止于访问文件进行读取。

以下是官方文档中的一段摘录，帮助我们理解该接口是如何作为通用访问点，允许我们访问插入到各种服务中的数据：

> ASP.NET Core 中的配置是使用一个或多个配置提供程序执行的。
> 配置提供程序使用各种配置源从键值对中读取配置数据。

以下是配置源的列表：

- 设置文件，如 `appsettings.json`
- 环境变量
- Azure Key Vault
- Azure App Configuration
- 命令行参数
- 已安装或创建的自定义提供程序
- 目录文件
- 内存中的.NET 对象

([https://docs.microsoft.com/aspnet/core/fundamentals/configuration/](https://docs.microsoft.com/aspnet/core/fundamentals/configuration/))

`IConfiguration` 和 `IOptions` 接口（我们将在下一章中看到）旨在从各种提供程序读取数据。
这些接口不适合在程序运行时读取和编辑配置文件。

`IConfiguration` 接口可通过 `builder` 对象（`builder.Configuration`）获得，它提供了读取值、对象或连接字符串所需的所有方法。

在查看了我们将用于配置应用程序的最重要接口之一后，我们想要定义良好的开发实践并使用任何开发人员的基本构建块：即类。
将配置复制到类中，将允许我们在代码的任何地方更好地使用内容。

我们定义包含属性的类和与 `appsettings` 文件对应的类：

Configuration classes

```csharp
public class MyCustomObject
{
    public string? CustomProperty { get; init; }
}
public class MyCustomStartupObject
{
    public string? CustomProperty { get; init; }
}
```

以下是我们刚刚看到的 C# 类对应的 JSON：

appsettings.json definition

```json
{
    "MyCustomObject": {
         "CustomProperty": "PropertyValue"
    },
    "MyCustomStartupObject": {
         "CustomProperty": "PropertyValue"
    },
    "ConnectionStrings": {
         "Default": "MyConnectionstringValueInAppsettings"
    }
}
```

接下来，我们将执行几个操作。

我们执行的第一个操作创建一个 `startupConfig` 对象的实例，该对象将是 `MyCustomStartupObject` 类型。
为了填充这个对象的实例，通过 `IConfiguration`，我们将从名为 `MyCustomStartupObject` 的部分读取数据：

```csharp
var startupConfig = builder.Configuration.GetSection(nameof(MyCustomStartupObject))
    .Get<MyCustomStartupObject>();
```

新创建的对象然后可以在 Minimal API 的各种处理程序中使用。

相反，在第二个操作中，我们使用依赖注入引擎请求 `IConfiguration` 对象的实例：

```csharp
app.MapGet("/read/configurations", (IConfiguration configuration) =>
{
    var customObject = configuration.GetSection(nameof(MyCustomObject))
        .Get<MyCustomObject>();
```

通过 `IConfiguration` 对象，我们将以类似于刚刚描述的操作的方式检索数据。
我们选择 `GetSection (nameof (MyCustomObject))` 部分，并使用 `Get<T>()` 方法将对象类型化为所需类型。

最后，在最后两个示例中，我们读取 `appsettings` 文件根级别上的单个键：

```csharp
MyCustomValue = configuration.GetValue<string>("MyCustomValue"),
ConnectionString = configuration.GetConnectionString("Default"),
```

`configuration.GetValue<T>("JsonRootKey")` 方法提取键的值并将其转换为对象；此方法用于从根级别属性读取字符串或数字。

在下一行中，我们可以看到如何利用 `IConfiguration` 方法读取 `ConnectionString`。

在 `appsettings` 文件中，连接字符串放置在一个特定的部分 `ConnectionStrings` 中，允许你命名字符串并读取它。
可以在这个部分放置多个连接字符串，以便在不同的对象中使用它们。

在 Azure App Service 的配置提供程序中，连接字符串应该带有一个前缀输入，该前缀也指示你正在尝试使用的 SQL 提供程序，如以下链接所述：
[https://docs.microsoft.com/azure/app-service/configure-common#configure-connection-strings](https://docs.microsoft.com/azure/app-service/configure-common#configure-connection-strings).

在运行时，连接字符串作为环境变量可用，带有以下连接类型前缀：

- SQLServer: **SQLCONNSTR_**
- MySQL: **MYSQLCONNSTR_**
- SQLAzure: **SQLAZURECONNSTR_**
- Custom: **CUSTOMCONNSTR_**
- PostgreSQL: **POSTGRESQLCONNSTR_**

为了完整性，我们将带回刚刚描述的整个代码，以便更好地全面了解如何在代码中利用 `IConfiguration` 对象：

```csharp
var builder = WebApplication.CreateBuilder(args);
var startupConfig = builder.Configuration.GetSection(nameof(MyCustomStartupObject))
    .Get<MyCustomStartupObject>();
app.MapGet("/read/configurations", (IConfiguration configuration) =>
{
    var customObject = configuration.GetSection(nameof(MyCustomObject))
        .Get<MyCustomObject>();
    return Results.Ok(new
    {
        MyCustomValue = configuration.GetValue<string>("MyCustomValue"),
        ConnectionString = configuration.
        GetConnectionString("Default"),
        CustomObject = customObject,
        StartupObject = startupConfig
    });
})
.WithName("ReadConfigurations");
```

我们已经看到了如何利用带有连接字符串的 `appsettings` 文件，但通常情况下，每个环境都有许多不同的文件。
让我们看看如何为每个环境利用一个文件。

#### `appsettings` 文件中的优先级

`appsettings` 文件可以根据应用程序所在的环境进行管理。
在这种情况下，实践是将该环境的关键信息放置在 `appsettings.{ENVIRONMENT}.json` 文件中。

根文件（即 `appsettings.json`）应该仅用于生产环境。

例如，如果我们在两个文件中为 "Priority" 键创建这些示例，我们会得到什么？

appsettings.json

```json
"Priority": "Root"
```
appsettings.Development.json
```json
"Priority":"Dev"
```

如果是*开发环境*，键的值将是 **Dev**，而在*生产环境*中，值将是 **Root**。

如果环境不是*生产*或*开发*环境会发生什么？例如，如果它被称为 _Stage_？
在这种情况下，如果没有指定任何 `appsettings.Stage.json` 文件，
读取的值将是 `appsettings.json` 文件之一的值，因此是 **Root**。

但是，如果我们指定了 `appsettings.Stage.json` 文件，将从该文件读取值。

接下来，让我们看看选项模式。
有一些框架提供的对象，用于在启动时或系统部门进行更改时加载配置信息。
让我们看看如何实现。

### Options 模式

`Options` 模式使用类来提供对相关设置组的强类型访问，即当配置设置根据场景隔离到单独的类中时。

`Options` 模式将通过不同的接口和不同的功能来实现。每个接口（见以下小节）都有自己的功能，帮助我们实现某些目标。

但让我们按顺序进行。我们为每种类型的接口定义一个对象（我们这样做是为了更好地表示示例），但同一个类可以用于在配置文件中注册更多选项。
这里保持文件结构同样很重要：

```csharp
public class OptionBasic
{
    public string? Value { get; init; }
}

public class OptionSnapshot
{
    public string? Value { get; init; }
}

public class OptionMonitor
{
    public string? Value { get; init; }
}

public class OptionCustomName
{
    public string? Value { get; init; }
}
```

每个选项都通过 `Configure` 方法在依赖注入引擎中注册，该方法还需要注册方法签名中存在的 `T` 类型。
如你所见，在注册阶段，我们声明了类型和文件中检索信息的部分，仅此而已：

```csharp
builder.Services.Configure<OptionBasic>(
    builder.Configuration.GetSection("OptionBasic"));
builder.Services.Configure<OptionMonitor>(
    builder.Configuration.GetSection("OptionMonitor"));
builder.Services.Configure<OptionSnapshot>(
    builder.Configuration.GetSection("OptionSnapshot"));
builder.Services.Configure<OptionCustomName>("CustomName1", 
    builder.Configuration.GetSection("CustomName1"));
builder.Services.Configure<OptionCustomName>("CustomName2", 
    builder.Configuration.GetSection("CustomName2"));
```

我们无须定义应该如何读取对象、读取的频率以及使用什么类型的接口。

唯一改变的是参数，如前面代码片段的最后两个示例所示。这个参数允许你为选项类型添加一个名称。
名称需要与方法签名中使用的类型匹配。这个功能亦因此称为**强命名选项（named options）**。

#### 其他 option 接口

其他 option 接口可以利用你刚刚定义的记录。一些支持命名选项，而一些不支持：

- `IOptions<TOptions>`:
  - 不支持以下内容：
    - 应用程序启动后读取配置数据
    - 强命名选项
  - 注册为单例并注入到任何服务生命周期中
- `IOptionsSnapshot<TOptions>`:
  - 在每个请求都应重新计算选项的场景中很有用
  - 注册为作用域，因此不能注入到单例服务中
  - 支持强命名选项
- `IOptionsMonitor<TOptions>`:
  - 用于检索选项并管理 `TOptions` 实例的选项通知
  - 注册为单例并可以注入到任何服务生命周期中
  - 支持以下内容：
    - 更改通知
    - 强命名选项
    - 可重新加载的配置
    - 选择性选项失效 (`IOptionsMonitorCache<TOptions>`)

我们想提醒你使用 `IOptionsFactory<TOptions>`，它负责创建选项的新实例。它有一个单一的 `Create` 方法。
默认实现获取所有已注册的 `IConfigureOptions<TOptions>` 和 `IPostConfigureOptions<TOptions>`，并首先执行所有配置，
然后执行后配置 ([https://docs.microsoft.com/aspnet/core/fundamentals/configuration/options#options-interfaces](https://docs.microsoft.com/aspnet/core/fundamentals/configuration/options#options-interfaces))。

`Configure` 方法之后可以跟在配置管道中的另一个方法。这个方法称为 `PostConfigure`，用于在每次配置或重新读取时修改配置。
以下是如何记录这种行为的示例：

```csharp
builder.Services.PostConfigure<MyConfigOptions>(myOptions =>
{
   myOptions.Key1 = "my_new_value_post_configuration";
});
```

#### 综合应用

定义了这些众多接口的理论之后，我们还需要看看 `IOptions` 在具体示例中的工作情况。

让我们看看刚刚描述的三个接口的使用以及 `IOptionsFactory` 的使用，它与 `Create` 方法和命名选项功能一起，检索对象的正确实例：

```csharp
app.MapGet("/read/options", (IOptions<OptionBasic> optionsBasic,
         IOptionsMonitor<OptionMonitor> optionsMonitor,
         IOptionsSnapshot<OptionSnapshot> optionsSnapshot,
         IOptionsFactory<OptionCustomName> optionsFactory) =>
{
    return Results.Ok(new
    {
        Basic = optionsBasic.Value,
        Monitor = optionsMonitor.CurrentValue,
        Snapshot = optionsSnapshot.Value,
        Custom1 = optionsFactory.Create("CustomName1"),
        Custom2 = optionsFactory.Create("CustomName2")
    });
})
.WithName("ReadOptions");
```

在前面的代码片段中，我们想提醒你注意使用不同的可用接口。

前面片段中使用的每个单独接口都有一个特定的生命周期，这决定了它的行为。
最后，每个接口在方法上都有细微的差异，正如我们在前面段落中已经描述的那样。

#### IOptions 和验证

最后但并非最不重要的是配置中数据的验证功能。
当必须发布应用程序的团队仍然执行手动或精细操作时，这非常有用，这些操作至少需要由代码进行验证。

在 .NET Core 出现之前，由于配置不正确，应用程序经常无法启动。
现在，有了这个功能，我们可以验证配置中的数据并抛出错误。

以下是一个示例：

Register option with validation

```csharp
builder.Services.AddOptions<ConfigWithValidation>()
    .Bind(builder.Configuration.GetSection(nameof(ConfigWithValidation)))
    // BELOW LINE ADD VALIDATION
    .ValidateDataAnnotations();

app.MapGet("/read/options", (IOptions<ConfigWithValidation> optionsValidation) =>
{
    return Results.Ok(new
    {
        Validation = optionsValidation.Value
    });
})
.WithName("ReadOptions");
```

这是一个明确报告错误的配置文件：

```json
"ConfigWithValidation": {
    "Email": "andrea.tosato@hotmail.it",
    "NumericRange": 1001
}
```

以下是包含验证逻辑的类：

```csharp
public class ConfigWithValidation
{
    [RegularExpression(@"^([\w\.\-]+)@([\w\-]+)((\.(\w){2,})+)$")]
    public string? Email { get; set; }
    
    [Range(0, 1000, ErrorMessage = "Value for {0} must be between {1} and {2}.")]
    public int NumericRange { get; set; }
}
```

应用程序在使用特定配置时会遇到错误，而不是在启动时。这也是因为，正如我们之前所见，`IOptions` 可以在 `appsettings` 更改后重新加载信息。

```
Microsoft.Extensions.Options.OptionsValidationException: 
  DataAnnotation validation failed for 'ConfigWithValidation' members: 'NumericRange' with the error:
    'Value for NumericRange must be between 0 and 1000.'.
```

#### 验证 `IOptions` 的最佳实践

此设置并不适用于所有应用场景，只有一些选项可以进行正式验证；
如果我们考虑连接字符串，它不一定在形式上不正确，但连接可能无法工作。

在应用此功能时要谨慎，特别是因为它在运行时报告错误而不是在启动时，
并且会给出内部服务器错误，这在应该处理的场景中不是最佳实践。

到目前为止，我们所看到的一切都是关于配置 `appsettings.json` 文件的，
但如果我们想使用其他来源进行配置管理呢？我们将在下一节中探讨这个问题。

### 配置源

如本节开头所述，`IConfiguration` 接口和所有 `IOptions` 变体不仅适用于 `appsettings` 文件，还适用于不同的来源。

每个来源都有其自身的特点，并且在提供程序之间访问对象的语法非常相似。 主要问题是当我们必须定义一个复杂对象或对象数组时；
在这种情况下，我们将看到如何操作并能够复制 JSON 文件的动态结构。

让我们看两个非常常见的用例。

#### 在 Azure App Service 中配置应用程序

让我们从 Azure 开始，特别是 Azure Web Apps 服务。

在 **“配置”** 页面上，有两个部分：**“应用程序设置”** 和 **“连接字符串”**。

在第一部分中，我们需要插入我们在前面示例中看到的键和值或 JSON 对象。

在 **“连接字符串”** 部分，你可以插入通常插入在 `appsettings.json` 文件中的连接字符串。
在本节中，除了文本字符串外，还需要设置连接类型，正如我们在 _“.NET 6 中的配置”_ 部分中看到的那样。

![Figure 3.12 – Azure App Service Application settings](/assets/images/minimal-apis/Figure_3.12_B17902.jpg)

##### 插入一个对象

要插入一个对象，我们必须为每个键指定父级。

格式如下：

**parent__key**

注意有**两个**下划线。

The object in the JSON file would be defined as follows:

```json
"MyCustomObject": {
    "CustomProperty": "PropertyValue"
}
```

所以，我们应该写 **MyCustomObject__CustomProperty**。

##### 插入一个数组

插入一个数组要详细得多。

格式如下：

**parent__child__ArrayIndexNumber_key**

JSON 文件中的数组将定义如下：

```json
{
    "MyCustomArray": {
        "CustomPropertyArray": [
            { "CustomKey": "ValueOne" },
            { "CustomKey ": "ValueTwo" }
        ]
    }
}
```

因此，要访问 `ValueOne` 值，我们应该写：

```
MyCustomArray__CustomPropertyArray__0__CustomKey
```

#### 在 Docker 中配置应用程序

如果我们正在为容器（即 Docker）进行开发，通常会在 `docker-compose` 文件中替换 `appsettings` 文件，
并且通常在覆盖文件中替换，因为它的行为类似于按环境划分的设置文件。

我们想简要概述通常用于配置托管在 Docker 中的应用程序的功能。
让我们详细看看如何定义根键和对象，以及如何设置连接字符串。以下是一个示例：

```csharp
app.MapGet("/env-test", (IConfiguration configuration) =>
{
    var rootProperty = configuration.GetValue<string>("RootProperty");
    var sampleVariable = configuration.GetValue<string>("RootSettings:SampleVariable");
    var connectionString = configuration.GetConnectionString("SqlConnection");
    return Results.Ok(new
    {
        RootProperty = rootProperty,
        SampleVariable = sampleVariable,
        Connection String = connectionString
    });
})
.WithName("EnvironmentTest");
```

使用配置的 Minimal API 的 `docker-compose.override.yaml` 文件如下：

```yaml
services:
  dockerenvironment:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - RootProperty=minimalapi-root-value
      - ConnectionStrings__SqlConnection=Server=minimal.db;Database=minimal_db;User Id=sa;Password=Taggia42!
```

这个示例中只有一个应用程序容器，实例化它的服务称为 `dockerenvironment`。

在配置部分，我们可以看到三个特殊之处，我们将逐行分析。

我们想向你展示的代码片段有几个非常有趣的组件：
配置根目录中的一个属性、由单个属性组成的一个对象以及到数据库的连接字符串。

在第一个配置中，你将设置一个作为配置根的属性。在这种情况下，它是一个简单的字符串：
```yaml
# First configuration
- RootProperty=minimalapi-root-value
```

在第二个配置中，我们将设置一个对象：
```yaml
# Second configuration
- RootSettings__SampleVariable=minimalapi-variable-value
```

对象称为 `RootSettings`，而它包含的唯一属性称为 `SampleVariable`。这个对象可以通过不同的方式读取。
我们建议使用我们之前广泛看到的 `IOptions` 对象。在前面的示例中，我们展示了如何通过代码访问对象中存在的单个属性。

在这种情况下，通过代码，你需要使用以下符号来访问值：`RootSettings:SampleVariable`。
这种方法在你需要读取单个属性时很有用，但我们建议使用 `IOptions` 接口来访问对象。

在最后一个示例中，我们向你展示如何设置名为 `SqlConnection` 的连接字符串。
这样，就可以很容易地从 `IConfiguration` 可用的基本方法中检索信息：
```yaml
# Third configuration
- ConnectionStrings__SqlConnection=Server=minimal.db;Database=minimal_db;User Id=sa;Password=Taggia42!
```

要读取信息，只需要暴露这样一个方法即可：

```csharp
GetConnectionString(“SqlConnection”)
```

有很多配置我们应用程序的场景；在下一节中，我们还将看到如何处理错误。

## 错误处理

错误处理是每个应用程序都必须提供的功能之一。错误的表示允许客户端理解错误并可能相应地处理请求。
通常，我们有自己定制的错误处理方法。

由于我们正在描述的是应用程序的关键功能，我们认为看看框架提供了什么以及使用什么更正确是合理的。

### 传统方法

.NET 为 Minimal API 提供了与传统开发中相同的工具：开发人员异常页面。这只不过是一个以纯文本格式报告错误的中间件。
这个中间件不能从 ASP.NET 管道中删除，并且仅在开发环境中工作 
([https://docs.microsoft.com/aspnet/core/fundamentals/error-handling](https://docs.microsoft.com/aspnet/core/fundamentals/error-handling)).

![Figure 3.13 – Minimal APIs pipeline, ExceptionHandler](/assets/images/minimal-apis/Figure_3.13_B17902.jpg)

如果在我们的代码中引发异常，在应用层捕获它们的唯一方法是通过在将响应发送到客户端之前激活的中间件。

错误处理中间件是标准的，可以如下实现：

```csharp
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = Application.Json;
        var exceptionHandlerPathFeature = 
            context.Features.Get<IExceptionHandlerPathFeature>()!;
        var errorMessage = new
        {
            Message = exceptionHandlerPathFeature.Error.Message
        };
        await context.Response.WriteAsync(JsonSerializer.Serialize(errorMessage));
        if (exceptionHandlerPathFeature?.Error is FileNotFoundException)
        {
            await context.Response.WriteAsync(" The file was not found.");
        }
        if (exceptionHandlerPathFeature?.Path == "/")
        {
            await context.Response.WriteAsync("Page: Home.");
        }
    });
});
```

我们在这里展示了中间件的一种可能实现。为了实现它，必须利用 `UseExceptionHandler` 方法，允许编写整个应用程序的管理代码。

通过 `var exceptionHandlerPathFeature = context.Features.Get<IExceptionHandlerPathFeature>()!;`,
我们可以访问错误堆栈并在输出中返回调用者感兴趣的信息：

```csharp
app.MapGet("/ok-result", () =>
{
    throw new ArgumentNullException("taggia-parameter", "Taggia has an error");
})
.WithName("OkResult");
```

当代码中发生异常时，如前面的示例所示，中间件会介入并处理返回给客户端的消息。

如果异常发生在内部应用程序堆栈中，中间件仍会介入，为客户端提供正确的错误和适当的指示。

### 问题详情和 IETF 标准

HTTP API 的问题详情是 2016 年批准的 IETF 标准。此标准允许使用标准字段和 JSON 表示法向调用者返回一组信息，以帮助识别错误。

HTTP 状态码有时不足以传达有关错误的足够信息以使其有用。
虽然 Web 浏览器背后的人可以通过 HTML 响应体了解问题的性质，但所谓 HTTP API 的非人类消费者（如机器、PC 和服务器）通常不能。

此规范定义了简单的 JSON 和 XML 文档格式以满足此目的。
它们旨在被 HTTP API 重用，这些 API 可以识别特定于其需求的不同问题类型。

因此，API 客户端可以被告知高级错误类以及问题的更详细信息 
([https://datatracker.ietf.org/doc/html/rfc7807](https://datatracker.ietf.org/doc/html/rfc7807)).

在.NET 中，有一个包含符合 IETF 标准的所有功能的包。

该包名为 `Hellang.Middleware.ProblemDetails`，你可以从以下地址下载：
[https://www.nuget.org/packages/Hellang.Middleware.ProblemDetails/](https://www.nuget.org/packages/Hellang.Middleware.ProblemDetails/).

现在让我们看看如何将该包插入项目并配置它：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.TryAddSingleton<IActionResultExecutor<ObjectResult>, 
    ProblemDetailsResultExecutor>();
builder.Services.AddProblemDetails(options =>
{
    options.MapToStatusCode<NotImplementedException>(StatusCodes.Status501NotImplemented);
});
var app = builder.Build();
app.UseProblemDetails();
```

如你所见，使这个包工作只需要两行代码：

- `builder.Services.AddProblemDetails`
- `app.UseProblemDetails();`

由于在 Minimal API 中，`IActionResultExecutor` 接口不在 ASP.NET 管道中，因此需要添加一个自定义类来在发生错误时处理响应。

为此，你需要添加一个类（如下所示）并在依赖注入引擎中注册它：

```csharp
builder.Services.TryAddSingleton<IActionResultExecutor<ObjectResult>,
    ProblemDetailsResultExecutor>();
```

以下是支持该包的类，也适用于 Minimal API：

```csharp
public class ProblemDetailsResultExecutor : IActionResultExecutor<ObjectResult>
{
    public virtual Task ExecuteAsync(ActionContext context, ObjectResult result)
    {
        ArgumentNullException.ThrowIfNull(context);
        ArgumentNullException.ThrowIfNull(result);
        var executor = Results.Json(result.Value, 
            null, 
            "application/problem+json", 
            result.StatusCode);
        return executor.ExecuteAsync(context.HttpContext);
    }
}
```

如前所述，处理错误消息的标准在 IETF 标准中已经存在了几年，但对于 C# 语言，需要添加刚刚提到的包。

现在让我们看看这个包如何处理我们在这里报告的一些端点上的错误：

```csharp
app.MapGet("/internal-server-error", () =>
{
    throw new ArgumentNullException("taggia-parameter", "Taggia has an error");
})
.Produces<ProblemDetails>(StatusCodes.Status500InternalServerError)
.WithName("internal-server-error");
```

我们使用这个端点抛出一个应用程序级别的异常。
在这种情况下，`ProblemDetails` 中间件会返回一个与错误一致的 JSON 错误。
然后我们免费获得了对未处理异常的处理：

```json
{
    "type": "https://httpstatuses.com/500",
    "title": "Internal Server Error",
    "status": 500,
    "detail": "Taggia has an error (Parameter 'taggia-parameter')",
    "exceptionDetails": [{
        /* for brevity */
    }],
    "traceId": "00-f6ff69d6f7ba6d2692d87687d5be75c5-e734f5f081d7a02a-00"
}
```

通过在 `Program` 文件中插入其他配置，你可以将一些特定的异常映射到 HTTP 错误。以下是一个示例：

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.MapToStatusCode<NotImplementedException>(StatusCodes.Status501NotImplemented);
});
```

带有 `NotImplementedException` 异常的代码被映射到 HTTP 错误代码 501：

```csharp
app.MapGet("/not-implemented-exception", () =>
{
    throw new NotImplementedException("This is an exception thrown from a Minimal API.");
})
.Produces<ProblemDetails>(StatusCodes.Status501NotImplemented)
.WithName("NotImplementedExceptions");
```

最后，可以使用其他字段扩展框架的 `ProblemDetails` 类或通过添加自定义文本来调用基方法。

以下是 `MapGet` 端点处理程序的最后两个示例：

```csharp
app.MapGet("/problems", () =>
{
    return Results.Problem(detail: "This will end up in the 'detail' field.");
})
.Produces<ProblemDetails>(StatusCodes.Status400BadRequest)
.WithName("Problems");

app.MapGet("/custom-error", () =>
{
    var problem = new OutOfCreditProblemDetails
    {
        Type = "https://example.com/probs/out-of-credit",
        Title = "You do not have enough credit.",
        Detail = "Your current balance is 30, but that costs 50.",
        Instance = "/account/12345/msgs/abc",
        Balance = 30.0m, 
        Accounts = { "/account/12345", "/account/67890" }
    };
    return Results.Problem(problem);
})
.Produces<OutOfCreditProblemDetails>(StatusCodes.Status400BadRequest)
.WithName("CreditProblems");

app.Run();

public class OutOfCreditProblemDetails : ProblemDetails
{
    public OutOfCreditProblemDetails()
    {
        Accounts = new List<string>();
    }
    public decimal Balance { get; set; }
    public ICollection<string> Accounts { get; }
}
```

## 总结

在本章中，我们看到了 Minimal API 实现的几个高级方面。
我们探索了 Swagger，它用于为 API 编写文档并为开发人员提供方便的工作调试环境。
我们看到了 CORS 如何处理托管在与当前 API 不同地址的应用程序的问题。
最后，我们看到了如何加载配置信息并处理应用程序中的意外错误。

我们探索了将使我们在短时间内提高生产力的要点。

在下一章中，我们将添加面向 SOLID 模式编程的基本构建块，即依赖注入引擎，
它将帮助我们更好地管理分散在各个层中的应用程序代码。

<br/><br/><br/><br/>
&gt;  [返回扉页](/books/minimal-apis)
