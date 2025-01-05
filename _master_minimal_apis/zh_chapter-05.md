---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-05
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---


## 使用日志记录识别错误

在本章中，我们将开始了解 .NET 提供的日志记录工具。
日志记录器是开发人员在调试应用程序或了解其在生产环境中的故障时必须使用的工具之一。
日志库已内置到 ASP.NET 中，并默认启用了多个功能。
本章的目的是深入研究我们习以为常的事物，并在过程中添加更多信息。

本章将涉及的主题如下：
* 探索 .NET 中的日志记录
* 利用日志记录框架
* 使用 Serilog 存储结构化日志

## 技术要求

如前几章所述，需要有 .NET 6 开发框架。

在执行测试本章中描述的示例时，没有特殊要求。

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，地址为 <br/>
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter05](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter05)

## 探索 .NET 中的日志记录

**ASP.NET Core** 模板创建了一个 **WebApplicationBuilder** 和一个 **WebApplication**，
它们提供了一种简化的方式来配置和运行无需启动类的 Web 应用程序。

如前所述，在 .NET 6 中，为了支持现有的 `Program.cs` 文件，已淘汰了 `Startup.cs` 文件。
所有启动配置都放置在此文件中，在 Minimal API 的情况下，端点实现也放置在此文件中。

我们刚刚描述的是每个 .NET 应用程序及其各种配置的起点。

在应用程序中进行日志记录意味着在代码的不同点跟踪证据，以检查其是否按预期运行。
日志记录的目的是随时间跟踪导致应用程序中出现意外结果或事件的所有条件。
在应用程序的开发和生产过程中，日志记录都可能很有用。

然而，对于日志记录，添加了多达四个提供程序来跟踪应用程序信息：

* **Console**：Console 提供程序将输出记录到控制台。
此日志在生产环境中不可用，因为 Web 应用程序的控制台通常不可见。
在开发过程中，当在桌面计算机上通过 Kestrel 在应用程序控制台窗口中运行应用程序时，此类型的日志对于快速记录很有用。
* **Debug**：Debug 提供程序使用 `System.Diagnostics.Debug` 类写入日志输出。
在开发时，我们习惯于在 Visual Studio 输出窗口中查看此部分。
在 Linux 操作系统下，信息根据发行版在以下位置跟踪：`/var/log/message` 和 `/var/log/syslog`。
* **EventSource**：在 Windows 上，此信息可以在 `EventTracing` 窗口中查看。
* **EventLog**（仅在 Windows 上运行时）：此信息显示在本机 Windows 窗口中，
因此只有在 Windows 操作系统上运行应用程序时才能看到它。

### .NET 最新版本中的新功能

在.NET 的最新版本中添加了新的日志记录提供程序。但是，这些提供程序在框架内未启用。

使用这些扩展来启用新的日志记录场景：`AddSystemdConsole`、`AddJsonConsole` 和 `AddSimpleConsole`。

