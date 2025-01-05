---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-02
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---

## 探索 Minimal APIs 及其优点

这一章我们将介绍一些与 .NET 6.0 中的 `Minimal APIs` 相关的基本话题，展示它与我们在以前版本的 .NET 中编写的基于控制器的
Web API 有何不同。我们还将尝试剖析这种编写 API 的新方法的利弊。

在本章中，我们将介绍以下主题：

- 路由
- 参数绑定
- 探索响应
- 控制序列化
- 架构 `Minimal APIs` 项目

## 技术要求

要照本章中的说明进行操作，您需要创建一个 ASP.NET Core 6.0 Web API 应用程序。 您可以使用以下选项之一：

- 选项 1：单击 Visual Studio 2022 的 **File** 菜单中的 **New | Project**  命令；然后，选择 **ASP.NET Core Web API** 模板。
  在向导中选择名称和工作目录，并确保在下一步中取消选中 **Use controllers（取消选中以使用 `Minimal APIs`**）选项。
- 选项 2：打开您的控制台、shell 或 Bash 终端，然后切换到您的工作目录。
  使用以下命令创建一个新的 Web API 应用程序：
  ```bash
  $ dotnet new webapi -minimal -o Chapter02
  ```

现在，通过双击项目文件在 Visual Studio 中打开项目，或者在 Visual Studio Code 中，通过在已打开的控制台中键入以下命令来打开项目：
```bash
$ cd Chapter02
$ code .
```

最后，您可以安全地删除所有与 **WeatherForecast** 示例相关的代码，因为本章不需要它。

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，网址为
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6/tree/main/Chapter02](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6/tree/main/Chapter02)。

## 路由

