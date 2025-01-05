---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-07
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---


## 与数据访问层的集成

在本章中，我们将学习在.NET 6.0 中向 Minimal API 添加数据访问层的一些基本方法。
我们将看到如何使用本书前面章节中涵盖的一些主题，通过 `Entity Framework（EF）`和 `Dapper` 访问数据。
这是访问数据库的两种方式。

本章将涵盖以下主题：
* 使用 Entity Framework
* 使用 Dapper

到本章结束时，你将能够在 Minimal API 项目中从头开始使用 EF，并出于相同目的使用 Dapper。
你还将能够判断在项目中哪种方法更合适。

## 技术要求

要跟随本章内容学习，你需要创建一个 ASP.NET Core 6.0 Web API 应用程序。你可以使用以下两种选项之一：

* 点击 Visual Studio 2022 的 “**文件**” 菜单中的 “**新建项目**” 选项，
然后选择 “**ASP.NET Core Web API**” 模板，在向导中选择名称和工作目录，
并确保在下一步中取消选中 “**使用控制器**” 选项。
* 打开控制台、shell 或 Bash 终端，并切换到工作目录。使用以下命令创建一个新的 Web API 应用程序：<br/>
`dotnet new webapi -minimal -o Chapter07`

现在，在 Visual Studio 中双击项目文件打开项目，
或者在 Visual Studio Code 中，在已经打开的控制台中输入以下命令：
```shell
cd Chapter07
code
```

最后，你可以安全地删除与 `WeatherForecast` 示例相关的所有代码，因为本章不需要它。
本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，地址为:
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter07](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter07)

## 使用 Entity Framework

可以肯定地说，如果我们正在构建一个 API，很可能会与数据进行交互。

此外，在应用程序重新启动或其他事件（如应用程序的新部署）之后，这些数据很可能需要持久化。
在 .NET 应用程序中，有许多持久化数据的选项，但 EF 是许多场景中最用户友好和常见的解决方案。

**Entity Framework Core（EF Core）** 是一个可扩展的、开源的、跨平台的数据访问库，用于 .NET 应用程序。
它使开发人员能够直接使用 .NET 对象与数据库进行交互，并且在大多数情况下，无需知道如何直接在数据库中编写数据访问代码。

此外，EF Core 支持许多数据库，包括 _SQLite_、_MySQL_、_Oracle_、_Microsoft SQL Server_ 和 _PostgreSQL_。

它还支持一个内存数据库，这有助于为我们的应用程序编写测试或简化开发周期，因为不需要运行实际的数据库。

在下一节中，我们将看到如何设置一个使用 EF 的项目及其主要功能。

### 设置项目

从项目根目录创建一个 `Icecream.cs` 类，并赋予它以下内容：
```csharp
namespace Chapter07.Models;

public class Icecream
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public string? Description { get; set; }
}
```

`Icecream` 类是我们项目中代表冰淇淋的对象。
这个类应该被称为数据模型，我们将在本章的后续部分使用它来映射到数据库表。

现在是时候向项目添加 EF Core NuGet 引用了。

你可以使用以下方法之一：
* 在一个新的终端窗口中，输入以下代码添加 EF Core InMemory 包：<br/>
`dotnet add package Microsoft.EntityFrameworkCore.InMemory`
* 如果你想使用 Visual Studio 2022 添加引用，右键单击 “**依赖项**”，然后选择 “**管理 NuGet 包**”。
搜索 `Microsoft.EntityFrameworkCore.InMemory` 并安装该包。

在下一节中，我们将把 EF Core 添加到我们的项目中。

### 将 EF Core 添加到项目中

为了将冰淇淋对象存储在数据库中，我们需要在项目中设置 EF Core。

要设置一个内存数据库，在 `Program.cs` 文件底部添加以下代码：

```csharp
class IcecreamDb : DbContext
{
    public IcecreamDb(DbContextOptions options) : base(options) { }

    public DbSet<Icecream> Icecreams { get; set; } = null!;
}
```

`DbContext` 对象表示与数据库的连接，用于保存和查询数据库中的实体实例。

`DbSet` 表示实体的实例，它们将在数据库中转换为实际的表。

在这种情况下，我们的数据库中将只有一个表，称为 `Icecreams`。

在 `Program.cs` 中，在 `builder` 初始化之后，添加以下代码：

```csharp
builder.Services.AddDbContext<IcecreamDb>(options => 
    options.UseInMemoryDatabase("icecreams"));
```

现在我们准备添加一些 API 端点来开始与数据库进行交互。

### 向项目添加端点

让我们添加代码在冰淇淋列表中创建一个新条目。在 `Program.cs` 中，在 `app.Run()` 行之前添加以下代码：

