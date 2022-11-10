---
title:  "New Relic Logs 深入浅出"
header:
  teaser: "/assets/images/teaser-2.jpg"
categories:
  - CSharp
tags:
  - NewRelic
  - Serilog
  - Logs
  - CSharp
excerpt: >
  习惯了 LogEntries Structure Logs 的开发切换到 New Relic 会发现很不适应，
  本篇结合博主的经验，来分析一下 New Relic Logs 从入门到精通的捷径……
---

## 开篇

应用程序开发随着上云和微服务这两个变革开始慢慢将应用程序日志变成了一个必要基本需求，对应市场上也推出了一些重量级的日志管理和服务，
比较出名的除了 ELK 自建，其他大部分都是云服务，陆陆续续也接触过的好几个服务：
* [New Relic](https://www.newrelic.com)：知名 APM 云服务，以监控为基础扩展相关联功能，
优势是监控和告警，后陆续加入日志管理和 DevOps 相关支持。
* [Application Insight](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)：
微软提供的 APM 工具集，内部带有日志收集分析，多年来不温不火，能用但也没多大用处，更多的用途是对 Azure 资源监控。
* [Amazon CloudWatch](https://www.sumologic.com)：AWS 的监控和管理平台，早期主要用途是 AWS 资源，
后来逐渐扩张到任意应用，很多监控和日志领域的事实标准都来自 CloudWatch。
* [Sumo Logic](https://www.sumologic.com)：一站式集监控，分析，排错，应用程序及其网路数据可视化的服务，
优点是对 Structured logging 支持比较好。
* [LogEntries](https://logentries.com)：Rapid7 旗下的日志收集和分析平台，查询语言比较简单，查询速度一流。


## New Relic 日志简介

> **备注**： 部分内容摘抄自 NewRelic 官方网站，仅仅代表个人观点。

New Relic 提供一个快速、可扩展的日志管理平台，可以很容易的将日志数据和其他监控指标和平台数据集成到一处，
以便后续深度分析并可视化应用程序和系统性能，分析线上问题，以此降低 **服务恢复时间 (mean-time-to-resolve, MTTR)**。 

而 New Relic 的核心优势就是这个集成，可以大大减少上下文切换，举个例子，假设当前的任务是调查线上 webapi 响应慢，这是一个很大的话题，
入手点也很多，在 New Relic 平台下大体的 best practice 是这样的：

1. 在 APM 下查询这个 webapi 的 Transaction 数据，判断一下是否是因为流量激增导致，关键指标是 Throughput (rpm) 和 Error rate
2. 查询 Transaction traces 中的 slowest sample，分析并查找最慢部分
3. 在 APM 下的日志中查询相关的日志，通常使用 TraceId 来匹配（也可以是其他自定义的关键特性），结合 Transaction 数据确定问题位置
4. 在 Logs 下搜索相关的错误日志（例如异常，特定的错误提示），判断是否内部错误

在这整个过程中，不需要专门去查找上下文的标识，在 New Relic 界面可以很容易的切换而不必担心错乱。

#### NewRelic 日志分类

New Relic 在内部将任何日志都映射成 `Log`，但是从公开接口可以推测有两种类型的日志，在文档中也有相应的说明，分别是：
- Logs in context （上下文日志）
- Forward logs （转发日志）

###### 上下文日志 (Logs in context)

`上下文日志` 通常是由较新的 New Relic agent 自动收集并通过内部接口上报的一类日志，无须安装和维护第三方软件，
这也是 New Relic 官方推荐的首选模式来自动处理日志。
它们的特点是自动插入 `NewRelic.LinkingMetadata` 来关联当前 Transaction 的上下文信息，通过附加如下的特性标签：
```
span.id, trace.id, hostname, entity.guid, entity.name
```

实际效果如下图，可以在 APM 的仪表面板页面看到与当前上下文相关的日志信息。
![APM logs in context](/assets/images/newrelic/logs_screenshot-full_apm-logs-in-context-in-ui.png)

需要注意的是，当前并不是所有 agent 都支持自动上报上下文日志，New Relic 目前支持语言和文档链接列表如下：
- [Go logs in context](https://docs.newrelic.com/docs/logs/logs-context/configure-logs-context-go) for agent `v3.18.0 or higher`
- [Java logs in context](https://docs.newrelic.com/docs/logs/logs-context/java-configure-logs-context-all) for agent `v7.6.0 or higher`
- [.NET logs in context](https://docs.newrelic.com/docs/logs/logs-context/net-configure-logs-context-all) for agent `v9.7.0.0 or higher`
- [Node.js logs in context](https://docs.newrelic.com/docs/logs/logs-context/configure-logs-context-nodejs) for agent `v8.11.0 or higher`
- [Python logs in context](https://docs.newrelic.com/docs/logs/logs-context/configure-logs-context-python) for agent `v7.12.0.176`
- [Ruby logs in context](https://docs.newrelic.com/docs/logs/logs-context/configure-logs-context-ruby) for agent `v8.6.0 or higher`

因此，对于当前不受支持的语言，也可以通过手动处理来将任意日志变成上下文日志，核心思路如下：
- 配置 New Relic agent 支持分布式跟踪(distributed tracing)
- 通过 agent API 获取 `NewRelic.LinkingMetadata` 并嵌入日志
- 选择合适的 Forward 将日志转发到 New Relic

###### 转发日志 (Forward logs)

顾名思义，转发日志就是集成插件或者第三方软件实现转发到 New Relic 的日志。这里 New Relic 提供了非常好的支持，
从系统日志，应用程序日志，解析转换处理日志到发送日志，都有非常丰富的生态来支撑。下图是 New Relic 官方给出的一个概览：

![Options to forward logs to New Relic](/assets/images/newrelic/logs_diagram_log-forward-options.png)

实际使用的时候这里反倒是很难抉择，官方也给出了一个建议方案列表

| 使用场景                     | 日志转发方案                                                                                                                                               |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| 从已写入本地磁盘的文件收集日志          | 首选 `Infrastructure agent`，然后是 `Fluent Bit`、`Fluentd`、Logstash 甚至 `syslog/TCP`                                                                        |
| 从云平台日志转发服务收集日志           | 参考 New Relic 针对 AWS， GCP，Azure 和 Heroku 提供的解决方案                                                                                                      |
| 从容器应用程序，标准应用程序或者k8s 收集日志 | 推荐使用 k8s plugin 或者 infrastructure agent                                                                                                              |
| 从应用自身或者承载应用的主机收集日志       | 直接使用 APM agent 或者 infrastructure agent                                                                                                               |
| 其他情况                     | - 使用 New Relic [Log API](https://docs.newrelic.com/docs/logs/log-api/introduction-log-api) 通过 HTTP 转发 <br/> - 使用 syslog 协议通过 TCP 端口转发 <br/> - 使用其他选项 |

正是方法太多，这里先按下不表，后面针对项目选择的场景做一些简单的示例。

#### 查询和分析日志

日志收集转储到 New Relic 上了， 如何使用呢？

此时就需要 Logs UI，New Relic将所有主要功能集中到一处，入口地址是 [one.newrelic.com](https://one.newrelic.com)
(这里如果是欧盟数据中心的话，地址则是 [one.eu.newrelic.com](https://one.eu.newrelic.com) )，左侧 `Logs` 菜单点击进去就是日志界面（如下图）。
默认界面是所有日志，指的是当前账号下的所有日志，如果有多个应用程序共有一个账号的话，
默认情况会把所有日志按照 `timestamp` 正序显示一个列表，最后一行则是最新的日志（向下滚动可以获取新的日志）。
这里有点坑的是 New Relic 仅仅显示 `timestamp` 和 `message` 字段，其他的字段需要点一下具体日志条目进入详情才能在右侧弹出的界面看到。

![New Relic logs UI](/assets/images/newrelic/logs_screenshot-full_logs-ui.png)

###### 属性和查询

正如上一小节提到的日志详情界面看到的，日志是有很多的特性字段的，肯定出现的字段却只有几个，
分别是 `timestamp`， `message`， `level`，`newrelic.source`， 其他特性字段来自三个可能：

- 来自 APM 的上下文特性字段 (`entity.guid`, `entity.name`, `span.id` 和 `trace.id`)
- 来自应用程序或者转发程序增加的特性字段
- 来自 New Relic [parsing](https://docs.newrelic.com/docs/logs/ui-data/parsing) 的特性字段

> **注意**：New Relic 日志的特性字段仅仅支持 Key:Value 形式的键值对，并且 Key 和 Value 都是纯字符串，因此不能支持 structured log

那么 New Relic 的日志查询也就以此为基础建立的，其查询规则也是基于 key:value 形式的 （本质是 Lucene query language 一个子集）,
这里 `key` 对应查询范围， `value` 给定查询条件，如果仅仅给定 `value`，那么就意味着在 `message` 这个特性字段里查找。

举个例子，假如已经将 http statuscode 写入日志的 status 字段，想要查询 4xx 和 5xx 的日志，
查询条件就是： `status: "4*"` 和 `status: "5*"`。

细说起来日志查询规则也是不少的，这里就不在一一列举，感兴趣的可以看到这个文档：<br />
[Query syntax for logs](https://docs.newrelic.com/docs/logs/ui-data/query-syntax-logs)


#### 日志和数据安全

日志数据安全一直是困扰企业安全团队的重要难题，同时也是开发人员最容易犯错的一点，
个中缘由就是日志分散且灵活，开发人员很容易忽视日志中的敏感数据，典型的一些事故类似：

- 将数据库连接字符串输出到日志
- 将客户邮箱输出到日志
- 将信用卡卡号写入日志
- 将生日信息输出到日志
- 其他隐私数据（地址，媒体账号等等）

举个例子，一把将上面的戒律全部破掉 ;)
```json
{
  "message": "The credit card number 4321-5678-9876-2345 belongs to user user@email.com (born on 01/02/2003) with SSN 123-12-1234",
  "creditCardNumber": "4321-5678-9876-2345",
  "email": "user@email.com",
  "birthday": "01/02/2003",
  "ssn": "123-12-1234",
  "department": "sales",
  "serviceName": "loginService",
  "dbHost": "fakeserver.postgres.database.azure.com"
}
```

New Relic 提供了[日志混淆功能](https://docs.newrelic.com/docs/logs/ui-data/obfuscation-ui)，但可惜的是需要马内才能启用，不差钱的老板可以随便上。
大部分的开发规范则要求在日志收集之前就要做到脱敏，因此需要对日志收集工具做一些改动。

对于 .NET 项目比较流行的工具是 Serilog，要想处理敏感数据，只需要提供自己的 `IDestructuringPolicy` 的实现并配置上就可以了。

## 集成 NewRelic 日志

前文讲了这么多的相关知识，接下来让我们真刀实枪的集成一下 New Relic 的各个方案吧。

#### 自动处理日志

首先，让我们再一次翻一下 [New Relic 上下文日志的 .NET 文档](https://docs.newrelic.com/docs/logs/logs-context/net-configure-logs-context-all)， 这里提供了罗列了对 .NET 生态下的多种日志框架的支持，
让我们从简单的做起，仅仅依赖 .NET Agent，自动处理上下文日志转发，配合日常使用的 Serilog 来快速演示一下。

> 注意：需要 Serilog 2.0+(.NET Framework), 2.5+(.NET Core)，还有 .NET Agent v9.7.0+

>  示例代码环境要求 .NET 6.0，并且只能在 window 和 linux（包括容器） 才能正常收集上下文日志数据

这个方案实质上并没有多少代码，这里从 webapi 模板开始：
```bash
$ dotnet new webapi -n dotnet-newrelic-logs-context
```
为了简化日志处理同时支持结构化日志，需要引入日志管理框架，这里以 Serilog 为例：
```bash
$ dotnet add package Serilog.AspNetCore
```
还需要几句简单的初始化和配置，这样本地运行起来就可以在应用程序的控制台里看到日志信息：
```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog();
```

接下来，为了方便调试，还需要修改工程文件，在 Debug 模式下引入 `NewRelic.Agent` ，
由于前文已经引入了 Serilog.AspNetCore ，这里工程文件的引用部分看起来像这样：
```xml
<ItemGroup>
  <PackageReference Include="NewRelic.Agent" Version="10.3.0" Condition=" '$(Configuration)' == 'Debug' " />
  <PackageReference Include="Serilog.AspNetCore" Version="6.0.1" />
  <PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
</ItemGroup>
```

然后执行的时候设置好 New Relic 要求的环境变量（当然也可以修改 `newrelic.config` 配置文件）就可以正常的往 New Relic发送数据了，
不过每次修改环境变量挺麻烦，有个简便可靠的方法是修改 `Properties/launchSettings.json`，这样只要每次输入敏感数据，其他的改一次就行，
这里要注意的是调试 window 和 linux 的配置略有不同，因此我创建了两个 profile 来保存：

```json
"webapi(linux)": {
  "commandName": "Project",
  "dotnetRunMessages": true,
  "launchBrowser": true,
  "launchUrl": "swagger",
  "applicationUrl": "https://localhost:7198;http://localhost:5067",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development",
    "CORECLR_ENABLE_PROFILING": "1",
    "CORECLR_PROFILER": "{36032161-FFC0-4B61-B559-F6C5D41BAE5A}",
    "CORECLR_NEWRELIC_HOME": "$(ProjectDir)bin/$(Configuration)/net6.0/newrelic",
    "CORECLR_PROFILER_PATH": "$(ProjectDir)bin/$(Configuration)/net6.0/newrelic/libNewRelicProfiler.so",
    "NEW_RELIC_DISTRIBUTED_TRACING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_FORWARDING_MAX_SAMPLES_STORED": "10000",
    "NEW_RELIC_APPLICATION_LOGGING_LOCAL_DECORATING_ENABLED": "false",
    "NEW_RELIC_APP_NAME": "dotnet-newrelic-log-context",
    "NEW_RELIC_LICENSE_KEY": "real-license-key"
  }
},
"webapi(windows)": {
  "commandName": "Project",
  "dotnetRunMessages": true,
  "launchBrowser": true,
  "launchUrl": "swagger",
  "applicationUrl": "https://localhost:7198;http://localhost:5067",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development",
    "CORECLR_ENABLE_PROFILING": "1",
    "CORECLR_PROFILER": "{36032161-FFC0-4B61-B559-F6C5D41BAE5A}",
    "CORECLR_NEWRELIC_HOME": "$(ProjectDir)bin/$(Configuration)/net6.0/newrelic",
    "CORECLR_PROFILER_PATH": "$(ProjectDir)bin/$(Configuration)/net6.0/newrelic/NewRelic.Profiler.dll",
    "NEW_RELIC_DISTRIBUTED_TRACING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED": "true",
    "NEW_RELIC_APPLICATION_LOGGING_FORWARDING_MAX_SAMPLES_STORED": "10000",
    "NEW_RELIC_APPLICATION_LOGGING_LOCAL_DECORATING_ENABLED": "false",
    "NEW_RELIC_APP_NAME": "dotnet-newrelic-log-context",
    "NEW_RELIC_LICENSE_KEY": "real-license-key"
  }
}
```

本地跑起来，在 swagger 里调用几次 API，在等上大约30秒就能在 New Relic APM logs 下面看到刚刚经历的日志信息了，
如下图所示，这里可以看到 `NewRelic.LinkingMetadata` 数据，但是，整个日志只有固定的 message 部分和可能的异常信息，其他的就没有了。

![log in context](/assets/images/newrelic/log-in-context.png)

上面都是在本地运行，现在大部分应用都会打包成容器镜像（docker image）在 k8s 上运行，因此还需要对 `Dockerfile` 做一些改造。
这里重点是两处：
- 使用包管理器安装对应 linux 系统的 New Relic agent（当然也可以使用 nuget 包，那样需要修改下一条的路径）
- 添加环境变量来配置 .NET Core profiler 和 New Relic agent。这里包括启用上下文日志

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final

# Install the agent
RUN apt-get update && \
    apt-get install -y wget ca-certificates gnupg --no-install-recommends && \
    echo 'deb http://apt.newrelic.com/debian/ newrelic non-free' | tee /etc/apt/sources.list.d/newrelic.list && \
    wget -q https://download.newrelic.com/548C16BF.gpg && \
    apt-key add 548C16BF.gpg && \
    apt-get update && \
    apt-get install -y newrelic-dotnet-agent --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

# Enable the agent
ENV CORECLR_ENABLE_PROFILING=1 \
    CORECLR_PROFILER={36032161-FFC0-4B61-B559-F6C5D41BAE5A} \
    CORECLR_NEWRELIC_HOME=/usr/local/newrelic-dotnet-agent \
    CORECLR_PROFILER_PATH=/usr/local/newrelic-dotnet-agent/libNewRelicProfiler.so \
    NEW_RELIC_DISTRIBUTED_TRACING_ENABLED=true \
    NEW_RELIC_APPLICATION_LOGGING_ENABLED=true \
    NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED=true \
    NEW_RELIC_APPLICATION_LOGGING_FORWARDING_MAX_SAMPLES_STORED=10000 \
    NEW_RELIC_APPLICATION_LOGGING_LOCAL_DECORATING_ENABLED=false \
    NEW_RELIC_APP_NAME=dotnet-newrelic-log-context
```

眼尖的看官肯定发现少了一个环境变量 `NEW_RELIC_LICENSE_KEY`，是的，它就是少了这一个变量，因为它是敏感数据，不能直接写到代码里。
实际使用的时候会在启动容器的时候穿进去，如下是示例使用 `docker` 命令启动：

```bash
$ docker run -it \
    -e NEW_RELIC_LICENSE_KEY=your_real_key \
    -p 8080:80 \
    dotnet-newrelic-log-context
```

至此上下文日志就集成完了，总结一下有如下优点

1. 无业务侵入性，基本上不用修改代码。
2. 自动和 Transaction 等数据关联，上下文清晰。
3. 使用场景广泛，只要支持集成 agent 就能干活。

缺点也是很明显的，也简单总结一下：

1. 日志内容简单，仅仅收集消息体和异常信息，需要定义好格式并配合 parsing 才能获取更直观的结构化日志。
2. 可配置项缺少，整体的灵活性就下降了很多
3. 日志与 NewRelic 服务紧绑定，不利于将来扩张

#### 手动处理日志

前文的例子已经搞清楚了 New Relic agent 自动处理上文日志的场景，接下来转入手动处理的场景。

![serilog log flow](/assets/images/newrelic/logs_diagram_dotnet-logs-in-context-serilog.png)

结合上面 Serilog 的日志处理流程图，可以看到 New Relic 日志有两个入口，分别是 APM Endpoint 和 Logs Endpoint，
手动处理的方式都是使用 Logs Endpoint，但是有两种处理模型：
- 实现一个特殊的 Sink，直接调用 Logs API 来发送日志
- 使用任意 Sink，利用第三方 forwarder 调用 Logs API 来转发日志

###### 集成 New Relic Logs API

其核心是实现一个特殊的 Serilog Sink，里面需要调用 New Relic Logs API 发送日志数据，
好在有人已经封装了一个完整的 nuget 包，因此从使用角度来说，这个方案还是挺容易的。

还是从 webapi 模板开始：

```bash
$ dotnet new webapi -n dotnet-newrelic-logs-api
$ dotnet add package Serilog.AspNetCore
```

关键的步骤是添加 sink，并配置好

```bash
$ dotnet add package Serilog.Sinks.NewRelic.Logs
```

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console() // optional, for local debug
    .WriteTo.NewRelicLogs(
        applicationName: Environment.GetEnvironmentVariable("NEW_RELIC_APP_NAME"),
        licenseKey: Environment.GetEnvironmentVariable("NEW_RELIC_LICENSE_KEY"))
    .CreateBootstrapLogger();

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog();
```

至此，基本的集成 logs API 就完成了，只需在环境变量设置 `NEW_RELIC_APP_NAME` 和 `NEW_RELIC_LICENSE_KEY`，
跑起来就能发送日志了，但是没有 `NewRelic.LinkingMetadata` 数据。也就是说，这样发送的日志只能在 New Relic Logs UI界面下看到，
在 APM logs 里是没有的，也就是说此时还不是上下文日志。

要想让这个方案的日志也是上下文日志，需要想办法补充 `NewRelic.LinkingMetadata` 数据，因此要安装 .NET agent 和 一个 Enrich 包：

```bash
$ dotnet add package NewRelic.LogEnrichers.Serilog
```

> 如需要 Debug 模式下引入 `NewRelic.Agent` 请参考前文修改项目文件和 Dockerfile


例行总结一下，直接集成 New Relic logs API的优缺点，先看一下优点：

1. 无业务侵入性，基本上不用修改代码。
2. 能够灵活的处理日志内容（任意增删特性标签）
3. 可以手动关联`NewRelic.LinkingMetadata` 数据转化成上下文日志

缺点有两个：
1. 日志与 NewRelic 服务紧绑定，不利于将来扩张
2. Sink 实现来自三方，有可能引入未知 bug


###### 集成转发日志

让我们再来看看手动处理日志的另一个方案，使用第三方 forwarder 处理日志，
New Relic 官方推荐使用 FluentIO，但这是一个比较重的方案，这里安利一个轻量级的新兴工具，叫 `Vector`，
详细的介绍可以参考 [Vector quickstart](https://vector.dev/docs/setup/quickstart/)。
接下来还需要做一些配置，首先是 Vector 的配置，修改 `vector.toml` 文件，在里面做三件事情：

1. 指定从哪里收集日志
2. 中间如何处理数据（格式化）
3. 如何转发日志

为了简化演示，这里我配置 Vector 从应用程序输出收集日志

```toml
[sources.dotnet]
  # REQUIRED - General
  type = "exec"
  command = ["dotnet", "/app/dotnet-newrelic-log-forward.dll"]
  mode = "streaming"
```

然后配置 transform 将消息转换成 JSON 实体
```toml
# Parse console logs
# See the Vector Remap Language reference for more info: https://vrl.dev
[transforms.parse_logs]
  # REQUIRED
  type = "remap"
  inputs = ["dotnet"]
  source = """
. = parse_json!(.message)
.forward = "vector" 
"""
```
最后调用 New Relic sink 插件将数据发送出去，需要注意的是，vector 插件需要两个重要参数。
`account_id` 和 `license_key`， 这两个值可以在 New Relic 的 API Key管理界面查询到。

```toml
# Print parsed logs to stdout
[sinks.newrelic]
  # REQUIRED
  type = "new_relic"
  inputs = ["dotnet"]
  account_id = "${NEW_RELIC_ACCOUNT_ID}"
  license_key = "${NEW_RELIC_LICENSE_KEY}" # example, no default
  api = "logs"
  # OPTIONAL
  region = "us"
```
另外，为了可以在本地控制台里看到程序的输出，这里额外配置了一个 sink，它会把转化好的数据以纯文本的方式回写到控制台。
```toml
[sinks.console]
  type = "console"
  inputs = [ "dotnet" ]
  target = "stdout"
  
  [sinks.console.encoding]
    codec = "text"
```

把视角切回到示例代码上，还是从 webapi 模板开始：

```bash
$ dotnet new webapi -n dotnet-newrelic-logs-forward
$ dotnet add package Serilog.AspNetCore
```

修改 Serilog 配置部分，这里我实现了一个我自己的 formatter，可以自动将结构化的数据扁平化，这样写入 New Relic 之后可以更好的查询。

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console(formatter: new StructuredJsonFormatter())
    .CreateBootstrapLogger();

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog();
```

因为 Vector 目前仅仅支持 linux 运行，因此我把运行封装到容器里了，里面我们只需要将 vector 启动起来，
vector会读取配置，再启动我们的 dotnet 程序

> 使用 vector exec 模式稳定性不适用于生产环境，请谨慎使用

这里需要一个定制的 `dockerfile`，从 dotnet runtime 派生下来，手动安装 vector

```dockerfile
# The runtime tag version should match the SDK tag version
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final

# Install the dependency packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates tzdata cu
    rm -rf /var/lib/apt/lists/* 

WORKDIR /app

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.vector.dev | bash -s

COPY --from=base /app .
COPY src/dotnet-newrelic-log-forward/entry.sh /app/entry.sh
COPY src/dotnet-newrelic-log-forward/vector.toml /etc/vector/vector.toml

#CMD ["dotnet", "dotnet-newrelic-log-forward.dll"]
CMD ["/root/.vector/bin/vector"]
```

对于转发日志这个方案，总结一下优缺点，先看优点

1. 能够灵活的处理日志内容（任意增删特性标签）
2. 无业务侵入性，基本上不用修改代码。
3. 没有服务绑定，后期可以灵活扩张，甚至迁移到其他日志服务
4. 可以手动关联`NewRelic.LinkingMetadata` 数据转化成上下文日志

缺点有两个：
1. 三方 forwarder 消耗更多资源
2. 三方 forwarder 引入更多复杂配置，引入 bug 风险较高

## 参考文档和链接

- 本文示例源代码： https://github.com/alfie029/dotnet-newrelic-log-tutorials
- New Relic 的 .NET 上下文日志处理： https://docs.newrelic.com/docs/logs/logs-context/net-configure-logs-context-all
- 使用 vector 转发 new relic日志： https://docs.newrelic.com/docs/logs/forward-logs/vector-output-sink-log-forwarding
