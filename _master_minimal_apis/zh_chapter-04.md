---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-04
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---


## Minimal APIs 项目中的依赖注入

在本章中，我们将讨论 .NET 6.0 中 Minimal API 的一些基本主题。
我们将了解它们与之前版本的.NET 中使用的基于控制器的 Web API 有何不同。
我们还将试图强调这种编写 API 的新方法的优缺点。

本章将涵盖以下主题：
- 什么是依赖注入？
- 在最小 API 项目中实现依赖注入

## 技术要求

要跟上本章中的叙述，你需要创建一个 ASP.NET Core 6.0 Web API 应用程序。
你可以参考 [第 2 章 “探索最小 API 及其优势”](/books/master-minimal-apis/chapter-02) 中的技术要求部分，了解如何创建。

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，地址为：<br/>
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter04](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter04)

## 什么是依赖注入？

一段时间以来，.NET 原生支持**依赖注入**（通常简称为 **DI**）软件设计模式。

依赖注入是在 .NET 中实现服务类及其依赖项之间的**控制反转**（**IoC**）模式的一种方式。
顺便说一下，在 .NET 中，许多基础服务都是使用依赖注入构建的，例如*日志记录*、*配置*和*其他服务*。

让我们看一个实际示例来更好地理解它是如何工作的。

一般来说，依赖项是依赖于另一个对象的对象。在下面的示例中，我们有一个 `LogWriter` 类，其中只有一个名为 `Log` 的方法：

```csharp
public class LogWriter
{
    public void Log(string message)
    {
        Console.WriteLine(
            $"LogWriter.Write(message: \"{message}\")");
    }
}
```

项目中的其他类或其他项目可以创建 `LogWriter` 类的实例并使用 `Log` 方法。

看下面的示例：

```csharp
public class Worker
{
    private readonly LogWriter _logWriter = new LogWriter();
    protected async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logWriter.Log(
                $"Worker running at: {DateTimeOffset.Now}");
             await Task.Delay(1000, stoppingToken);
        }
    }
}
```

这个类直接依赖于 `LogWriter` 类，并且在项目的每个类中都是硬编码的。

这意味着如果你想更改 `Log` 方法，你会遇到一些问题；例如，你必须在解决方案的每个类中替换实现。

前面的实现如果你想在解决方案中实现单元测试会有一些问题。创建 `LogWriter` 类的模拟并不容易。

依赖注入可以通过对我们的代码进行一些更改来解决这些问题：

- 使用接口抽象依赖项。
- 在与 .NET 连接的内置服务中注册依赖注入。
- 将服务注入到类的构造函数中。

前面的事情可能看起来需要对你的代码进行很大的更改，但实际上它们很容易实现。

让我们看看如何通过前面的示例实现这个目标：

1. 首先，我们将创建一个 `ILogWriter` 接口来抽象我们的日志记录器：
```csharp
public interface ILogWriter
{
    void Log(string message);
}
```
2. 接下来，在一个名为 `ConsoleLogWriter` 的实际类中实现这个 `ILogWriter` 接口：
```csharp
public class ConsoleLogWriter : ILogWriter
{
    public void Log(string message)
    {
        Console.WriteLine(
            $"ConsoleLogWriter.Write(message: \"{message}\")");
    }
}
```
3. 现在，更改 `Worker` 类，用新的 `ILogWriter` 接口替换显式的 `LogWriter` 类：
```csharp
public class Worker
{
    private readonly ILogWriter _logWriter;
    public Worker(ILogWriter logWriter)
    {
        _logWriter = logWriter;
        protected async Task ExecuteAsync(
            CancellationToken stoppingToken)
        {
            while(!stoppingToken.IsCancellationRequested)
            {
                _logWriter.Log(
                    $"Worker running at: {DateTimeOffset.Now}");
                await Task.Delay(1000, stoppingToken);
            }
        }
    }
}
```

如你所见，以这种新方式工作很容易，并且优点很多。以下是依赖注入的一些优点：

* 可维护性
* 可测试性
* 可重用性

现在我们需要执行最后一步，即在应用程序启动时注册依赖项。

4.  在 `Program.cs` 文件的顶部添加以下代码行：
```csharp
builder.Services.AddScoped<ILogWriter, ConsoleLogWriter>();
```

在下一节中，我们将讨论依赖注入生命周期之间的差异，
这是在 Minimal API 项目中使用依赖注入之前需要理解的另一个概念。


### 理解依赖注入生命周期

在上一节中，我们了解了在项目中使用依赖注入的好处以及如何转换我们的代码以使用它。

在最后一段中，我们将我们的类作为服务添加到 .NET 的 `ServiceCollection` 中。

在本节中，我们将尝试理解每个依赖注入生命周期之间的差异。

服务生命周期定义了容器创建对象后对象的存活时间。