```csharp
app.MapPost("/icecreams", async (IcecreamDb db, Icecream icecream) =>
{
    await db.Icecreams.AddAsync(icecream);
    await db.SaveChangesAsync();
    return Results.Created($"/icecreams/{icecream.Id}", icecream);
});
```

`MapPost` 函数的第一个参数是 `DbContext`。
默认情况下，Minimal API 架构使用依赖注入来共享 `DbContext` 的实例。

>
> 依赖注入： 
> 
> 如果你想了解更多关于依赖注入的信息，请转到 [第 4 章 “最小 API 项目中的依赖注入”](/books/master-minimal-apis/chapter-04)。

为了将一个条目保存到数据库中，我们直接使用代表对象的实体的 `AddAsync` 方法。

为了在数据库中持久化新条目，我们需要调用 `SaveChangesAsync()` 方法，
该方法负责在最后一次调用 `SaveChangesAsync()` 之前保存对数据库所做的所有更改。

以非常相似的方式，我们可以添加端点来检索冰淇淋数据库中的所有条目。

在添加冰淇淋的代码之后，我们可以添加以下代码：

```csharp
app.MapGet("/icecreams", async (IcecreamDb db) => 
    await db.Icecreams.ToListAsync());
```

同样，在这种情况下，`DbContext` 作为参数可用，我们可以直接从 `DbContext` 中的实体检索数据库中的所有条目。

通过 `ToListAsync()` 方法，应用程序加载数据库中的所有实体并将它们作为端点结果发送回来。

确保你已经保存了项目中的所有更改并运行应用程序。
一个新的浏览器窗口将打开，你可以导航到 `/swagger` URL：

![Figure 7.1 – Swagger browser window](/assets/images/master-minimal-apis/Figure_7.01_B17902.jpg)

选择 `POST /icecreams` 按钮，然后点击 “**Try it out**”。

用以下 JSON 替换请求体内容：
```json
{
    "id": 0,
    "name": "icecream 1",
    "description": "description 1"
}
```

点击 “**Execute**”：

![Figure 7.2 – Swagger response](/assets/images/master-minimal-apis/Figure_7.02_B17902.jpg)

现在我们的数据库中至少有一个条目，我们可以尝试其他端点来检索数据库中的所有条目。

向下滚动页面一点，选择 `GET /icecreams`，然后点击 “**Try it out”**，再点击 “**Execute**”。

你将在 “**Response Body**” 下看到一个列表。

让我们看看如何通过向端点添加其他 CRUD 操作来完成这个第一个演示：

1. 要通过 ID 获取一个条目，在之前创建的 `app.MapGet` 路由下添加以下代码：
```csharp
app.MapGet("/icecreams/{id}", async (IcecreamDb db, int id) => 
    await db.Icecreams.FindAsync(id));
```
2. 接下来，通过执行 POST 调用（如前一节所述）在数据库中添加一个条目。
3. 点击 `GET /icecreams/{id}`，然后点击 “**Try it out**”。
4. 在 `id` 参数字段中插入值 `1`，然后点击 “**Execute**”。
5. 你将在 “Response Body” 部分看到该条目。
6. 以下是 API 的响应示例：
```json
{
    "id": 1,
    "name": "icecream 1",
    "description": "description 1"
}
```
这是响应的样子：
![Figure 7.3 – Response result](/assets/images/master-minimal-apis/Figure_7.03_B17902.jpg)


要通过 ID 更新一个条目，我们可以创建一个新的 MapPut 端点，
带有两个参数：带有实体值的条目和我们要更新的数据库中旧实体的 ID。

代码应该如下所示：

```csharp
app.MapPut("/icecreams/{id}", async (IcecreamDb db, Icecream updateicecream) =>
{
    var icecream = await db.Icecreams.FindAsync(id);
    if (icecream is null) return Results.NotFound();
    icecream.Name = updateicecream.Name;
    icecream.Description = updateicecream.Description;
    await db.SaveChangesAsync();
    return Results.NoContent();
});
```

需要明确的是，首先我们需要使用参数中的 ID 在数据库中找到条目。
如果我们在数据库中找不到条目，向调用者返回一个 `Not Found` HTTP 状态是一个好的做法。

如果我们在数据库中找到实体，我们用新值更新实体，
并在发送回 HTTP 状态 `No Content` 之前保存数据库中的所有更改。

我们需要执行的最后一个 CRUD 操作是从数据库中删除一个条目。

这个操作与更新操作非常相似，因为首先我们需要在数据库中找到条目，然后我们可以尝试执行删除操作。

以下代码片段展示了如何使用 Minimal API 的正确 HTTP 动词实现删除操作：

