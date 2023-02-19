---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/minimal-apis/preface
layout: single
classes: wide
sidebar:
  nav: "minimal_apis"
---

# 前言

每个开发人员都有一个梦想，那就是简化代码。 而 `Minimal APIs` 这个 .NET 6 中的一项新功能恰恰好剑指代码简化。
它旨在 ASP.NET Core 中构建仅仅具有最小依赖的 API， 并通过使用更紧凑的代码语法来简化开发。

使用 `Minimal APIs` 的开发人员在很多情况下将能够利用此语法以更少的代码和更少的文件来更快地完成工作。 
这里，您将了解到 .NET 6 的主要新功能，并了解 `Minimal APIs` 的基本概念，显然这些在 .NET 5 和以前的版本中不存在的。
此外您将了解如何为 API 文档启用 `Swagger` 以及 `CORS`，以及如何处理应用程序错误。 
还有，您将学习并使用 Microsoft 最新 .NET 框架下的依赖注入，以更好地构建代码。 
最后，您将看到.NET 6 中和 `Minimal APIs` 同期引入的性能和基准测试的改进。

到本书结束时，您将能够利用 `Minimal APIs`，并了解它们与 Web API 的经典开发有何关联。

# 适用对象

本书适用于想要构建 .NET 和 .NET Core API 并希望学习 .NET 6 新功能的 .NET 开发人员，
但假定您已具备 `C#`、`.NET`、`Visual Studio` 和 `REST API` 的基本知识。

# 涵盖内容

- "Minimal APIs 简介" 介绍了在 .NET 6 中引入 `Minimal APIs` 背后的动机。
我们将解释 .NET 6 的主要新功能以及 .NET 团队使用此最新版本所做的工作。你会明白写这本书的原因。 
- "探索 Minimal APIs 及其优点" 介绍了 `Minimal APIs` 与 .NET 5 和所有早期版本不同的基本方法。
我们将详细探讨 `System.Text.JSON` 的路由和序列化。最后，我们将以一些与编写第一个 REST API 相关的概念结束。 
- "使用 Minimal APIs" 介绍了 `Minimal APIs` 与 .NET 5 和所有早期版本不同的高级方法。
我们将详细探讨如何为 API 文档启用 `Swagger`。我们将看到如何启用 `CORS` 以及如何处理应用程序错误。 
- "Minimal APIs 项目中的依赖注入"，向您介绍了依赖注入，并介绍了如何在 `Minimal APIs` 中使用它。 
- "使用日志记录识别错误" 介绍了 .NET 提供的日志记录工具。日志是开发人员必须用来调试应用程序或了解其在生产中的故障的工具之一。
日志记录相关库已内置到 ASP.NET 中，具有默认已启用其中多个功能。 
- "探索验证和映射" 将教您如何验证 API 的传入数据以及如何返回任何错误或消息。验证数据后，可以将其映射到模型，然后用于处理请求。 
- "与数据访问层的集成" 可帮助您了解在 `Minimal APIs` 中访问和使用数据层的最佳实践。 
- "添加身份验证和授权" 介绍如何利用我们自己的数据库或云服务（如 Azure Active Directory）编写身份验证和授权系统。 
- "利用全球化和本地化" 介绍了如何在 `Minimal APIs` 项目中利用多语言系统，并以客户端的相同语言提供错误。 
- "评估 Minimal APIs 的性能并对其进行基准测试" 介绍了 .NET 6 中的改进点以及在 `Minimal APIs` 中使有所体现部分。


# 本书需求和使用指南

您需要具有 ASP.NET 和 Web 开发工作负载的 `Visual Studio 2022`，或者在您的计算机上安装 `Visual Studio Code` 和 `K6`。

所有代码示例都已在 Windows 操作系统上使用 `Visual Studio 2022` 和 `Visual Studio Code` 进行了测试。

| 本书涉及的软硬件           | 操作系统要求                   |
|--------------------|--------------------------|
| Visual Studio 2022 | Windows, macOS, or Linux |
| Visual Studio Code | Windows, macOS, or Linux |
| K6 (ver 0.39)      | Windows                  |


**如果您使用的是本书的电子版本，我们建议您自己输入代码或从本书的 GitHub 存储库访问代码（下一节中提供了链接）。
这样做将帮助您避免与代码复制和粘贴相关的任何潜在错误。**

此外，要完全理解这本书，您还需要微软 web 技术的基本开发技能。


# 下载示例代码

您可以 从 GitHub 仓库下载本书的示例代码文件， 地址是：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core-6)。
如果代码有更新，它将在 GitHub 存储库中更新。

我们还在 [https://github.com/PacktPublishing/](https://github.com/PacktPublishing/) 上提供了丰富的书籍和视频目录中的其他代码包。快来看看吧！

# 格式约定

本书通篇使用了多种文本约定。


**文本中的代码**：指示文本中的代码字、数据库表名称、文件夹名称、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。
下面是一个示例：“在 Minimal APIs 中，我们使用 **WebApplication** 对象的 **Map*** 方法定义路由模式。

代码块如下：

```csharp
.app.MapGet（"/hello-get", () => "[GET] Hello World！");
.app.MapPost（"/hello-post", () => "[POST] Hello World！");
.app.MapPut（"/hello-put", () => "[PUT] Hello World！");
.app.MapDelete（"/hello-delete", () => "[DELETE] Hello World！");
```

当我们希望提请您注意代码块的特定部分时，相关行或项目以注释提醒：

```csharp
if (app.Environment.IsDevelopment())
{
    // insert below two lines
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

**粗体**, 表示您在屏幕上看到的新术语、重要单词或单词。例如，菜单或对话框中的单词以粗体显示。
下面是一个示例："打开 Visual Studio 2022，然后从主屏幕单击 **创建新项目** "。

任何命令行输入或输出如下示例：
```bash
$ dotnet new webapi -minimal -o Chapter01
```

提示或者重要标识

> 像这样以引用格式出现


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/minimal-apis)
