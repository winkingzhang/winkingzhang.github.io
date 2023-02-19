---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/minimal-apis/chapter-01
layout: single
classes: wide
sidebar:
  nav: "minimal_apis"
---

## Minimal APIs 简介

这是本书的这一章，我们将介绍一些与 .NET 6.0 中的 `Minimal APIs` 相关的基本信息，并展示如何为 .NET 6 设置开发环境，
更具体地说，如何使用 ASP.NET Core 开发 `Minimal APIs` 项目。

我们将首先从 Minimal APIs 的简要历史开始；然后是使用 Visual Studio 2022 和 Visual Code Studio 创建一个新的 `Minimal APIs` 项目。
最后，我们将看一下我们项目的文件结构。

到本章结束，您将能够创建一个新的 `Minimal APIs` 项目，并以此模版开发 REST API。

在本章中，我们将涵盖一下话题：

- 微软 Web API 简史
- 创建新的 `Minimal APIs` 项目
- 查看项目文件结构


## 技术要求

要使用 ASP.NET Core 6 里的 `Minimal APIs` 技术，首先需要在开发环境中安装 .NET 6。

如果您尚未安装，那我们参照下面的步骤来安装它：

1. 导航到以下链接：[https://dotnet.microsoft.com](https://dotnet.microsoft.com)。
2. 点击下载按钮。
3. 默认情况下，浏览器会为您选择正确的操作系统，但如果没有，请在页面顶部选择您的操作系统。
4. 下载 .NET 6.0 SDK 的 LTS 版本。
5. 启动下载到的安装程序。
6. 重新启动计算机（这不是强制性的）。

您可以在终端中使用以下命令查看开发计算机上安装了哪些 SDK：
```bash
$ dotnet --list-sdks
3.1.426 [/usr/local/share/dotnet/sdk]
6.0.404 [/usr/local/share/dotnet/sdk]
```
在开始写代码之前，您还需要一个代码编辑器或集成开发环境 （IDE）。您可以从以下列表中选择自己喜欢的：

- Visual Studio Code for Windows、Mac 或 Linux (免费)
- Visual Studio 2022 (商业/社区免费)
- Visual Studio 2022 for Mac (商业/社区免费)

> <b>提示</b><br/>
> 除以上推荐列表之外还可以选择 JetBrains Rider (商业) 

在过去的几年里，Visual Studio Code 在开发者社区中非常流行，当然在微软社区中更是非常流行。
这里即使您将 Visual Studio 2022 用于日常工作，我们也建议您下载并安装 Visual Studio Code 并尝试一下。

以下步骤是下载并安装 Visual Studio Code 和一些扩展：

1. 导航到 [https://code.visualstudio.com](https://code.visualstudio.com)。
2. 下载稳定版或预览体验成员版。
3. 启动安装程序。
4. 启动 Visual Studio Code。
5. 单击扩展图标。 
你将在列表顶部看到 C# 扩展。
6. 单击“安装”按钮并等待。

另外还可以安装其他推荐的扩展，以便更好的使用 C# 和 ASP.NET Core 进行开发。下表是我们建议的一些常用扩展：

| 扩展名称                  | 描述                                      |
|-----------------------|-----------------------------------------|
| MSBuild project tools | 对 MSBuild 项目文件提供智能提醒                    |
| REST Client           | 在 Visual Studio Code 里直接发送 HTTP 请求和查看响应 |
| ILSpy.NET Decompiler  | 反编译 MSIL 代码                             |


当然，如果要继续使用 .NET 开发人员最广泛使用的 IDE，推荐下载并安装 Visual Studio 2022。

如果您没有许可证，请检查是否可以使用社区版。获得社区版许可证有一些限制，但如果您是学生、拥有开源项目或想以个人身份使用，则完全没有问题。
以下是下载和安装 Visual Studio 2022 的方法和步骤：

1. 导航到 [https://visualstudio.microsoft.com/downloads/](https://visualstudio.microsoft.com/downloads/)。 
2. 选择“Visual Studio 2022 版本 17.x 或更高版本”并下载。
3. 启动安装程序。 
4. 在“工作负载”选项卡上，选择以下内容：
   - ASP.NET 和 Web 开发
   - Azure 开发
5. 在“单个组件”选项卡上，选择以下内容：
   - 适用于 Windows 的 Git

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，网址为 
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6/tree/main/Chapter01](https://visualstudio.microsoft.com/downloads/)。
现在，您有一个可以尝试并符合本书中使用的代码的环境了。


## 微软 Web API 简史

早在2007年，.NET Web 应用程序随着 ASP.NET MVC 的引入而爆发了一次变革。
从那时起，.NET 为其他语言中常见的模型-视图-控制器模式提供了内置支持。

五年后的2012年，RESTful API 成为互联网上的新趋势，.NET通过一种称为 ASP.NET Web API 技术来开发 API 的新方法对此做出了回应。
这是对 Windows Communication Foundation （WCF） 技术的重大改进，因为它更容易为 Web 开发服务。
后来，在 ASP.NET Core 中，这些框架被统一为 ASP.NET Core MVC：一个用于开发Web应用程序和API的统一框架。

在 ASP.NET Core MVC 应用程序中，控制器负责接受输入、编排操作，并在最后返回响应。开发人员可以使用筛选器、绑定、验证等来扩展整个初处理管道。
因此称得上一个功能齐全的框架，胜任构建现代 Web 应用程序。

但在现实世界中，也有一些场景和用例不需要 MVC 框架的所有功能，或者您必须考虑性能的约束。
而 ASP.NET Core 实现了许多中间件，您可以随意从应用程序中删除或添加到应用程序中，虽然在这种情况下，您可能需要自己实现许多常见功能。

最后，ASP.NET Core 6.0 用 `Minimal APIs` 填补了这些空白。

现在我们已经介绍了 Minimal APIs 的简要历史，我们将在下一节中开始创建新的 `Minimal APIs` 项目。

## 创建新的 Minimal APIs 项目

让我们从我们的第一个项目开始，在编写 RESTful API 时尝试探讨剖析 `Minimal APIs` 方法的新模板。

在本节中，我们将创建第一个 `Minimal APIs` 项目。我们将从使用 Visual Studio 2022 开始，
然后我们将展示如何使用 Visual Studio Code 和 .NET CLI 创建项目。

### 使用 Visual Studio 2022 创建项目

按照以下步骤在 Visual Studio 2022 中创建新项目：

1. 打开 Visual Studio 2022，在主屏幕上，单击创建新项目：
![Figure_1.01_B17902.jpg](/books/minimal-apis/image/Figure_1.01_B17902.jpg)
2. 在下一个屏幕上，在窗口顶部的文本框中编写 API，然后选择名为 ASP.NET Core Web API 的模板：
![Figure_1.02_B17902.jpg](/books/minimal-apis/image/Figure_1.02_B17902.jpg)
3. 接下来，在”配置新项目“屏幕上，插入新项目的名称，并选择新解决方案的根文件夹：
![Figure_1.03_B17902.jpg](/books/minimal-apis/image/Figure_1.03_B17902.jpg)
4. 在以下”其他信息“屏幕上，确保从”框架“下拉列表中选择”.NET 6.0（长期支持）”。
最重要的是，取消选中使用控制器（取消选中以使用 Minimal APIs）选项。
![Figure_1.04_B17902.jpg](/books/minimal-apis/image/Figure_1.04_B17902.jpg)
5. 单击”创建“，几秒钟后，您将看到新的 `Minimal APIs` 项目的代码。

现在我们将展示如何使用 Visual Studio Code 和 .NET CLI 创建相同的项目。

### 使用 Visual Studio Code 创建项目

使用 Visual Studio Code 创建项目比使用 Visual Studio 2022 更容易、更快捷，
因为您不必使用 UI 或向导，而只需使用终端和 .NET CLI。

无需为此安装任何新内容，因为 .NET CLI 包含在 .NET 6 安装中（与以前版本的 .NET SDK 一样）。
按照以下步骤使用 Visual Studio Code 创建项目：

1. 打开控制台、shell 或 Bash 终端，然后切换到工作目录。
2. 使用以下命令创建新的 Web API 应用程序：
   ```bash
   $ dotnet new webapi -minimal -o Chapter01
   ```
   如您所见，我们在前面的命令中插入了 `-minimal` 参数，以使用 `Minimal APIs` 项目模板，而不是带有控制器的 ASP.NET Core 模板。
3. 现在使用以下命令打开带有Visual Studio Code的新项目：
   ```bash
   $ cd Chapter01
   $ code .
   ```

现在我们知道了如何创建一个新的 `Minimal APIs` 项目，我们将快速浏览一下这个新模板的结构。

## 查看项目结构

无论你使用的是 Visual Studio 还是 Visual Studio Code，你都应该在 **Program.cs** 文件中看到以下代码：

```csharp
var builder = WebApplication.CreateBuilder(args);
// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", 
    "Balmy", "Hot", "Sweltering", "Scorching"
};
app.MapGet("/weatherforecast", () =>
{
    var forecast = Enumerable.Range(1, 5).Select(index =>
      new WeatherForecast
      (
         DateTime.Now.AddDays(index),
         Random.Shared.Next(-20, 55),
         summaries[Random.Shared.Next(summaries.Length)]
      ))
      .ToArray();
      return forecast;
}).WithName("GetWeatherForecast");
app.Run();

internal record WeatherForecast(DateTime Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}

```

首先，使用 `Minimal APIs` 方法，你的所有代码都将在 **Program.cs** 文件中。
如果你已经是一个经验丰富的 .NET 开发人员，那将很容易理解这些代码，并且您会发现它类似于您一直使用的控制器方法的一些内容，只不过放到了一起。

归根结底，这是编写 API 的另一种方式，它依然基于 ASP.NET Core。

但是，如果您是 ASP.NET 新手，这种单一文件方法也很容易理解。很容易理解如何扩展模板中的代码并向此 API 添加更多功能。

不要忘记，最小意味着它包含构建 HTTP API 所需的最少组件集，但这并不意味着您将要构建的应用程序将很简单。
它需要像任何其他 .NET 应用程序一样具有良好设计才行。

最后一点，`Minimal APIs` 方法不能替代 MVC 方法。这只是写同样东西的另一种方式。

让我们回到代码。

这里 `Minimal APIs` 的模板使用 .NET 6 Web 应用程序的新方法：顶级语句。
这意味着项目只有一个 **Program.cs** 文件，而不是使用两个文件来配置应用程序。

如果您不喜欢这种编码风格，可以将应用程序转换为 Core 3.x/5 的旧模板 ASP.NET。这种方法在 .NET 中仍然有效。

> 提示<br/> **.NET 6 顶级语法** 模版的信息可以参考：
> [https://docs.microsoft.com/dotnet/core/tutorials/top-level-templates](https://docs.microsoft.com/dotnet/core/tutorials/top-level-templates)

默认情况下，新模板包括对 OpenAPI 规范的支持，更具体地说，是对 Swagger 的支持。 
也就是说我们有开箱即用的接口文档和测试页，无需任何额外的配置。

您可以在以下两行代码中看到 Swagger 的默认配置：
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

很多时候，您不希望将 Swagger 和所有接口暴露给生产或类生产环境。
默认模板预制仅在开发环境中启用也是开箱即用的，因此有以下几行代码：
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```
如代码分支判断条件，应用程序如果在开发环境中运行，则包含 Swagger 文档，否则就没有。

> 提示<br/>
> 后续会在 [第三章, 使用 Minimal APIs](/books/minimal-apis/chapter-03) 专门介绍 Swagger 相关话题

正如前文提到在模板的中几行代码，它为我们揭晓了 .NET 6 Web 应用程序的另一个通用概念：环境 （Environment）。

通常，当我们开发专业应用程序时，应用程序的开发、测试并最终发布给最终用户会经历很多阶段。
按照惯例，这些阶段受到监管，称为开发、分期和生产。作为开发人员，我们可能希望根据当前环境更改应用程序的行为。

有几种方法可以访问此信息，但在现代 .NET 6 应用程序中检索实际环境的典型方法是使用环境变量。
您可以直接从 **Program.cs** 文件中的 `app` 变量访问环境变量。
以下代码块演示如何直接从应用程序的启动点检索有关环境的所有信息：

```csharp
if (app.Environment.IsDevelopment())
{
      // 开发环境 (dev/qa) 专用代码
}
if (app.Environment.IsStaging())
{
      // 类生成环境代码
}
if (app.Environment.IsProduction())
{
      // 生成环境代码
}
```

在许多情况下，您可以定义其他环境，并且可以使用以下代码检查自定义环境：
```csharp
if (app.Environment.IsEnvironment("TestEnvironment"))
{
    // 定制环境 TestEnvironment 特定代码
}
```

为了在 `Minimal APIs` 中定义路由和处理程序，我们使用 `MapGet`，`MapPost`，`MapPut` 和 `MapDelete` 方法。
如果您习惯使用 HTTP 动词，您会注意到动词 **Patch** 不存在，但您可以使用 `MapMethods` 定义任何一组动词。

例如，如果要创建一个新的接口来将一些数据发布到 API，则可以编写以下代码：
```csharp
app.MapPost("/weatherforecast", async (WeatherForecast model, IWeatherService repo) =>
{
    // ...
});
```

正如您在前面的简短代码中看到的那样，使用新的 `Minimal APIs` 模板添加新接口非常容易。
以前，使用绑定参数编写新接口并使用依赖关系注入显得有些困难，尤其是对于新开发人员而言。

> 提示<br/> 后续会在 [第二章，探索 Minimal APIs 及其优点](/books/minimal-apis/chapter-02) 探讨路由，
> 在 [第四章，Minimal APIs 项目中的依赖注入](/books/minimal-apis/chapter-04) 探讨依赖注入


## 总结

在本章中，我们首先从 `Minimal APIs` 的简要历史开始。
接下来，我们看到了如何使用 Visual Studio 2022 以及 Visual Studio Code 和 .NET CLI 创建一个项目。
之后，我们检查了新模板的结构，如何访问不同的环境，以及如何开始与 REST 接口交互。

在下一章中，我们将看到如何绑定参数、新的路由配置以及如何自定义响应。


<br/><br/>
&gt;  [返回扉页](/books/minimal-apis)