```csharp
app.MapDelete("/icecreams/{id}", async (IcecreamDb db, int id) =>
{
    var icecream = await db.Icecreams.FindAsync(id);
    if (icecream is null)
    {
        return Results.NotFound();
    }
    db.Icecreams.Remove(icecream);
    await db.SaveChangesAsync();
    return Results.Ok();
});
```

在本节中，我们学习了如何在 Minimal API 项目中使用 EF。

我们看到了如何添加 NuGet 包来开始使用 EF，以及如何在 Minimal API.NET 6 项目中实现完整的 CRUD API 端点集。

在下一节中，我们将看到如何使用 `Dapper` 实现相同的项目，具有相同的逻辑，但将其作为访问数据的主要库。

## 使用 Dapper

`Dapper` 是一个 **对象关系映射器（ORM）**，或者更确切地说，是一个微型 ORM。
使用 `Dapper`，我们可以在 .NET 项目中直接编写 SQL 语句，就像在 SQL Server（或其他数据库）中一样。
在项目中使用 Dapper 的一个最大优点是性能，因为它不会将查询从 .NET 对象进行转换，
并且在应用程序和访问数据库的库之间不添加任何层。
它扩展了 `IDbConnection` 对象并提供了许多查询数据库的方法。
这意味着我们必须编写与数据库提供程序兼容的查询。

它支持同步和异步方法执行。以下是 `Dapper` 添加到 `IDbConnection` 接口的方法列表：
* `Execute`
* `Query`
* `QueryFirst`
* `QueryFirstOrDefault`
* `QuerySingle`
* `QuerySingleOrDefault`
* `QueryMultiple`

如前所述，它为所有这些方法提供了一个异步版本。你可以通过在方法名末尾添加 `Async` 关键字找到正确的方法。

在下一节中，我们将看到如何设置一个使用 `Dapper` 与 _SQL Server LocalDB_ 一起使用的项目。

### 设置项目

我们要做的第一件事是创建一个新的数据库。
你可以使用默认情况下随 Visual Studio 安装的 _SQL Server LocalDB_ 实例，
或者你环境中的其他 _SQL Server_ 实例。

你可以在数据库中执行以下脚本创建一个表并填充数据：
```sql
CREATE TABLE [dbo].[Icecreams](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [Name] [nvarchar](50) NOT NULL,
    [Description] [nvarchar](255) NOT NULL
)
GO
INSERT [dbo].[Icecreams] ([Name], [Description]) VALUES ('icecream 1', 'description 1')
INSERT [dbo].[Icecreams] ([Name], [Description]) VALUES ('icecream 2', 'description 2')
INSERT [dbo].[Icecreams] ([Name], [Description]) VALUES ('icecream 3', 'description 3')
```

一旦我们有了数据库，我们可以在 Visual Studio 终端中使用以下命令安装这些 NuGet 包：
```shell
Install-Package Dapper
Install-Package Microsoft.Data.SqlClient
```

现在我们可以继续添加代码来与数据库进行交互。在这个例子中，我们将使用 **存储库模式（repository）**。

### 创建存储库模式

在本节中，我们将创建一个简单的存储库模式，但我们将尽量使其简单，以便我们能够理解 `Dapper` 的主要功能：

1、在 `Program.cs` 文件中，添加一个简单的类来代表我们数据库中的实体：
```csharp
public class Icecream
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public string? Description { get; set; }
}
```

2、之后，修改 `appsettings.json` 文件，在文件末尾添加连接字符串：
```json
{
    "ConnectionStrings": {
        "SqlConnection": "Data Source=(localdb)\\MSSQLLocalDB;Initial Catalog=Chapter07;Integrated Security=True;Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;"
    }
}
```

3、在项目根目录创建一个新类 `DapperContext`，并赋予它以下代码：
```csharp
public class DapperContext
{
    private readonly IConfiguration _configuration;
    private readonly string _connectionString;

    public DapperContext(IConfiguration configuration)
    {
        _configuration = configuration;
        _connectionString = _configuration.GetConnectionString(
            "SqlConnection");
    }

    public IDbConnection CreateConnection() => 
       new SqlConnection(_connectionString);
}
```
我们通过依赖注入注入 `IConfiguration` 接口，以便从设置文件中检索连接字符串。

4、现在我们将创建存储库的接口和实现。为此，在 `Program.cs` 文件中添加以下代码：
```csharp
public interface IIcecreamsRepository
{
}

public class IcecreamsRepository : IIcecreamsRepository
{
    private readonly DapperContext _context;

    public IcecreamsRepository(DapperContext context)
    {
        _context = context;
    }
}
```
在接下来的部分中，我们将向接口和存储库实现添加一些代码。<br/>
最后，我们可以将上下文、接口及其实现注册为服务。

5、在 `Program.cs` 文件中，在 `builder` 初始化之后添加以下代码：
```csharp
builder.Services.AddSingleton<DapperContext>();
builder.Services.AddScoped<IIcecreamsRepository, IcecreamsRepository>();
```
现在我们准备实现第一个查询。