在注册时，依赖项需要定义生命周期。这定义了创建新服务实例的条件。
在下面的列表中，你可以找到 .NET 中定义的生命周期：

* **瞬态（Transient）**：每次请求时都会创建类的新实例。
* **作用域（Scoped）**：每个作用域（例如，对于相同的 HTTP 请求）创建一次类的新实例。
* **单例（Singleton）**：仅在第一次请求时创建类的新实例。下一次请求将使用相同类的相同实例。

在 Web 应用程序中，通常只发现前两个生命周期，即瞬态和作用域。

如果有特定的用例需要单例，这并不是被禁止的，但作为最佳实践，建议在 web 应用程序中避免使用它们。

在前两种情况下，瞬态和作用域，服务在请求结束时被释放。

在下一节中，我们将看到如何在一个简短的演示中实现我们在最后两节中提到的所有概念（依赖注入的定义及其生命周期），你可以将其用作下一个项目的起点。

## 在 Minimal API 项目中实现依赖注入

在了解了如何在 ASP.NET Core 项目中使用依赖注入之后，让我们尝试理解如何在我们的 Minimal API 项目中使用它，
从使用 `WeatherForecast` 端点的默认项目开始。

这是 `WeatherForecast` GET 端点的实际代码：

```csharp
app.MapGet("/weatherforecast", () =>
{
    var forecast = Enumerable.Range(1, 5)
        .Select(index => new WeatherForecast(
            DateTime.Now.AddDays(index),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
});
```

如前所述，这段代码可以工作，但不容易测试它，尤其是天气新值的创建。

最好的选择是使用一个服务来创建假值并通过依赖注入使用它。

让我们看看如何更好地实现我们的代码：

1. 首先，在 `Program.cs` 文件中添加一个名为 `IWeatherForecastService` 的新接口，并定义一个返回 `WeatherForecast` 实体数组的方法：
```csharp
public interface IWeatherForecastService
{
    WeatherForecast[] GetForecast();
}
```
2. 下一步是创建继承自该接口的类的实际实现。
   代码应该如下所示：
```csharp
public class WeatherForecastService : IWeatherForecastService
{
}
```
3. 现在将项目模板中的代码剪切并粘贴到我们新的服务实现中。最终代码如下所示：
```csharp
public class WeatherForecastService : IWeatherForecastService
{
    public WeatherForecast[] GetForecast()
    {
        var summaries = new[]
        {
            "Freezing", "Bracing", "Chilly", "Cool",
            "Mild", "Warm", "Balmy", "Hot", "Sweltering",
            "Scorching"
        };

        var forecast = Enumerable.Range(1, 5).
        Select(index => new WeatherForecast(
            DateTime.Now.AddDays(index),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
        return forecast;
    }
}
```
4. 我们现在准备将 `WeatherForecastService` 的实现作为依赖注入添加到我们的项目中。
为此，在 `Program.cs` 文件的第一行代码下方插入以下行：
```csharp
builder.Services.AddScoped<IWeatherForecastService, WeatherForecastService>();
```

当应用程序启动时，将我们的服务插入到服务集合中。我们的工作还没有完成。

我们需要在 `WeatherForecast` 端点的默认 `MapGet` 实现中使用我们的服务。

Minimal API 有自己的参数绑定实现，并且很容易使用。

首先，为了使用依赖注入实现我们的服务，我们需要从端点中删除所有旧代码。

删除代码后的端点代码如下所示：

```csharp
app.MapGet("/weatherforecast", () =>
{
});
```

我们可以通过简单地用新代码替换旧代码来轻松改进我们的代码并使用依赖注入：

```csharp
app.MapGet("/weatherforecast", (IWeatherForecastService weatherForecastService) =>
{
    return weatherForecastService.GetForecast();
});
```

在 Minimal API 项目中，服务集合中的实际服务实现作为参数传递给函数，并且你可以直接使用它们。

有时，在启动阶段，你可能需要在主函数中直接使用依赖注入中的服务。
在这种情况下，你必须从服务集合中检索实现的实例，如下代码片段所示：

```csharp
using (var scope = app.Services.CreateScope())
{
    var service = scope.ServiceProvider
        .GetRequiredService<IWeatherForecastService>();
    service.GetForecast();
}
```

在本节中，我们从默认模板开始在 Minimal API 项目中实现了依赖注入。

我们重用了现有代码，但使用了更适合未来维护和测试的架构逻辑来实现它。

## 总结
依赖注入是在现代应用程序中实现的一种非常重要的方法。
在本章中，我们了解了什么是依赖注入并讨论了其基本原理。
然后，我们看到了如何在 Minimal API 项目中使用依赖注入。

在下一章中，我们将关注现代应用程序的另一个重要层，并讨论如何在 Minimal API 项目中实现日志记录策略。

<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