根据 [https://docs.microsoft.com/aspnet/core/fundamentals/routing](https://docs.microsoft.com/aspnet/core/fundamentals/routing)
上提供的 Microsoft 官方文档，为路由给出了以下定义：

> 路由负责匹配传入的 HTTP 请求并将这些请求分派到应用程序的可执行端点，也就是是应用程序的可执行请求处理代码单元。<br/>
> 通常端点是在应用程序中定义并在应用程序启动时配置。 <br/>
> 端点匹配过程可以从请求的 URL 中提取值并提供这些值以进行请求处理。<br/>
> 使用应用程序中的端点信息，路由还能够生成映射到端点的 URL。

在基于 Controller 的 Web API 中，路由是通过 **Startup.cs** 中的 `UseEndpoints()` 方法定义的，或者在操作方法上使用数据注解，
例如 **Route**、**HttpGet**、**HttpPost**、**HttpPut**、**HttpPatch** 和 **HttpDelete**。
正如第 1 章 `Minimal APIs` 简介中提到的，我们使用 **WebApplication** 对象的 **Map*** 方法来定义路由模式。 如下代码所示：

```csharp
app.MapGet("/hello-get", () => "[GET] Hello World!");
app.MapPost("/hello-post", () => "[POST] Hello World!");
app.MapPut("/hello-put", () => "[PUT] Hello World!");
app.MapDelete("/hello-delete", () => "[DELETE] Hello World!");
```

在这个代码片段中，我们定义了四个端点，对应不同的路由和方法。当然，也可以使用相同的路由模式来匹配不同的 HTTP 动词。

> **提示** <br/>
> 一旦我们向我们的应用程序添加端点（例如，使用 **MapGet()**），
> **UseRouting()** 就会自动添加到中间件管道的开头，**UseEndpoints()** 会自动添加到管道的末尾。

正如此处所示，ASP.NET Core 6.0 为最常见的 HTTP 动词提供了 **Map*** 方法。
如果我们需要使用其他动词，可以使用通用的 **MapMethods** 方法：

```csharp
app.MapMethods("/hello-patch", new[] { HttpMethods.Patch },  () => 
    "[PATCH] Hello World!");
app.MapMethods("/hello-head", new[] { HttpMethods.Head },  () => 
    "[HEAD] Hello World!");
app.MapMethods("/hello-options", new[] {  HttpMethods.Options }, () => 
    "[OPTIONS] Hello World!");
```

在接下来的部分中，我们将详细说明路由是如何有效工作以及我们该如何控制其行为。

### 路由处理程序

当路由 URL 匹配时执行的方法（根据参数和约束，如以下各节所述）称为**路由处理程序**。
路由处理程序可以是 lambda 表达式、本地函数、实例方法或静态方法，无论是同步的还是异步的都可以：

- 首先这是一个 lambda 表达式的例子（内联或使用变量）：

```csharp
app.MapGet("/hello-inline", () => "[INLINE LAMBDA] Hello World!");

var handler = () => "[LAMBDA VARIABLE] Hello World!";
app.MapGet("/hello", handler);
```

- 再来一个本地函数的例子：

```csharp
string Hello() => "[LOCAL FUNCTION] Hello World!";
app.MapGet("/hello", Hello);
```

- 接下来看到的是实例方法的例子：

```csharp
var handler = new HelloHandler();
app.MapGet("/hello", handler.Hello);

class HelloHandler
{
    public string Hello() => "[INSTANCE METHOD] Hello World!";
}
```

- 最后是一个静态函数的例子：

```csharp
app.MapGet("/hello", HelloHandler.Hello);

class HelloHandler
{
    public static string Hello() => "[STATIC METHOD] Hello World!";
}
```

### 路由参数

与之前版本的 .NET 一样，我们可以创建带有参数的路由模式，这些参数将由处理程序自动捕获：

```csharp
app.MapGet("/users/{username}/products/{productId}",
    (string username, int productId) => 
        $"The Username is {username} and the product Id  is {productId}");
```

路由可以包含任意数量的参数。 当对该路由发出请求时，参数将被捕获、解析并作为参数传递给相应的处理程序。
这样，处理程序将始终接收类型化参数（在前面的示例中，我们确定 username 是**字符串**，productId 是 **int**）。

如果无法将路由值转换为指定的类型，则会抛出 `BadHttpRequestException` 类型的异常，并且 API 会响应 **400 Bad Request** 消息。

### 路由约束

路由约束用于限制路由参数的有效类型。
典型的约束允许我们指定参数必须是数字、字符串或 GUID。
要指定路由约束，我们只需在参数名称后添加一个冒号，然后指定约束名称：

```csharp
app.MapGet("/users/{id:int}", (int id) => $"The user Id is {id}");
app.MapGet("/users/{id:guid}", (Guid id) => $"The user Guid is {id}");
```

`Minimal APIs` 支持所有在以前版本的 ASP.NET Core 中已经可用的路由约束。
您可以在以下链接中找到完整的路线限制列表：<br/>
[https://docs.microsoft.com/aspnet/core/fundamentals/routing#route-constraint-reference](https://docs.microsoft.com/aspnet/core/fundamentals/routing#route-constraint-reference)

如果根据约束，没有路由匹配指定的路径，此时不会有异常抛出，而是会收到一条 `404 Not Found` 的响应消息，
因为事实上，如果约束不适合，则代表路由本身是不可访问的。如下表所示，这些情况下，都会收到 404 响应：

| Route           | Path        |
|-----------------|-------------|
| users/{id:int}  | users/marco |
| users/{id:guid} | users/42    |

默认情况下，处理程序中未声明为路由约束的所有其他参数都体现在查询字符串中。如下示例：

```csharp
// Matches hello?name=Marco
app.MapGet("/hello", (string name) => $"Hello, {name}!");
```

下一节将探讨参数绑定，其中，我们将更深入地了解如何使用绑定来进一步自定义路由，
具体方法是指定在何处搜索路由参数、如何更改其名称以及如何使用可选路由参数。

## 参数绑定

参数绑定是将请求数据（即 URL 路径、查询字符串或正文）转换为路由处理程序可以使用的强类型参数的过程。
ASP.NET Core `Minimal APIs` 支持以下绑定源：

- 路由值
- 查询字符串
- Http 标头
- Http 正文（仅 JSON，默认支持的唯一格式）
- 服务提供者（依赖注入）

我们将在 [第 4 章 实施依赖注入](/books/master-minimal-apis/chapter-04) 中详细讨论依赖注入。

正如我们将在本章后面看到的那样，如有必要，我们可以自定义为特定输入执行绑定的方式。
不幸的是，在当前版本中，`Minimal APIs` 本身不支持从 **Form** 进行绑定，这意味着，也不支持 **IFormFile**。
为了更好地理解参数绑定的工作原理，让我们看一下以下 API 的例子：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<PeopleService>();
var app = builder.Build();
app.MapPut("/people/{id:int}", 
    (int id, bool notify, Person person, PeopleService peopleService) => { });
app.Run();

public class PeopleService { }

public record class Person(string FirstName, string LastName);
```

传递给处理程序的参数通过以下方式解析:

| 参数            | 来源               |
|---------------|------------------|
| id            | 路由               |
| Notify        | 查询字符串（大小写敏感）     |
| Person        | Http 正文（JSON 格式） |
| peopleService | 服务提供者            |

正如我们所看到的，ASP.NET Core 能够根据路由模式和参数本身的类型，自动推断出在哪里搜索用于绑定的参数。
例如，请求正文中应包含 Person 类等复杂类型。

如果需要，就像在以前的 ASP.NET Core 版本中一样，我们可以使用 **特性（Attribute）** 来显式指定参数的绑定位置，并且可以选择为它们使用不同的名称。
请参阅以下端点：

```csharp
app.MapGet("/search", string q) => { });
```

可以使用 `/search?q=text` 调用 API。 但是，使用 **q** 作为参数的名称并不是一个好主意，因为它的含义不是不言自明的。
因此，我们可以使用 **FromQueryAttribute** 修改处理程序：

```csharp
app.MapGet("/search", ([FromQuery(Name = "q")] string searchText) => { });
```

> **提示**<br/>
> 根据标准，**GET**、**DELETE**、**HEAD** 和 **OPTIONS** 等 HTTP 动词永远不应该有正文内容。
> 尽管如此，你依然可以强制使用它，此时需要显式地将 [FromBody] 属性添加到处理程序参数；
> 否则，您将收到 **InvalidOperationException** 错误。
>
> **但是，请记住，这是一种不好的做法。**

默认情况下，路由处理程序中的所有参数都是必要的。
因此，如果根据路由，ASP.NET Core 找到了一个有效的路由，但没有提供所有必需的参数，我们将得到一个错误。
例如，让我们看看下面的方法：

```csharp
app.MapGet("/people", (int pageIndex, int itemsPerPage) => { });
```

如果我们在没有 **pageIndex** 或 **itemsPerPage** 查询字符串值的情况下调用端点，
我们将收到 `BadHttpRequestException` 错误，并且响应将是 **400 Bad Request**。

要使参数可选，我们只需将它们声明为可为空（nullable）或提供默认值。 默认值是最常见的一种做法。
但是，如果我们采用默认着急这种解决方案，我们就不能在处理程序使用 lambda 表达式。 我们需要另一种方法，例如，局部函数：

```csharp
// 注释部分无法通过编译
//app.MapGet("/people", (int pageIndex = 0, int itemsPerPage = 50) => { });
string SearchMethod(int pageIndex = 0, int itemsPerPage = 50) => 
    $"Sample result for page {pageIndex} getting {itemsPerPage} elements";
app.MapGet("/people", SearchMethod);
```

如上示例情况，这里处理的是一个查询字符串，但相同的规则适用于所有绑定源。

请记住，如果我们使用 **可空引用类型**（在 .NET 6.0 项目中默认启用）并且我们有一个字符串参数，
例如，它可能为**空（null）**，我们需要将其声明为**可空（nullable）** - 否则，我们将收到 `BadHttpRequestException` 错误。
以下示例如何正确地将 **orderBy** 查询字符串参数定义为可选：

```csharp
app.MapGet("/people", (string? orderBy) => $"Results ordered by {orderBy}");
```

### 特殊绑定

在基于控制器的 Web API 中，继承自 **Microsoft.AspNetCore.Mvc.ControllerBase** 的控制器可以访问一些允许它获取请求和响应上下文的属性：
**HttpContext**、**Request**、**Response** 和 **User**。 在 `Minimal APIs` 中，我们没有基类，但我们仍然可以访问此信息，
因为它被视为一个特殊的绑定，对任何处理程序始终可用：

```csharp
app.MapGet("/products", 
    (HttpContext context, HttpRequest req, HttpResponse res, ClaimsPrincipal user) => { });
```

> **提示**<br/>
> 我们依然还可以使用 **IHttpContextAccessor** 接口访问所有这些对象，就像我们在以前的 ASP.NET Core 版本中所做的那样。


### 自定义绑定

在某些情况下，参数绑定的默认工作方式不足以满足我们的目的。 
在 `Minimal APIs` 中，我们不支持 **IModelBinderProvider** 和 **IModelBinder** 接口，但我们有两个替代方案来实现自定义模型绑定。

> **重要提示**<br/>
> 
> 基于控制器的项目中的 **IModelBinderProvider** 和 **IModelBinder** 接口允许我们定义请求数据和应用程序模型之间的映射。 
> ASP.NET Core 提供的默认模型绑定器支持大多数常见数据类型，但如果有必要，我们可以通过创建自己的提供程序来扩展系统。 
> 我们可以通过以下链接找到更多信息 <br/>
> [https://docs.microsoft.com/aspnet/core/mvc/advanced/custom-model-binding](https://docs.microsoft.com/aspnet/core/mvc/advanced/custom-model-binding)


如果我们想将来自路由、查询字符串或标头的参数绑定到自定义类型，我们可以向该类型添加静态 **TryParse** 方法：

```csharp
// GET /navigate?location=43.8427,7.8527
app.MapGet("/navigate", (Location location) => 
    $"Location: {location.Latitude}, {location.Longitude}");

public class Location
{
    public double Latitude { get; set; }
    public double Longitude { get; set; }

    public static bool TryParse(string? value, IFormatProvider? provider,
        out Location? location)
    {
          if (!string.IsNullOrWhiteSpace(value))
          {
               var values = value.Split(',', StringSplitOptions.RemoveEmptyEntries);
               if (values.Length == 2 
                   && double.TryParse(values[0],
                       NumberStyles.AllowDecimalPoint,
                       CultureInfo.InvariantCulture,
                       out var latitude)
                   && double.TryParse(values[1],
                       NumberStyles.AllowDecimalPoint,
                       CultureInfo.InvariantCulture,
                       out var longitude))
               {
                       location = new Location 
                       {
                           Latitude = latitude,
                           Longitude = longitude
                       };
                       return true;
               }
          }
          location = null;
          return false;
    }
}
```

在 **TryParse** 方法中，我们可以尝试拆分输入参数并检查它是否包含两个十进制值：在这种情况下，我们解析数字以构建 **Location** 对象并返回 `true`。
否则，我们返回 `false`，因为无法初始化 **Location** 对象。

> **重要提示**<br/>
>
> 当 `Minimal APIs` 发现一个类型包含静态 **TryParse** 方法时，即使它是一个复杂类型，它也会基于路由模板，假定它是在路由或查询字符串中传递的。 
> 我们可以使用 **[FromHeader]** 属性来更改绑定源到 Http 标头。 
> 
> 在任何情况下，都不会为请求的正文调用 **TryParse**。


如果我们需要完全控制绑定的执行方式，我们可以在类型上实现静态 **BindAsync** 方法。 这不是一个很常见的解决方案，但在某些情况下，它可能很有用：

```csharp
// POST /navigate?lat=43.8427&lon=7.8527
app.MapPost("/navigate", (Location location) => 
   $"Location: {location.Latitude}, {location.Longitude}");

public class Location
{
    // ...
    public static ValueTask<Location?> BindAsync(HttpContext context,
        ParameterInfo parameter)
    {
        if (double.TryParse(context.Request.Query["lat"], 
                NumberStyles.AllowDecimalPoint, 
                CultureInfo.InvariantCulture, 
                out var latitude) &&
            double.TryParse(context.Request.Query["lon"], 
                NumberStyles.AllowDecimalPoint, 
                CultureInfo.InvariantCulture, 
                out var longitude))
        {
                var location = new Location 
                {
                    Latitude = latitude,
                    Longitude = longitude
                };
                return ValueTask.FromResult<Location?>(location);
        }
        return ValueTask.FromResult<Location?>(null);
    }
}
```

正如我们所见，**BindAsync** 方法将整个 **HttpContext** 作为参数，因此我们可以读取创建传递给路由处理程序的实际 **Location** 对象所需的所有信息。
在此示例中，我们读取了两个查询字符串参数（`lat` 和 `lon`），但是（在 **POST**、**PUT** 或 **PATCH** 方法的情况下）我们还可以读取请求的整个主体并手动解析其内容。 
这可能很有用，例如，如果我们需要处理格式不是 JSON 的请求（如前所述，默认情况下唯一支持的格式）。

如果 **BindAsync** 方法返回 `null`，而相应的路由处理程序参数不能采用此值（如前例所示），我们将收到 **HttpBadRequestException** 错误。
像往常一样，将包含在 **400 Bad Request** 响应中。

> **重要提示**<br/>
>
> 我们不应该在同一类型同时定义 **TryParse** 和 **BindAsync** 方法； 如果两者都存在，则 **BindAsync** 始终具有优先权（即永远不会调用 **TryParse**）。

现在我们已经了解了参数绑定并了解了如何使用它和自定义其行为，让我们看看如何在 `Minimal APIs` 中处理响应。

## 探索响应

与基于控制器的项目一样，使用 `Minimal APIs` 的路由处理程序，我们可以直接返回一个字符串或一个类（无论同步或异步）：

- 当返回一个字符串（如上一节的示例），框架将字符串直接写入响应，将其内容类型设置为 **text/plain** 并将状态代码设置为 **200 OK**
- 当返回一个类实例，这个实例对象会被序列化为 JSON 格式内容并设置响应类型为 **application/json** 以及 **200 OK** 的状态代码

但是，在实际应用程序中，我们通常需要控制响应类型和状态代码。 
在这种情况下，我们可以使用静态 **Results** 类，它允许我们返回 `IResult` 接口的一个实例，它在 `Minimal APIs` 中就像 `IActionResult` 对控制器所做的那样。 
例如，我们可以使用它来返回 **201 Created** 响应而不是 **400 Bad Request** 或 **404 Not Found** 消息。 让我们看一些例子：

```csharp
app.MapGet("/ok", () => Results.Ok(new Person("Donald", "Duck")));
app.MapGet("/notfound", () => Results.NotFound());
app.MapPost("/badrequest", () =>
{
    // Creates a 400 response with a JSON body.
    return Results.BadRequest(new { ErrorMessage = "Unable to complete the request" });
});
app.MapGet("/download", (string fileName) => Results.File(fileName));

record class Person(string FirstName, string LastName);
```

**Results** 类的每个方法负责设置与方法本身的含义相对应的响应类型和状态代码（例如，`Results.NotFound()` 方法返回 **404 Not Found** 响应）。
请注意，即使我们通常需要在 **200 OK** 响应（使用 `Results.Ok()`）的情况下返回一个对象，它也不是唯一允许这样做的方法。 
许多其他方法允许我们包含自定义响应； 在所有这些情况下，响应类型都将设置为 `application/json`，对象将自动进行 JSON 序列化。

当前版本的 `Minimal APIs` 不支持内容协商 （Content Negotiation）。 
在使用 `Results.Bytes()`、`Results.Stream()` 和 `Results.File()` 获取文件时，或者在使用 `Results.Text()` 和 `Results.Content()` 时，只有少数方法允许我们显式设置内容类型 。
在所有其他情况下，当我们处理复杂对象时，响应将采用 JSON 格式。
这是一个精确的设计选择，因为大多数开发人员很少需要支持其他媒体类型。
通过仅支持 JSON 而不执行内容协商，`Minimal APIs` 可以非常高效。

但是，这种方法在所有情况下都不够。 在某些情况下，我们可能需要创建自定义响应类型，例如，如果我们想要返回 HTML 或 XML 响应而不是标准 JSON。 
我们可以手动使用 `Results.Content()` 方法（它允许我们将内容指定为具有特定内容类型的简单字符串），
但是，如果我们有此要求，最好实现自定义 **IResult** 类型，以便 该解决方案可以重复使用。

例如，假设我们想要序列化 XML 而不是 JSON 中的对象。 然后我们可以定义一个实现 **IResult** 接口的 **XmlResult** 类：

```csharp
public class XmlResult : IResult
{
   private readonly object value;
   
   public XmlResult(object value)
   {
       this.value = value;
   }
   
   public Task ExecuteAsync(HttpContext httpContext)
   {
       using var writer = new StringWriter();
          
       var serializer = new XmlSerializer(value.GetType());
       serializer.Serialize(writer, value);
       var xml = writer.ToString();
       httpContext.Response.ContentType = MediaTypeNames.Application.Xml;
       httpContext.Response.ContentLength = Encoding.UTF8.GetByteCount(xml);
       return httpContext.Response.WriteAsync(xml);
   }
}
```

**IResult** 接口要求我们实现 **ExecuteAsync** 方法，该方法接收当前 **HttpContext** 作为参数。 
我们使用 **XmlSerializer** 类序列化该值，然后将其写入响应，指定正确的响应类型。

```csharp
public static class ResultExtensions
{
    public static IResult Xml(this IResultExtensions resultExtensions,
        object value) => new XmlResult(value);
}
```

通过这种方式，我们在 **Results.Extensions** 属性上有一个新的 **Xml** 方法可用：

```csharp
app.MapGet("/xml", () => Results.Extensions.Xml(new City { Name = "Taggia" }));

public record class City
{
    public string? Name { get; init; }
}
```

这种方法的好处是我们可以在需要处理 XML 的任何地方重用它，而不必手动处理序列化和响应类型（我们应该使用 `Result.Content()` 方法来代替）。

> **提示**
>
> 如果我们想要执行内容验证，我们需要手动检查 **HttpRequest** 对象的 **Accept** 标头，我们可以将其传递给我们的处理程序，然后相应地创建正确的响应。

在分析了如何以 `Minimal APIs` 正确处理响应之后，我们将在下一节中了解如何控制数据序列化和反序列化的方式。


## 控制序列化

如前几节所述，`Minimal APIs` 仅提供对 JSON 格式的内置支持。 特别是，该框架使用 **System.Text.Json** 进行序列化和反序列化。
在基于控制器的 API 中，我们可以更改此默认值并改用 JSON.NET。 这在使用最少的 API 时是不可能的：我们根本无法替换序列化程序。

内置序列化器使用以下选项：

- 序列化不区分大小写的属性名称
- 驼峰命名规则
- 支持引用数字（数字属性的 JSON 字符串）

> **提示**
>
> 我们可以在以下链接中找到有关 **System.Text.Json** 命名空间及其提供的所有 API 的更多信息：<br/>
> [https://docs.microsoft.com/dotnet/api/system.text.json](https://docs.microsoft.com/dotnet/api/system.text.json)


在基于控制器的API中，我们可以通过在 **AddControllers()** 之后流畅地调用 **AddJsonOptions()** 来自定义这些设置。 
在 `Minimal APIs` 中，我们无法使用这种方法，因为我们根本没有控制器，因此我们需要明确调用 **JsonOptions** 的配置方法。 
因此，让我们考虑一下这个处理程序：

```csharp
app.MapGet("/product", () =>
{
    var product = new Product("Apple", null, 0.42, 6);
    return Results.Ok(product); 
});

public record class Product(string Name, 
    string? Description, 
    double UnitPrice, 
    int Quantity)
{
    public double TotalPrice => UnitPrice * Quantity;
}
```

使用默认的JSON选项，我们得到此结果：
```json
{
    "name": "Apple",
    "description": null,
    "unitPrice": 0.42,
    "quantity": 6,
    "totalPrice": 2.52
}
```

现在，让我们配置 **JsonOptions**：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.Configure<Microsoft.AspNetCore.Http.Json.JsonOptions>(options =>
{
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    options.SerializerOptions.IgnoreReadOnlyProperties = true;
});
```
再次调用 **/product**，我们现在将获得以下内容：
```json
{
    "name": "Apple",
    "unitPrice": 0.42,
    "quantity": 6
}
```
正如预期的那样，description 属性尚未被序列化，因为它是无效的，以及 totalPrice ，因为它是只读属性，因此也未包含在响应中。

**JsonOptions** 的另一个典型用例是，当我们要添加将自动应用于每个序列化或避难所化的转换器时，
例如，**JsonStringEnumConverter** 将枚举值转换为字符串或从字符串中转换。

> **重要提示**<br/>
>
> 请注意，`Minimal APIs` 使用的 **JsonOptions** 类是 **Microsoft.AspNetCore.Http.Json** 名称空间中的一个。
> 不要将其与 **Microsoft.AspNetCore.Mvc** 名称空间中定义的一个混淆； 
> 对象的名称是相同的，但是后者仅适用于控制器，因此如果在 `Minimal APIs` 项目中设置，则没有效果。

由于只有JSON的支持，如果我们不明确添加对其他格式的支持，如先前的部分所述（例如，使用自定义类型上的 **Bindasync** 方法），
`Minimal APIs` 将自动对该验证执行某些验证。 Http内容绑定的来源并处理以下方案：

| 问题                                    | 响应代码 |
|---------------------------------------|------|
| 内容类型（ContentType）不是`application/json` | 415  |
| 无法将内容作为JSON读取                         | 400  |

这些情况下，由于 Http 内容验证失败，我们的路由处理程序将永远不会被调用，并且我们将直接返回如上的表中提供的响应状态代码。

现在，我们已经涵盖了开始开发 `Minimal APIs` 所需的所有主要内容。 
但是，还有另一件重要的事情要谈论：以最佳实践来设计一个真正的项目以避免在组织架构上犯错误。

## 架构 `Minimal APIs` 项目

到目前为止，我们已经直接在 **Program.cs** 文件中编写了路由处理程序。 
这是一个完美支持的场景：使用 `Minimal APIs`，我们可以在这个文件中编写所有代码。 
事实上，几乎所有示例都采用了此解决方案。 
然而，虽然这是可行的，但我们可以很容易地想象这种方法将导致非结构化的项目结构，后续将无法维护。
如果我们只有很少的端点，那还好 —— 否则，最好将我们的处理程序组织在单独的文件中。

假设我们在 **Program.cs** 文件中有以下代码，因为我们必须处理 **CRUD** 操作：

```csharp
app.MapGet("/api/people", (PeopleService peopleService) => { });
app.MapGet("/api/people/{id:guid}", 
    (Guid id, PeopleService peopleService) => { });
app.MapPost("/api/people", (Person Person, PeopleService people) => { });
app.MapPut("/api/people/{id:guid}", 
    (Guid id, Person person, PeopleService people) => { });
app.MapDelete("/api/people/{id:guid}", (Guid id, PeopleService people) => { });
```

很容易想象，如果我们在这里拥有所有实现（即使我们使用 **PeopleService** 来提取业务逻辑），这个文件很容易爆炸。
因此，在实际场景中，内联 lambda 方法并不是最佳实践。 我们应该使用我们在路由部分介绍的其他方法来定义处理程序。 
因此，创建一个外部类来保存所有路由处理程序是个好主意：

```csharp
public class PeopleHandler
{
   public static void MapEndpoints(IEndpointRouteBuilder app)
   {
       app.MapGet("/api/people", GetList);
       app.MapGet("/api/people/{id:guid}", Get);
       app.MapPost("/api/people", Insert);
       app.MapPut("/api/people/{id:guid}", Update);
       app.MapDelete("/api/people/{id:guid}", Delete);
   }
         
   private static IResult GetList(PeopleService peopleService) 
   { /* ... */ }
   private static IResult Get(Guid id, PeopleService peopleService) 
   { /* ... */ }
   private static IResult Insert(Person person, PeopleService people) 
   { /* ... */ }
   private static IResult Update(Guid id, Person person, PeopleService people) 
   { /* ... */ }
   private static IResult Delete(Guid id) 
   { /* ... */ }
}
```

我们已将所有端点定义分组在 **PeopleHandler.MapEndpoints** 静态方法中，该方法将 **IEndpointRouteBuilder** 接口作为参数，该接口又由 **WebApplication** 类实现。
然后，我们没有使用 lambda 表达式，而是为每个处理程序创建了单独的方法，这样代码就干净多了。 
这样，要在我们的 `Minimal APIs` 中注册所有这些处理程序，我们只需要在 **Program.cs** 中添加以下代码：

```csharp
var builder = WebApplication.CreateBuilder(args);
// ..
var app = builder.Build();
// ..
PeopleHandler.MapEndpoints(app);
app.Run();
```

### 更进一步

刚刚展示的方法使我们能够更好地组织一个 `Minimal APIs` 项目，但仍然需要我们为要定义的每个处理程序明确地向 **Program.cs** 添加一行。
使用一个接口和一些反射，我们可以创建一个简单且可重用的解决方案，以使用 `Minimal APIs` 来简化我们的工作。

因此，让我们从定义以下接口开始：

```csharp
public interface IEndpointRouteHandler
{
   public void MapEndpoints(IEndpointRouteBuilder app);
}
```

顾名思义，我们需要让我们所有的处理程序（与之前的 **PeopleHandler** 一样）实现它：

```csharp
public class PeopleHandler : IEndpointRouteHandler
{
    public void MapEndpoints(IEndpointRouteBuilder app)
    {
        // ...
    }
    // ...
}

```

> **提示**
> 
> **MapEndpoints** 方法不再是静态的，因为现在它是 IEndpointRouteHandler 接口的实现。

现在我们需要一个新的扩展方法，它使用反射扫描程序集以查找实现此接口的所有类并自动调用它们的 **MapEndpoints** 方法：

```csharp
public static class IEndpointRouteBuilderExtensions
{
    public static void MapEndpoints(this IEndpointRouteBuilder app,
        Assembly assembly)
    {
        var endpointRouteHandlerInterfaceType = typeof(IEndpointRouteHandler);
        var endpointRouteHandlerTypes = assembly.GetTypes().Where(t => t.IsClass 
            && !t.IsAbstract 
            && !t.IsGenericType
            && t.GetConstructor(Type.EmptyTypes) != null
            && endpointRouteHandlerInterfaceType.IsAssignableFrom(t));
        foreach (var endpointRouteHandlerType in endpointRouteHandlerTypes)
        {
            var instantiatedType = (IEndpointRouteHandler) Activator.CreateInstance(
                endpointRouteHandlerType)!;
            instantiatedType.MapEndpoints(app);
        }
    }
}
```

> **提示**
>
> 如果您想更详细地了解反射及其在 .NET 中的工作方式，可以从浏览以下页面开始：<br/>
> [https://docs.microsoft.com/dotnet/csharp/programming-guide/concepts/reflection](https://docs.microsoft.com/dotnet/csharp/programming-guide/concepts/reflection)

有了所有这些零件，最后要做的是在 `run()` 方法之前调用 **Program.cs** 文件中的扩展方法：
```csharp
app.MapEndpoints(Assembly.GetExecutingAssembly());
app.Run();
```

这样，当我们添加新处理程序时，我们只需要创建一个实现 **IEndPointRouteHandler** 接口的新类即可。 
**Program.cs** 中不需要其他更改将新的端点添加到路由引擎中。

在外部文件中编写路线处理程序，并考虑一种自动化端点注册的方法，使得 **Program.cs** 不会为每个功能添加而增长，
这才是构建 `Minimal APIs` 项目的正确方法。

## 总结

ASP.NET Core `Minimal APIs` 代表了一种在 .NET 世界中编写 HTTP API 的新方法。
在本章中，我们介绍了开始开发 `Minimal APIs` 所需的所有基本技能、如何有效地处理它们，以及在决定遵循此架构时要考虑的最佳实践。

在下一章中，我们将重点介绍一些高级概念，例如使用 Swagger 记录 API、定义正确的错误处理系统以及将 `Minimal APIs` 与单页应用程序集成。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