你可以在以下链接中找到有关如何配置日志以及 ASP.NET 的基本设置的更多详细信息：
[https://docs.microsoft.com/aspnet/core/fundamentals/host/generichost](https://docs.microsoft.com/aspnet/core/fundamentals/host/generichost)

我们已经开始了解框架为我们提供了什么；现在我们需要了解如何在我们的应用程序中利用它。
在继续之前，我们需要了解什么是日志记录层。
这是一个基本概念，将帮助我们将信息分解为不同的层，并根据需要启用它们：

| 日志级别        | 描述                                                              |
|-------------|-----------------------------------------------------------------|
| Trace       | 包含对调试最有用的详细信息。这些消息通常包含在代码中用于调试目的的方法调用的详细信息。                     |
| Debug       | 用于调试目的的详细信息，通常在开发过程中使用。此日志级别包含的信息比 `Trace` 少，但对于分析代码的详细行为仍然很有用。 |
| Information | 跟踪应用程序的正常操作和流程。这些消息通常是关于应用程序正在执行的操作的信息，例如启动服务、处理请求等。            |
| Warning     | 指示出现了意外情况，但应用程序仍在继续运行。此日志级别用于记录可能导致问题的情况，但尚未导致应用程序失败。           |
| Error       | 表示发生了错误，可能会影响应用程序的部分功能。此日志级别用于记录导致应用程序出现错误的情况。                  |
| Critical    | 表示严重错误，可能会导致应用程序停止运行或无法正常工作。此日志级别用于记录最严重的错误情况。                  |
| None        | 禁用所有日志记录。                                                       |

上面的表格显示了从最详细到最不详细的日志级别。

要了解更多信息，你可以阅读题为 “Logging in.NET Core and ASP.NET Core” 的文章，
该文章在此处详细解释了日志记录过程：
[https://docs.microsoft.com/aspnet/core/fundamentals/logging/](https://docs.microsoft.com/aspnet/core/fundamentals/logging/)

如果我们选择 `Information` 作为我们的日志级别，
那么此级别及以下的所有内容（即 `Critical` 级别）都将被跟踪，跳过 `Debug` 和 `Trace`。

我们已经了解了如何利用日志层；现在，让我们继续编写一条记录信息的语句，并允许我们将有价值的内容插入到跟踪系统中。

### 配置日志记录

要开始使用日志记录组件，你需要了解一些信息来开始跟踪数据。
每个日志记录器对象（`ILogger<T>`）必须有一个关联的类别。日志类别允许你以高清晰度对跟踪层进行分段。
例如，如果我们想要跟踪在某个类或 ASP.NET 控制器中发生的所有事情，而不必重写所有代码，我们需要启用我们感兴趣的类别。

类别是一个 `T` 类。再简单不过了。你可以重用日志方法注入的类的类型化对象。
例如，如果我们正在实现 `MyService`，并且我们想要跟踪在该服务中发生的所有事情，
并使用相同的类别，我们只需要从依赖注入引擎请求一个 `ILogger<MyService>` 对象实例。

一旦定义了日志类别，我们需要调用 `ILogger<T>` 对象并利用该对象的公共方法。
在上一节中，我们查看了日志层。每个日志层都有自己的方法来跟踪信息。
例如，`LogDebug` 是用于跟踪 `Debug` 层信息的指定方法。

让我们看一个示例。我在 `Program.cs` 文件中创建了一个记录：
```csharp
internal record CategoryFiltered();
```

此记录用于定义我只想在必要时跟踪的特定日志类别。
为此，建议定义一个类或记录作为其本身的目的，并启用必要的跟踪级别。

在 `Program.cs` 文件中定义的记录没有命名空间；
在定义包含所有必要信息的 `appsettings` 文件时，我们必须记住这一点。

如果日志类别在命名空间内，我们必须考虑类的全名。
在这种情况下，它是 `LoggingSamples.Categories.MyCategoryAlert`：

```csharp
namespace LoggingSamples.Categories
{
    public class MyCategoryAlert
    {
    }
}
```

如果我们不指定类别，如下例所示，所选的日志级别将是默认级别：
```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning",
            "CategoryFiltered": "Information",
            "LoggingSamples.Categories.MyCategoryAlert": "Debug"
        }
    }
}
```

任何包含基础设施日志的内容，例如 Microsoft 日志，都保留在特殊类别中，
例如 `Microsoft.AspNetCore` 或 `Microsoft.EntityFrameworkCore`。

Microsoft 日志类别的完整列表可以在以下链接中找到：
[https://docs.microsoft.com/aspnet/core/fundamentals/logging/#asp-net-core-and-ef-core-categories](https://docs.microsoft.com/aspnet/core/fundamentals/logging/#asp-net-core-and-ef-core-categories)

有时，我们需要根据跟踪提供程序定义某些日志级别。
例如，在开发过程中，我们希望在日志控制台中看到所有信息，但只希望在日志文件中看到错误。

要做到这一点，我们不需要更改配置代码，只需为每个提供程序定义其级别。
以下是一个示例，显示了如何显示 Microsoft 类别中从 `Information` 层到其下的所有跟踪信息：

```csharp
{
    "Logging": {
        "Logging": { // Default, all providers.
            "LogLevel": {
                "Microsoft": "Warning"
            },
            "Console": { // Console provider.
                "LogLevel": {
                    "Microsoft": "Information"
                }
            }
        }
    }
}
```

现在我们已经了解了如何启用日志记录以及如何过滤各种类别，剩下的就是将此信息应用于 Minimal API。

在下面的代码中，我们注入了两个具有不同类别的 `ILogger` 实例。
这不是常见的做法，但我们这样做是为了使示例更具体并展示日志记录器的工作原理：

```csharp
app.MapGet("/first-log", (ILogger<CategoryFiltered> loggerCategory, 
    ILogger<LoggingSamples.Categories.MyCategoryAlert> loggerAlertCategory) =>
{
    loggerCategory.LogInformation("I'm information {MyName}",
        "My Name Information");
    loggerCategory.LogDebug("I'm debug {MyName}", 
        "My Name Debug");
    loggerCategory.LogInformation("I'm debug {Data}", 
        new PayloadData("CategoryRoot", "Debug"));
    loggerAlertCategory.LogInformation("I'm information {MyName}", 
        "Alert Information");
    loggerAlertCategory.LogDebug("I'm debug {MyName}", 
        "Alert Debug");
    var p = new PayloadData("AlertCategory", "Debug");
    loggerAlertCategory.LogDebug("I'm debug {Data}", p);
    return Results.Ok();
})
.WithName("GetFirstLog");
```

在前面的代码片段中，我们注入了两个具有不同类别的日志记录器实例；每个类别跟踪一条单独的信息。
信息根据我们将在下面描述的模板写入。
此示例的效果是，根据级别，我们可以显示或禁用单个类别显示的信息，而无需更改代码。

我们开始按级别和类别过滤日志。
现在，我们想向你展示如何定义一个模板，使我们能够定义消息并使其部分动态化。

### 自定义日志消息

日志方法要求的消息字段是一个简单的字符串对象，我们可以通过日志框架将其丰富和序列化为适当的结构。
因此，消息对于识别故障和错误至关重要，在其中插入对象可以极大地帮助我们识别问题：

```csharp
string apples = "apples";
string pears = "pears";
string bananas = "bananas";
logger.LogInformation("My fruit box has: {pears}, {bananas}",
    apples, pears, bananas);
```

消息模板包含将内容插值到文本消息中的占位符。

除了文本之外，还需要传递参数来替换占位符。
因此，参数的顺序是有效的，但占位符的名称对于替换不是必需的。

结果将考虑位置参数而不是占位符名称：

```
My fruit box has: apples, pears, bananas
```

现在你知道如何自定义日志消息了。接下来，让我们了解基础设施日志记录，这在处理更复杂的场景时至关重要。

### 基础设施日志记录

在本节中，我们想向你介绍一个在 ASP.NET 应用程序中鲜为人知且很少使用的主题：**W3C 日志**。

此日志是所有 Web 服务器（不仅是 **Internet Information Services (IIS)**）使用的标准。
它也适用于 NGINX 和许多其他 Web 服务器，并且也可以在 Linux 上使用。
它还用于跟踪各种请求。然而，日志无法了解调用内部发生的事情。

因此，此功能侧重于基础设施，即进行了多少调用以及调用到哪个端点。

在本节中，我们将看到如何启用跟踪，默认情况下，跟踪存储在文件中。
该功能需要一些时间来查找，但启用了必须使用适当的实践和工具（如 `OpenTelemetry`）管理的更复杂的场景。

#### OpenTelemetry

OpenTelemetry 是一组工具、API 和 SDK。
我们使用它来检测、生成、收集和导出遥测数据（指标、日志和跟踪），以帮助分析软件性能和行为。
你可以在 OpenTelemetry 官方网站上了解更多信息：
[https://opentelemetry.io/](https://opentelemetry.io/)

要配置 W3C 日志记录，需要注册 `AddW3CLogging` 方法并配置所有可用选项。
要启用日志记录，只需添加 `UseW3CLogging`。

日志的写入方式不变；这两个方法启用刚刚描述的场景并开始将数据写入 W3C 日志标准：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddW3CLogging(logging =>
{
    logging.LoggingFields = W3CLoggingFields.All;
});
var app = builder.Build();
app.UseW3CLogging();
app.MapGet("/first-w3c-log", (IWebHostEnvironment webHostEnvironment) =>
{
    return Results.Ok(new {
        PathToWrite = webHostEnvironment.ContentRootPath 
    });
})
.WithName("GetW3CLog");
```

我们报告创建的文件的头部（稍后将跟踪信息的头部）：
```text
#Version: 1.0
#Start-Date: 2022-01-03 10:34:15
#Fields: date time c-ip cs-username s-computername s-
```

我们已经看到了如何跟踪有关托管我们应用程序的基础设施的信息；
现在，我们想利用 .NET 6 中的新功能提高日志性能，这些功能有助于设置标准日志消息并避免错误。

### 源生成器（Source generators）

.NET 6 的一个新特性是源生成器（Source generators）；它们是在编译时生成可执行代码的性能优化工具。
因此，在编译时生成可执行代码会提高性能。在程序执行阶段，所有结构都与程序员在编译前编写的代码相当。

使用 `$""` 进行字符串插值通常很棒，并且比 `string.Format()` 使代码更具可读性，
但在编写日志消息时几乎不应使用它：
```csharp
logger.LogInformation($"I'm {person.Name}-{person.Surname}");
```

此方法在控制台的输出与使用结构日志记录时相同，但存在几个问题：
* 你会失去**结构化日志**，并且无法按*格式值过滤*或在 NoSQL 产品的自定义字段中归档日志消息。
* 同样，你不再有一个恒定的*消息模板*来查找所有相同的日志。
* 在将字符串传递给 `LogInformation` 之前，会提前对 person 进行序列化。
* 即使未启用日志过滤器，也会进行序列化。
为了避免处理日志，需要检查层是否处于活动状态，这会使代码的可读性大大降低。

假设你决定更新日志消息以包括 Age 以澄清为什么要写入日志：
```csharp
public partial class LogGenerator
{
    private readonly ILogger<LogGeneratorCategory> _logger;

    public LogGenerator(ILogger<LogGeneratorCategory> logger)
    {
        _logger = logger;
    }

    [LoggerMessage(
        EventId = 100,
        EventName = "Start",
        Level = LogLevel.Debug,
        Message = "Start Endpoint: {endpointName} with data {dataIn}")]
    public partial void StartEndpointSignal(string endpointName, 
        object dataIn);

    [LoggerMessage(
        EventId = 101,
        EventName = "StartFiltered",
        Level = LogLevel.Debug,
        Message = "Log level filtered: {endpointName} with data {dataIn}")]
    public partial void LogLevelFilteredAtRuntime(LogLevel logLevel, 
        string endpointName, object dataIn);
}

public class LogGeneratorCategory { }
```

在前面的示例中，我们创建了一个部分类，注入了日志记录器及其类别，并实现了两个方法。这些方法在以下代码中使用：

```csharp
app.MapPost("/start-log", (PostData data, LogGenerator logGenerator) =>
{
    logGenerator.StartEndpointSignal("start-log", data);
    logGenerator.LogLevelFilteredAtRuntime(LogLevel.Debug, "start-log", data);
})
.WithName("StartLog");

internal record PostData(DateTime Date, string Name);
```

注意在第二个方法中，我们也有在运行时定义日志级别的可能性。

在幕后，[LoggerMessage] 源生成器生成 `LoggerMessage.Define()` 代码以优化你的方法调用。
以下输出显示了生成的代码：

```csharp
[global:System.CodeDom.Compiler.GeneratedCodeAttribute("Microsoft.Extensions.Logging.LoggerMessageGenerator", "6.0.0.0")]
public partial void LogLevelFilteredAtRuntime(global::Microsoft.Extensions.Logging.LogLevel logLevel, global::System.String endpointName, global::System.Object dataIn)
{
    if (_logger.IsEnabled(logLevel))
    {
        _logger.Log(
            logLevel,
            new global::Microsoft.Extensions.Logging.EventId(101, "StartFiltered"),
            new LogLevelFilteredAtRuntimeStructuredLoggerMessageFormat<string, object>(endpointName, dataIn),
            null,
            LogLevelFilteredAtRuntimeStructuredLoggerMessageFormat<string, object>.Format);
    }
}
```

在本节中，你已经了解了一些日志记录提供程序、不同的日志级别、
如何配置它们、如何修改消息模板的部分、启用日志记录以及源生成器的好处。
在下一节中，我们将更多地关注日志记录提供程序。

## 利用日志记录框架

如本章开头所述，日志记录框架已经设计了一系列提供程序，无需添加任何其他包。
现在，让我们探索如何使用这些提供程序以及如何构建自定义提供程序。
我们将仅分析 _Console_ 日志提供程序，因为它包含了在其他日志提供程序上复制相同推理所需的所有元素。

### Console 日志

`Console` 日志提供程序是最常用的，因为在开发过程中，它为我们提供了大量信息并收集所有应用程序错误。

自 .NET 6 以来，此提供程序加入了 `AddJsonConsole` 提供程序，
除了像控制台一样跟踪错误外，它还将错误序列化为人类可读的 JSON 对象。

在以下示例中，我们展示了如何配置 `JsonConsole` 提供程序，并在写入 JSON 有效负载时添加缩进：

```csharp
builder.Logging.AddJsonConsole(options =>
{
    options.JsonWriterOptions = new JsonWriterOptions
    {
        Indented = true
    };
});
```

正如我们在前面的示例中看到的，我们将使用消息模板跟踪信息：

```csharp
app.MapGet("/first-log", (ILogger<CategoryFiltered> loggerCategory) =>
{
    loggerCategory.LogInformation("I'm information {MyName}",
        "My Name Information");
    loggerCategory.LogDebug("I'm debug {MyName}", 
        "My Name Debug");
    loggerCategory.LogInformation("I'm debug {Data}", 
        new PayloadData("CategoryRoot", "Debug"));
    loggerCategory.LogDebug("I'm debug {Data}", 
        new PayloadData("AlertCategory", "Debug"));
    return Results.Ok();
})
.WithName("GetFirstLog");
```

最后，重要的是要注意：`Console` 和 `JsonConsole` 提供程序不会序列化通过消息模板传递的对象，而只会写入类名。

```csharp
var p = new PayloadData("AlertCategory", "Debug");
loggerCategory.LogDebug("I'm debug {Data}", p);
```

这绝对是提供程序的一个限制。因此，我们建议使用结构化日志记录工具，
如 `NLog`、`log4net` 和 `Serilog`，我们将在稍后讨论。

我们展示了前面几行代码使用刚刚描述的两个提供程序的输出：
![Figure_5.1 - AddJsonConsole output](/assets/images/master-minimal-apis/Figure_5.1_B17902.jpg)

上图显示了格式化为 JSON 的日志，与传统控制台日志相比有几个额外的细节。

![Figure 5.2 – Default logging provider Console output](/assets/images/master-minimal-apis/Figure_5.2_B17902.jpg)

上图显示了默认日志记录提供程序 Console 的输出。

鉴于默认提供程序，我们想向你展示如何创建一个适合应用程序需求的自定义提供程序。

### 创建自定义提供程序

微软设计的日志记录框架可以轻松定制。因此，让我们学习如何创建一个自定义提供程序。

为什么要创建自定义提供程序呢？简单地说，是为了不依赖日志记录库，并更好地管理应用程序的性能。
最后，它还封装了特定场景的一些自定义逻辑，使代码更易于管理和可读。

在以下示例中，我们简化了使用场景，以向你展示创建一个可用的日志记录提供程序所需的最少组件。

提供程序的一个基本部分是能够配置其行为。
让我们创建一个可以在应用程序启动时自定义或从 `appsettings` 检索信息的类。

在我们的示例中，我们定义了一个固定的 `EventId` 来验证每日滚动文件逻辑和写入文件的路径：
```csharp
public class FileLoggerConfiguration
{
    public int EventId { get; set; }
    public string PathFolderName { get; set; } = "logs";
    public bool IsRollingFile { get; set; }
}
```

我们正在编写的自定义提供程序将负责将日志信息写入文本文件。
我们通过实现一个名为 `FileLogger` 的日志类来实现这一点，该类实现了 `ILogger` 接口。

在类逻辑中，我们所做的就是实现 `Log` 方法并检查将信息放入哪个文件。

我们将目录验证放在下一个文件中，但将所有控制逻辑放在此方法中更正确。
我们还需要确保 `Log` 方法在应用程序级别不会抛出异常。日志记录器不应影响应用程序的稳定性。

```csharp
public class FileLogger : ILogger
{
    private readonly string name;
    private readonly Func<FileLoggerConfiguration> getCurrentConfig;

    public FileLogger(string name, 
        Func<FileLoggerConfiguration> getCurrentConfig)
    {
        this.name = name;
        this.getCurrentConfig = getCurrentConfig;
    }

    public IDisposable BeginScope<TState>(TState state) => default!;

    public bool IsEnabled(LogLevel logLevel) => true;

    public void Log<TState>(LogLevel logLevel, 
        EventId eventId, 
        TState state, 
        Exception? exception, 
        Func<TState, Exception?, string> formatter)
    {
        if (!IsEnabled(logLevel))
        {
            return;
        }

        var config = getCurrentConfig();
        if (config.EventId == 0 || config.EventId == eventId.Id)
        {
            string line = $"{name} - {formatter(state, exception)}";
            string fileName = config.IsRollingFile? RollingFileName : FullFileName;
            string fullPath = Path.Combine(config.PathFolderName, fileName);
            File.AppendAllLines(fullPath, new[] { line });
        }
    }

    private static string RollingFileName => 
        $"log-{DateTime.UtcNow:yyyy-MM-dd}.txt";
    private const string FullFileName = "logs.txt";
}
```

现在，我们需要实现 `ILoggerProvider` 接口，该接口旨在创建刚刚讨论的日志记录器类的一个或多个实例。

在这个类中，我们检查前面段落中提到的目录，
但我们也通过 `IOptionsMonitor<T>` 检查 `appsettings` 文件中的设置是否更改：

```csharp
public class FileLoggerProvider : ILoggerProvider
{
    private readonly IDisposable onChangeToken;
    private FileLoggerConfiguration currentConfig;
    private readonly ConcurrentDictionary<string, FileLogger> _loggers
        = new();

    public FileLoggerProvider(
        IOptionsMonitor<FileLoggerConfiguration> config)
    {
        currentConfig = config.CurrentValue;
        CheckDirectory();
        onChangeToken = config.OnChange(updateConfig =>
        {
            currentConfig = updateConfig;
            CheckDirectory();
        });
    }

    public ILogger CreateLogger(string categoryName)
    {
        return _loggers.GetOrAdd(categoryName, 
            name => FileLogger(name, 
            () => currentConfig));
    }

    public void Dispose()
    {
        _loggers.Clear();
        onChangeToken.Dispose();
    }

    private void CheckDirectory()
    {
        if (!Directory.Exists(currentConfig.PathFolderName))
        {
            Directory.CreateDirectory(currentConfig.PathFolderName);
        }
    }
}
```

最后，为了简化其在应用程序启动阶段的使用和配置，我们还定义了一个扩展方法来注册刚刚提到的各种类。

`AddFile` 方法将注册 `ILoggerProvider` 并将其与配置耦合
（作为示例非常简单，但它封装了配置和使用自定义提供程序的几个方面）：

```csharp
public static class FileLoggerExtensions
{
    public static ILoggingBuilder AddFile(
        this ILoggingBuilder builder)
    {
        builder.AddConfiguration();
        builder.Services.TryAddEnumerable(
            ServiceDescriptor.Singleton<ILoggerProvider, FileLoggerProvider>());
        LoggerProviderOptions.RegisterProviderOptions<FileLoggerConfiguration, 
            FileLoggerProvider>(builder.Services);
        return builder;
    }

    public static ILoggingBuilder AddFile(
        this ILoggingBuilder builder, 
        Action<FileLoggerConfiguration> configure)
    {
        builder.AddFile();
        builder.Services.Configure(configure);
        return builder;
    }
```

我们在 `Program.cs` 文件中使用 `AddFile` 扩展记录了所看到的一切，如下所示：

```csharp
builder.Logging.AddFile(configuration =>
{
    configuration.PathFolderName = Path.Combine(
        builder.Environment.ContentRootPath, "logs");
    configuration.IsRollingFile = true;
});
```

输出如图 5.3 所示，我们可以在前五行中看到 `Microsoft` 日志类别（这是经典的应用程序启动信息）：
![Figure 5.3 – File log provider output](/assets/images/master-minimal-apis/Figure_5.3_B17902.jpg)

然后，调用了我们在前面部分中报告的 Minimal API 的处理程序。
如你所见，没有序列化异常数据或传递给日志记录器的数据。

要添加此功能，需要重写 `ILogger formatter` 并支持对象的序列化。
这将为你提供在生产场景中有用的日志记录框架所需的一切。

我们已经看到了如何配置日志以及如何自定义提供程序对象以创建发送到服务或存储的结构化日志。

在下一节中，我们想描述 Azure Application Insights 服务，它对于日志记录和应用程序监控都非常有用。

### Azure Application Insights

除了已经看到的提供程序之外，最常用的提供程序之一是 **Azure Application Insights**。
此提供程序允许你将每个单独的日志事件发送到 Azure 服务。
要将此提供程序插入到我们的项目中，我们只需安装以下 NuGet 包：

```xml
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.18.0" />
```

注册提供程序非常容易。

我们首先注册 Application Insights 框架，`AddApplicationInsightsTelemetry`，
然后在 `AddApplicationInsights` 日志记录框架上注册其扩展。

在前面描述的 NuGet 包中，用于将组件记录到日志记录框架的包也作为参考存在：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplicationInsightsTelemetry();
builder.Logging.AddApplicationInsights();
```

要注册 instrumentation key，这是在 Azure 上注册服务后颁发的密钥，你需要将此信息传递给注册方法。
我们可以通过将其放置在 `appsettings.json` 文件中使用以下格式来避免硬编码此信息：
```json
{
    "ApplicationInsights": {
        "InstrumentationKey": "your-key"
    }
}
```

此过程也在文档中进行了描述
（[https://docs.microsoft.com/it-it/azure/azure-monitor/app/asp-netcore#enable-application-insights-server-side-telemetry-no-visualstudio](https://docs.microsoft.com/it-it/azure/azure-monitor/app/asp-netcore#enable-application-insights-server-side-telemetry-no-visualstudio)）

通过启动前面部分中已经讨论过的方法，我们将所有信息挂钩到 Application Insights。

Application Insights 将日志分组在特定的跟踪下。
跟踪是对 API 的调用，因此在该调用中发生的所有事情在逻辑上都被分组在一起。
此功能利用了 `WebServer` 信息，特别是由 W3C 标准为每个调用发出的 `TraceParentId`。

通过这种方式，如果我们处于微服务应用程序或多个相互协作的服务中，
Application Insights 可以绑定各种 Minimal API 之间的调用。

![Figure 5.4 – Application Insights with a standard log provider](/assets/images/master-minimal-apis/Figure_5.4_B17902.jpg)

我们注意到日志记录框架的默认格式化程序不会序列化 `PayloadData` 对象，而只会写入对象的文本。

在我们将投入生产的应用程序中，还需要跟踪对象的序列化。

了解对象在时间上的状态对于在数据库中运行查询或读取从数据库读取的数据时分析发生的错误至关重要。

### 使用 Serilog 存储结构化日志

正如我们刚刚讨论的，在日志中跟踪结构化对象对于理解错误有很大帮助。

因此，我们建议使用众多日志记录框架之一：`Serilog`。

Serilog 是一个全面的库，已经编写了许多接收器，允许你存储日志数据并稍后搜索它。

Serilog 是一个日志记录库，允许你在多个数据源上跟踪信息。
在 Serilog 中，这些源称为接收器，它们允许你通过对传递给日志记录系统的数据进行序列化来在日志中写入结构化数据。

让我们看看如何开始在 Minimal API 应用程序中使用 Serilog。
让我们安装以下 NuGet 包。
我们的目标是跟踪我们到目前为止一直在使用的相同信息，特别是 `Console` 和 `ApplicationInsights`：

```xml
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.18.0" />
<PackageReference Include="Serilog.AspNetCore" Version="4.1.0" />
<PackageReference Include="Serilog.Settings.Configuration" Version="3.3.0" />
<PackageReference Include="Serilog.Sinks.ApplicationInsights" Version="3.1.0" />
```

第一个包是应用程序中所需的 **ApplicationInsights** SDK。
第二个包允许我们在 ASP.NET 管道中注册 Serilog，并能够利用 Serilog。
第三个包允许我们在 `appsettings` 文件中配置框架，而无需重写应用程序来更改参数或代码。
最后，我们有添加 **ApplicationInsights** 接收器的包。

在 `appsettings` 文件中，我们创建一个新的 `Serilog` 部分，在其中我们应该在 `Using` 部分注册各种接收器。
我们注册日志级别、接收器、丰富每个事件信息的丰富器以及属性，如应用程序名称：

```json
{
    "Serilog": {
        "Using": [
            "Serilog.Sinks.Console",
            "Serilog.Sinks.ApplicationInsights"
        ],
        "MinimumLevel": "Verbose",
        "WriteTo": [
            {
                "Name": "Console"
            },
            {
                "Name": "ApplicationInsights",
                "Args": {
                    "restrictedToMinimumLevel": "Information",
                    "telemetryConverter": "Serilog.Sinks.ApplicationInsights.Sinks.ApplicationInsightsTelemetryConverters.TraceTelemetryConverter, Serilog.Sinks.ApplicationInsights"
                }
            }
        ],
        "Enrich": [
            "FromLogContext"
        ],
        "Properties": {
            "Application": "MinimalApi.Packt"
        }
    }
}
```

现在，我们只需在 ASP.NET 管道中注册 `Serilog`：

```csharp
using Microsoft.ApplicationInsights.Extensibility;
using Serilog;

var builder = WebApplication.CreateBuilder(args);
builder.Logging.AddSerilog();
builder.Services.AddApplicationInsightsTelemetry();
var app = builder.Build();
Log.Logger = new LoggerConfiguration()
    .WriteTo.ApplicationInsights(
        app.Services.GetRequiredService<ITelemetryConfiguration>())
    .CreateLogger();
```

通过 `builder.Logging.AddSerilog()` 语句，我们将 Serilog 注册到日志记录框架，
所有记录的事件都将通过通常的 `ILogger` 接口传递给它。
由于框架需要注册 `TelemetryConfiguration` 类来注册 `ApplicationInsights`，
我们被迫将配置挂钩到 Serilog 的静态 `Logger` 对象。
这都是因为 Serilog 会将来自 Microsoft 日志记录框架的信息转换到 Serilog 框架，并添加所有必要的信息。

用法与之前非常相似，但这次我们在消息模板中添加一个 @（at），这将告诉 Serilog 序列化发送的对象。

通过这个非常简单的 {@Person} 措辞，
我们将能够实现序列化对象并将其发送到 `ApplicationInsights` 服务的目标：

```csharp
app.MapGet("/serilog", (ILogger<CategoryFiltered> loggerCategory) =>
{
    loggerCategory.LogInformation("I'm {@Person}", 
        new Person("Andrea", "Tosato", new DateTime(1986, 11, 9)));
    return Results.Ok();
})
.WithName("GetFirstLog");

internal record Person(string Name, string Surname, DateTime BirthDate);
```

最后，我们必须在 ApplicationInsights 服务中找到以 JSON 格式序列化的完整数据。

![Figure 5.5 – Application Insights with structured data](/assets/images/master-minimal-apis/Figure_5.5_B17902.jpg)


## 总结
我们已经看到了几个 Minimal API 实现的日志记录方面。

我们开始欣赏 ASP.NET 提供的日志记录框架，并了解如何配置和自定义它。
我们专注于如何定义消息模板以及如何使用源生成器避免错误。

我们看到了如何使用新的提供程序以 JSON 格式序列化日志并创建自定义提供程序。
这些元素对于掌握日志记录工具并根据自己的喜好进行自定义非常重要。

不仅提到了应用程序日志，还提到了基础设施日志，它与 Application Insights 一起成为监控应用程序的关键要素。
最后，我们了解到有现成的工具，如 Serilog，通过安装一些 NuGet 包，只需几步就能帮助我们获得即用功能。

在下一章中，我们将介绍验证输入对象到 API 的机制。
这是一个基本功能，可以向调用返回正确的错误，
并丢弃不准确的请求或由非法活动（如垃圾邮件和攻击）引发的请求，
这些活动旨在给我们的服务器带来负载。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
