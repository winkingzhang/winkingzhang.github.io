---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/master-minimal-apis/chapter-10
layout: single
classes: wide
sidebar:
  nav: "master_minimal_apis"
---


## 评估和衡量 Minimal API 的性能

本章旨在理解创建 Minimal APIs 框架的动机之一。

本章将提供一些明显的数据和示例，说明如何使用传统方法测量 ASP.NET 6 应用程序的性能，
以及如何使用 Minimal API 方法测量其性能。

性能是任何正常运行的应用程序的关键；然而，它常常被忽视。

一个高性能且可扩展的应用程序不仅取决于我们的代码，还取决于开发框架。
如今，我们已经从 .NET 完整框架和 .NET Core 发展到了 .NET，并且可以开始体会到新版本.NET 在性能上的提升。
不仅是因为引入了新功能和框架的清晰性，更主要的是因为框架已经被完全重写和改进，
具有许多使其快速且与其他语言相比极具竞争力的特性。

在本章中，我们将通过比较其代码与传统开发的相同代码来评估 Minimal API 的性能。
我们将了解如何利用 `BenchmarkDotNet` 框架评估 Web 应用程序的性能，该框架在其他应用场景中也可能有用。

通过 Minimal APIs，我们有了一个新的简化框架，通过省略一些在 ASP.NET 中常见的组件来帮助提高性能。

本章将涉及以下主题：

* Minimal APIs 的改进
* 使用负载测试探索性能
* 使用 `BenchmarkDotNet` 对 Minimal APIs 进行基准测试

## 技术要求

许多系统可以帮助我们测试框架的性能。

我们可以测量一个应用程序相对于另一个应用程序每秒可以处理多少请求，假设应用程序负载相同。在这种情况下，我们谈论的是负载测试。

为了测试 Minimal APIs，我们需要安装 k6，这是我们将用于进行测试的框架。

我们将在仅运行 .NET 应用程序的 Windows 机器上进行负载测试。

要安装 k6，可以执行以下操作之一：