### 使用 Dapper 查询数据库

首先，修改 `IIcecreamsRepository` 接口，添加一个新方法：

```csharp
public Task<IEnumerable<Icecream>> GetIcecreams();
```

然后，在 `IcecreamsRepository` 类中实现这个方法：

```csharp
public async Task<IEnumerable<Icecream>> GetIcecreams()
{
    var query = "SELECT * FROM Icecreams";
    using (var connection = _context.CreateConnection())
    {
        var result = await connection.QueryAsync<Icecream>(query);
        return result.ToList();
    }
}
```

让我们试着理解这个方法中的所有步骤。我们创建了一个名为 `query` 的字符串，
在其中存储从数据库获取所有实体的 SQL 查询。

然后，在 `using` 语句内部，我们使用 `DapperContext` 创建连接。

一旦创建了连接，我们使用它调用 `QueryAsync` 方法，并将查询作为参数传递。
`Dapper` 会在数据库结果返回时自动将它们转换为 `IEnumerable<T>`。

以下是接口和我们第一个实现的最终代码：

```csharp
public interface IIcecreamsRepository
{
    public Task<IEnumerable<Icecream>> GetIcecreams()
}

public class IcecreamsRepository : IIcecreamsRepository
{
    private readonly DapperContext _context;

    public IcecreamsRepository(DapperContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Icecream>> GetIcecreams()
    {
        var query = "SELECT * FROM Icecreams";
        using (var connection = _context.CreateConnection())
        {
            var result = await connection.QueryAsync<Icecream>(query);
            return result.ToList();
        }
    }
}
```

在下一节中，我们将看到如何向数据库添加一个新实体以及如何使用 `ExecuteAsync` 方法运行查询。

### 使用 Dapper 在数据库中添加新实体

现在我们将管理向数据库添加一个新实体，以便在未来实现 API 的 POST 请求。

修改接口，添加一个名为 `CreateIcecream` 的新方法，带有一个 `Icecream` 类型的输入参数：

```csharp
public Task CreateIcecream(Icecream icecream);
```

现在我们必须在存储库类中实现这个方法：

```csharp
public async Task CreateIcecream(Icecream icecream)
{
    var query = @"INSERT INTO Icecreams (Name, Description) 
VALUES (@Name, @Description)";
    var parameters = new DynamicParameters();
    parameters.Add("Name", icecream.Name, DbType.String);
    parameters.Add("Description", icecream.Description, DbType.String);
    using (var connection = _context.CreateConnection())
    {
        await connection.ExecuteAsync(query, parameters);
    }
}
```

在这里，我们创建查询和一个动态参数对象，以便将所有值传递给数据库。

我们用方法参数中的 `Icecream` 对象的值填充参数。

我们使用 `Dapper` 上下文创建连接，然后使用 `ExecuteAsync` 方法执行 `INSERT` 语句。

这个方法返回一个整数值，表示数据库中受影响的行数。
在这种情况下，我们不使用这个信息，但如果需要，你可以将这个值作为方法的结果返回。

### 在端点中实现存储库

为了给我们的 Minimal API 做最后的完善，我们需要实现两个端点来管理我们存储库模式中的所有方法：

```csharp
app.MapPost("/icecreams", async (IIcecreamsRepository repository,
    Icecream icecream) =>
{
    await repository.CreateIcecream(icecream);
    return Results.Ok();
});

app.MapGet("/icecreams", async (IIcecreamsRepository repository) => 
    await repository.GetIcecreams());
```

在这两个映射方法中，我们将存储库作为参数传递，因为在 Minimal API 中，服务通常作为映射方法的参数传递。

这意味着存储库在代码的所有部分都可用。

在 `MapGet` 端点中，我们使用存储库从存储库的实现中加载所有实体，并将结果作为端点的结果。

在 `MapPost` 端点中，除了存储库参数外，我们还从请求体中接受 `Icecream` 实体，
并将相同的实体作为参数传递给存储库的 `CreateIcecream` 方法。

## 总结

在本章中，我们学习了如何在 Minimal API 项目中使用两种在实际场景中最常见的工具与数据访问层进行交互：EF 和 Dapper。

对于 EF，我们涵盖了一些基本功能，例如设置项目以使用此 ORM 以及如何执行一些基本操作来实现完整的 CRUD API 端点。

我们对 Dapper 也做了基本相同的事情，从一个空项目开始，添加 Dapper，
设置项目以使用 SQL Server LocalDB 工作，并实现与数据库实体的一些基本交互。

在下一章中，我们将专注于 Minimal API 项目中的身份验证和授权。首先，保护数据库中的数据非常重要。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