- 如果使用 Chocolatey 包管理器（[https://chocolatey.org/](https://chocolatey.org/)），
可以使用以下命令安装非官方的 k6 包：<br />
`choco install k6`
- 如果使用 Windows 包管理器（[https://github.com/microsoft/winget-cli](https://github.com/microsoft/winget-cli)），
可以使用以下命令从 k6 清单中安装官方包：<br />
`winget install k6`
- 也可以使用 Docker 测试在互联网上发布的应用程序： <br />
`docker pull loadimpact/k6`
- 或者像我们一样，在 Windows 机器上安装 k6 并从命令行启动所有操作。
可以从以下链接下载 k6：[https://dl.k6.io/msi/k6-latest-amd64.msi](https://dl.k6.io/msi/k6-latest-amd64.msi)。

在本章的最后部分，我们将测量调用 API 的 HTTP 方法的持续时间。
我们将站在系统的末端，就好像 API 是一个黑盒一样，测量反应时间。
我们将使用 `BenchmarkDotNet` 工具 —— 要在项目中包含它，我们需要引用其 NuGet 包：

```shell
dotnet add package BenchmarkDotNet
```

本章的所有代码示例都可以在本书的 GitHub 存储库中找到，链接如下：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter10](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter10)

## Minimal APIs 的改进

Minimal APIs 的设计不仅是为了提高 API 的性能，还为了更好的代码便利性以及与其他语言的相似性，以吸引来自其他平台的开发人员。

从 .NET 框架的角度来看，性能有所提高，因为每个版本都有令人难以置信的改进，并且从应用程序管道简化的角度来看也是如此。
让我们详细看看哪些功能没有被移植，以及是什么提高了这个框架的性能。

Minimal APIs 执行管道省略了以下功能，这使得框架更轻量级：

* 过滤器，如 `IAsyncAuthorizationFilter`、`IAsyncActionFilter`、`IAsyncExceptionFilter`、`IAsyncResultFilter` 和 `IasyncResourceFilter`
* 模型绑定
* 表单绑定，如 `IFormFile`
* 内置验证
* 格式化程序
* 内容协商
* 一些中间件
* 视图渲染
* `JsonPatch`
* `OData`
* API 版本控制

#### .NET 6 中的性能改进

.NET 在每个版本中都在不断提高其性能。在最新版本的框架中，已经报告了相对于以前版本的改进。
可以在以下链接中找到 .NET 6 新增功能的完整总结：
[https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/)

## 使用负载测试探索性能

如何评估 Minimal APIs 的性能呢？有许多方面需要考虑，在本章中，我们将尝试从它们能够支持的负载的角度来解决这个问题。
我们决定采用一个工具 —— `k6`，它对 Web 应用程序进行负载测试，并告诉我们一个 Minimal API 每秒可以处理多少请求。

正如其创建者所描述的，k6 是一个开源负载测试工具，使工程团队的性能测试变得容易和高效。
该工具是免费的、以开发人员为中心且可扩展的。使用 `k6`，可以测试系统的可靠性和性能，并更早地发现性能回归和问题。
这个工具将帮助构建具有弹性和高性能的可扩展应用程序。

在我们的案例中，我们希望使用该工具进行性能评估，而不是负载测试。在负载测试期间应该考虑许多参数，
但我们将只关注 `http_reqs` 指标，它表示系统正确处理了多少请求。

我们同意 `k6` 创建者关于我们测试目的的观点，即 *性能* 和 *综合监控* 。

### 用例

k6 的用户通常是开发人员、QA 工程师、SDETs 和 SREs。
他们使用 k6 测试 API、微服务和网站的性能和可靠性。
常见的 k6 用例包括以下内容：

* **负载测试**：k6 针对最小资源消耗进行了优化，专为运行高负载测试（峰值、压力和浸泡测试）而设计。
* **性能和综合监控**：使用 k6，可以用小负载运行测试，以持续验证生产环境的性能和可用性。
* **混沌和可靠性测试**：k6 提供了一个可扩展的架构。可以使用 k6 模拟流量作为混沌实验的一部分，或者从 k6 测试中触发它们。

然而，如果我们想从刚刚描述的角度评估应用程序，我们必须做出几个假设。
当进行负载测试时，它通常比我们在本节中进行的测试要复杂得多。
当应用程序受到大量请求的冲击时，并非所有请求都会成功。
如果只有极少数的响应失败，我们可以说测试成功通过。
特别是，我们通常将结果的 95 或 98 百分位数作为得出测试数字的统计数据。

在这种背景下，我们可以按如下方式进行逐步负载测试：在 “ramp up” 阶段，
系统将在大约 15 秒内将 **虚拟用户（VU）** 负载从 0 增加到 50。
然后，我们将在接下来的 60 秒内保持用户数量稳定，最后在最后 15 秒内将负载降为零个虚拟用户。

每个新编写的测试阶段都在 JavaScript 文件的 `stages` 部分中表示。因此，测试是在简单的经验评估下进行的。

首先，我们为 ASP.NET Web API 和 Minimal API 创建三种类型的响应：

* 纯文本。
* 针对调用的非常小的 JSON 数据 —— 数据是静态的且始终相同。
* 在第三种响应中，我们使用 HTTP POST 方法将 JSON 数据发送到 API。
对于 Web API，我们检查对象的验证，而对于 Minimal API，由于没有验证，我们返回接收到的对象。

以下代码将用于比较 Minimal API 和传统方法之间的性能：

**Minimal API**

```csharp
app.MapGet("text-plain", () => Results.Content("response"))
  .WithName("GetTextPlain");

app.MapPost("validations", (ValidationData validation) => { });

app.MapGet("jsons", () =>
{
    var response = new[]
    {
        new PersonData { 
            Name = "Andrea", 
            Surname = "Tosato", 
            BirthDate = new DateTime(2022, 01, 01) 
        },
        new PersonData { 
            Name = "Emanuele", 
            Surname = "Bartolesi", 
            BirthDate = new DateTime(2022, 01, 01) 
        },
        new PersonData { 
            Name = "Marco", 
            Surname = "Minerva", 
            BirthDate = new DateTime(2022, 01, 01) 
        }
    };
    return Results.Ok(response);
})
.WithName("GetJsonData");
```

**传统方法**

对于传统方法，设计了三个不同的控制器，如下所示：

```csharp
[Route("text-plain")]
[ApiController]
public class TextPlainController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Content("response");
    }
}

[Route("validations")]
[ApiController]
public class ValidationsController : ControllerBase
{
    [HttpPost]
    public ActionResult Post(ValidationData data)
    {
        return Ok(data);
    }
}

public class ValidationData
{
    [Required]
    public int Id { get; set; }
    [Required]
    [StringLength(100)]
    public string Description { get; set; }
}

[Route("jsons")]
[ApiController]
public class JsonsController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        var response = new[]
        {
            new PersonData { 
                Name = "Andrea", 
                Surname = "Tosato", 
                BirthDate = new DateTime(2022, 01, 01) 
            },
            new PersonData { 
                Name = "Emanuele", 
                Surname = "Bartolesi", 
                BirthDate = new DateTime(2022, 01, 01) 
            },
            new PersonData { 
                Name = "Marco", 
                Surname = "Minerva", 
                BirthDate = new DateTime(2022, 01, 01) 
            }
        };
        return Ok(response);
    }
}

public class PersonData
{
    public string Name { get; set; }
    public string Surname { get; set; }
    public DateTime BirthDate { get; set; }
}
```

在下一节中，我们将定义一个 `options` 对象，在其中定义这里描述的执行阶段。我们定义所有条款以认为测试通过。
作为最后一步，我们编写实际的测试，它只是根据测试调用 HTTP 端点（使用 `GET` 或 `POST`）。

### 编写 k6 测试

让我们为上一节中描述的每个场景创建一个测试：

```javascript
import http from "k6/http";
import { check } from "k6";

export let options = {
    summaryTrendstats: ["avg", "p(95)"],
    stages: [
        // 在 10 秒内线性地将虚拟用户从 1 增加到 50
        { target: 50, duration: "10s" },
        // 在接下来的 1 分钟内保持 50 个虚拟用户
        { target: 50, duration: "1m" },
        // 在最后 15 秒内线性地将虚拟用户从 50 减少到 0
        { target: 0, duration: "15s" }
    ],
    thresholds: {
        // 我们希望所有 HTTP 请求持续时间的第 95 百分位数小于 500ms
        "http_req_duration": ["p(95)<500"],
        // 基于我们定义的自定义指标的阈值，用于跟踪应用程序故障
        "check_failure_rate": [
            // 全局故障率应小于 0.01
            "rate<0.01",
            // 如果故障率超过 0.05，则提前中止测试
            { threshold: "rate<=0.05", abortOnFail: true }
        ]
    }
};

export default function () {
    // 执行 HTTP GET 调用
    let response = http.get("http://localhost:7060/jsons");
    // 如果任何指定条件失败，check() 返回 false
    check(response, {
        "status is 200": (r) => r.status === 200
    });
}
```

在上述 `JavaScript` 文件中，我们使用 k6 语法编写了测试。
我们定义了选项，例如测试的评估阈值、要测量的参数以及测试应该模拟的阶段。
一旦我们定义了测试的选项，我们只需要编写代码来调用我们感兴趣的 API —— 在我们的案例中，
我们定义了三个测试来调用我们想要评估的三个端点。

### 运行 k6 性能测试

现在我们已经编写了测试性能的代码，让我们运行测试并生成测试统计信息。

我们将报告收集到的测试的所有一般统计信息：

1、 首先，我们需要启动 Web 应用程序来运行负载测试。
让我们同时启动 ASP.NET Web API 应用程序和 Minimal API 应用程序。
我们公开 URL，包括 HTTPS 和 HTTP 协议。

2、 将 shell 移动到根文件夹，并在两个不同的 shell 中运行以下两个命令：
```shell
dotnet ./MinimalAPI.Sample/bin/Release/net6.0/MinimalAPI.Sample.dll \
    --urls=https://localhost:7059/;http://localhost:7060/
dotnet ./ControllerAPI.Sample/bin/Release/net6.0/ControllerAPI.Sample.dll \
    --urls="https://localhost:7149/;http://localhost:7150/"
```

3、 现在，我们只需要为每个项目运行三个测试文件。

- 这是用于基于控制器的 Web API 的测试：<br />
`k6 run ./K6/Controllers/json.js --summary-export=./K6/results/controller-json.json`
- 这是用于 Minimal API 的测试： <br />
`k6 run ./K6/Minimal/json.js --summary-export=./K6/results/minimal-json.json`

以下是结果。

对于传统开发模式下具有纯文本内容类型的测试，每秒处理的请求数为 1,547：

![Figure 10.1 – The load test for a controller-based API and plain text](/assets/images/master-minimal-apis/Figure_10.1_B17902.jpg)

对于传统开发模式下具有 JSON 内容类型的测试，每秒处理的请求数为 1,614：

![Figure 10.2 – The load test for a controller-based API and JSON result](/assets/images/master-minimal-apis/Figure_10.2_B17902.jpg)

对于传统开发模式下具有 JSON 内容类型和模型验证的测试，每秒处理的请求数为 1,602：

![Figure 10.3 – The load test for a controller-based API and validation payload](/assets/images/master-minimal-apis/Figure_10.3_B17902.jpg)

对于 Minimal API 开发模式下具有纯文本内容类型的测试，每秒处理的请求数为 2,285：

![Figure 10.4 – The load test for a minimal API and plain text](/assets/images/master-minimal-apis/Figure_10.4_B17902.jpg)

对于 Minimal API 开发模式下具有 JSON 内容类型的测试，每秒处理的请求数为 2,030：

![Figure 10.5 – The load test for a minimal API and JSON result](/assets/images/master-minimal-apis/Figure_10.5_B17902.jpg)

对于 Minimal API 开发模式下具有 JSON 内容类型且无验证有效负载的测试，每秒处理的请求数为 2,070：

![Figure 10.6 – The load test for a minimal API and no validation payload](/assets/images/master-minimal-apis/Figure_10.6_B17902.jpg)

在下图中，我们展示了三个测试功能的比较，报告了具有相同功能的处理请求数：

![Figure 10.7 – The performance results](/assets/images/master-minimal-apis/Figure_10.7_B17902.jpg)

正如我们可能预期的那样，Minimal APIs 比基于控制器的 Web APIs 快得多。

差异约为 30%，这可不是一个小数字。显然，如前所述，Minimal APIs 为了优化性能省略了一些功能，最明显的是数据验证。

在示例中，有效负载非常小，差异不是很明显。

随着有效负载和验证规则的增加，两个框架之间的速度差异只会增加。

我们已经看到了如何使用负载测试工具测量性能，然后评估在相同数量的机器和连接用户的情况下每秒可以处理多少请求。

我们还可以使用其他工具来了解 Minimal APIs 对性能的积极影响。

## 使用 BenchmarkDotNet 对 Minimal APIs 进行基准测试

`BenchmarkDotNet` 是一个框架，允许测量编写的代码并比较不同版本或使用不同 .NET 框架编译的库之间的性能。

这个工具用于计算执行任务所需的时间、使用的内存和许多其他参数。

我们的案例是一个非常简单的场景。我们想要比较两个针对相同版本的 .NET Framework 编写的应用程序的响应时间。

我们如何进行这种比较呢？我们获取一个 `HttpClient` 对象，并开始调用我们也为负载测试用例定义的方法。

因此，我们将获得两个方法之间的比较，这两个方法利用相同的 `HttpClient` 对象并调用具有相同功能的方法，
但一个是使用 ASP.NET Web API 和传统控制器编写的，而另一个是使用 Minimal APIs 编写的。

`BenchmarkDotNet` 帮助将方法转换为基准测试，跟踪它们的性能，并共享可重现的测量实验。

在幕后，它执行了许多神奇的操作，由于 `perfolizer` 统计引擎，保证了可靠和精确的结果。
`BenchmarkDotNet` 可以防止常见的基准测试错误，并在基准测试设计或获得的测量结果有问题时发出警告。
该库已被超过 6,800 个项目采用，包括 .NET Runtime，并得到了 .NET Foundation 的支持
（[https://benchmarkdotnet.org/](https://benchmarkdotnet.org/)）。

### 运行 BenchmarkDotNet

我们将编写一个类，代表调用两个 Web 应用程序的 API 的所有方法。
让我们充分利用启动功能并准备我们将通过 POST 发送的对象。
标记为 `[GlobalSetup]` 的函数在运行时不会计算，这有助于我们准确计算从调用到 Web 应用程序响应之间的时间：

1、 在 `Program.cs` 中注册实现 `BenchmarkDotNet` 的所有类：

```csharp
BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly)
    .Run(args);
```

在上述代码片段中，我们注册了当前程序集，其中包含了性能计算所需的所有函数。
标记为 `[Benchmark]` 的方法将被反复执行，以确定平均执行时间。

2、 应用程序必须在发布模式下编译，并且尽可能在生产环境中进行：

```csharp
namespace DotNetBenchmarkRunners
{
    [SimpleJob(RuntimeMoniker.Net60, baseline: true)]
    [JsonExporter]
    public class Performances
    {
        private readonly HttpClient clientMinimal = 
            new HttpClient();
        private readonly HttpClient clientControllers = 
            new HttpClient();
        private readonly ValidationData data = 
            new ValidationData()
            {
                Id = 1,
                Description = "Performnce"
            };

        [GlobalSetup]
        public void Setup()
        {
            clientMinimal.BaseAddress = 
                new Uri("https://localhost:7059");
            clientControllers.BaseAddress = 
                new Uri("https://localhost:7149");
        }

        [Benchmark]
        public async Task Minimal_Json_Get() => 
            await clientMinimal.GetAsync("/jsons");

        [Benchmark]
        public async Task Controller_Json_Get() => 
            await clientControllers.GetAsync("/jsons");

        [Benchmark]
        public async Task Minimal_TextPlain_Get() => 
            await clientMinimal.GetAsync("/text-plain");

        [Benchmark]
        public async Task Controller_TextPlain_Get() => 
            await clientControllers.GetAsync("/text-plain");

        [Benchmark]
        public async Task Minimal_Validation_Post() => 
            await clientMinimal.PostAsJsonAsync("/validations", data);

        [Benchmark]
        public async Task Controller_Validation_Post() => 
            await clientControllers.PostAsJsonAsync("/validations", data);
    }

    public class ValidationData
    {
        public int Id { get; set; }
        public string Description { get; set; }
    }
}
```

3、 在启动基准应用程序之前，先启动 Web 应用程序：

```shell
# Minimal API 应用程序
dotnet ./MinimalAPI.Sample/bin/Release/net6.0/MinimalAPI.Sample.dll \
    --urls="https://localhost:7059/;http://localhost:7060/"

# 基于控制器的应用程序
dotnet ./ControllerAPI.Sample/bin/Release/net6.0/ControllerAPI.Sample.dll \
    --urls=https://localhost:7149/;http://localhost:7150/
```

通过启动这些应用程序，将执行各种步骤，并提取带有我们在此报告的时间轴的总结报告：

```shell
dotnet ./DotNetBenchmarkRunners/bin/Release/net6.0/DotNetBenchmarkRunners.dll \
    --filter *
```

对于执行的每个方法，都会报告平均值或平均执行时间。

| 方法                         | 平均值      | 误差       | 标准偏差     | 比率   |
|----------------------------|----------|----------|----------|------|
| Minimal Json Get           | 545.0 μs | 5.23 ps  | 4.89 μs  | 1.00 |
| Controller JsonGet         | 840.1 μs | 13.29 ps | 11.78 μs | 1.00 |
| Minimal TextPlain Get      | 596.8 μs | 3.84 μs  | 3.20 μs  | 1.00 |
| Controller TextPlain Get   | 891.7 μs | 7.50 μs  | 9.21 μs  | 1.00 |
| Minimal Validation Post    | 596.8 μs | 5.95 μs  | 5.28 μs  | 1.00 |
| Controller Validation Post | 905.6 μs | 5.18 μs  | 4.59 μs  | 1.00 |

在下表中，`Error` 表示由于测量误差，平均值可能会有多大的变化。
最后，标准偏差（`StdDev`）表示与平均值的偏差。时间以 `μs` 为单位，如果没有相应的仪器，很难通过经验测量。

## 总结

在本章中，我们使用两种截然不同的方法比较了 Minimal APIs 与传统方法的性能。
Minimal APIs 并非仅为性能而设计，仅基于此进行评估是一个糟糕的起点。
表 10.1 表明 Minimal APIs 和传统 ASP.NET Web API 应用程序的响应之间存在很大差异。

测试在相同的机器上使用相同的资源进行。我们发现 Minimal APIs 的性能比传统框架高出约 30%。

我们已经了解了如何测量应用程序的速度 —— 这对于了解应用程序是否能够承受负载以及它可以提供什么样的响应时间很有用。
我们也可以在关键代码的小部分上利用这一点。最后需要注意的是，测试的应用程序实际上非常简单。
在 ASP.NET Web API 应用程序中应该评估的验证部分几乎无关紧要，因为只需要考虑两个字段。
随着 Minimal APIs 中省略的组件数量增加，两个框架之间的差距也会增大。

<br/><br/><br/><br/>
&gt;  [返回扉页](/books/master-minimal-apis)
