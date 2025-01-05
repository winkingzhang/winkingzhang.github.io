---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/minimal-apis/chapter-09
layout: single
classes: wide
sidebar:
  nav: "minimal_apis"
---


## 利用全球化和本地化

在开发应用程序时，考虑多语言支持十分重要；多语言应用程序能够覆盖更广泛的受众。
对于 Web API 来说亦是如此：端点返回的消息（如验证错误）应当本地化，并且服务应当能够处理不同文化并应对时区问题。
在本章中，我们将探讨全球化和本地化，并解释在 Minimal API 中可用于处理这些概念的功能。
所提供的信息和示例将指导我们为服务添加多语言支持并正确处理所有相关行为，以便开发出全球适用的应用程序。

本章将涵盖以下主题：

* 介绍全球化和本地化
* 本地化 Minimal API 应用程序
* 使用资源文件
* 在验证框架中集成本地化
* 为全球化的 Minimal API 添加 UTC 支持


## 技术要求

要遵循本章中的描述，需创建一个 ASP.NET Core 6.0 Web API 应用程序。
参考 [第 1 章 “Minimal API 简介”](/books/minimal-apis/chapter-01) 中的技术要求部分了解创建方法。

若使用控制台、shell 或 Bash 终端创建 API，请记住将工作目录更改为当前章节编号（Chapter09）。

本章所有代码示例可在本书的 GitHub 存储库中找到，地址为：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter09](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter09)

## 介绍全球化和本地化

在考虑国际化时，我们必须处理全球化和本地化，这两个术语看似指相同概念，但实际上涉及不同领域。
全球化是设计能够管理和支持不同文化的应用程序的任务。
本地化是使应用程序适应特定文化的过程，例如为每个支持的文化提供翻译资源。

> 注意：
> 
> 国际化、全球化和本地化通常分别缩写为 I18N、G11N 和 L10N。

与前几章介绍的其他功能一样，全球化和本地化可由 ASP.NET Core 提供的相应中间件和服务处理，
并且在 Minimal API 和基于控制器的项目中工作方式相同。

你可在官方文档中找到关于全球化和本地化的精彩介绍，分别位于 
[https://docs.microsoft.com/dotnet/core/extensions/globalization](https://docs.microsoft.com/dotnet/core/extensions/globalization) 和 
[https://docs.microsoft.com/dotnet/core/extensions/localization](https://docs.microsoft.com/dotnet/core/extensions/localization) 。
在本章其余部分，我们将重点介绍如何在 Minimal API 项目中添加对这些功能的支持，
从而引入一些重要概念并解释如何在 ASP.NET Core 中利用全球化和本地化。

## 本地化 Minimal API 应用程序

要在 Minimal API 应用程序中启用本地化，请按以下步骤操作：

1、 使应用程序可本地化的第一步是通过设置相应选项指定支持的文化，如下所示：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
var supportedCultures = new CultureInfo[] { new("en"), new("it"), new("fr") };
builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    options.SupportedCultures = supportedCultures;
    options.SupportedUICultures = supportedCultures;
    options.DefaultRequestCulture = new RequestCulture(supportedCultures.First());
});
```

在示例中，我们希望支持三种文化 —— 英语、意大利语和法语，因此创建一个 `CultureInfo` 对象数组。

我们定义的是中性文化，即具有语言但不与特定国家或地区关联的文化。
也可使用特定文化，如 `en-US` 或 `en-GB` 来代表特定地区的文化。
例如，`en-US` 指美国流行的英语文化，`en-GB` 指英国流行的英语文化。
这种区别很重要，因为根据场景，可能需要特定国家的信息来正确实现本地化。
例如，若要显示日期，需知道美国的日期格式是 `M/d/yyyy`，而英国是 `dd/MM/yyyy`。
所以在这种情况下，使用特定文化至关重要。
若需支持不同文化间的语言差异，也需使用特定文化，
因为同一个单词在不同国家可能有不同拼写（如美国的 “_color_” 与英国的 “_colour_”）。
不过，对于 Minimal API 场景，使用中性文化就足够了。

2、 接下来配置 `RequestLocalizationOptions`，设置文化并指定在未提供文化信息时使用的默认文化。
同时指定支持的文化和支持的 UI 文化：

* 支持的文化控制依赖文化的函数（如日期、时间和数字格式）的输出。
* 支持的 UI 文化用于选择搜索哪些翻译字符串（来自 `.resx` 文件）。稍后会讨论 `.resx` 文件。

在典型应用程序中，文化和 UI 文化设置为相同值，但如有需要，也可使用不同选项。

3、 现在已配置服务支持全球化，需将本地化中间件添加到 ASP.NET Core 管道，使其能自动设置请求的文化。
使用以下代码实现：

```csharp
var app = builder.Build();
//...
app.UseRequestLocalization();
//...
app.Run();
```

在上述代码中，通过 `UseRequestLocalization()` 将 `RequestLocalizationMiddleware` 添加到 ASP.NET Core 管道，
以设置每个请求的当前文化。此任务通过使用一系列 `RequestCultureProvider` 完成，这些提供程序可从各种来源读取文化信息。
默认提供程序包括：

* `QueryStringRequestCultureProvider`：搜索文化和 `ui-culture` 查询字符串参数。
* `CookieRequestCultureProvider`：使用 ASP.NET Core `cookie`。
* `AcceptLanguageHeaderRequestProvider`：从 `Accept-Language` HTTP 头读取请求的文化。

对于每个请求，系统会按此顺序尝试使用这些提供程序，直到找到第一个能确定文化的提供程序。

若无法设置文化，则使用 `RequestLocalizationOptions` 的 `DefaultRequestCulture` 属性中指定的文化。

如有必要，也可更改请求文化提供程序的顺序，甚至定义自定义提供程序来实现自己的确定文化的逻辑。
更多信息可在 [https://docs.microsoft.com/aspnet/core/fundamentals/localization#use-a-custom-provider](https://docs.microsoft.com/aspnet/core/fundamentals/localization#use-a-custom-provider) 查看。

> 重要提示：
> 
> 本地化中间件必须在可能使用请求文化的任何其他中间件之前插入。

在 Web API（无论是基于控制器还是 Minimal API）中，通常通过 `Accept-Language` HTTP 头设置请求文化。
接下来将看到如何扩展 Swagger，使其在调用方法时能添加此头。

### 为 Minimal API 添加全球化支持到 Swagger

我们希望 Swagger 能为每个请求指定 `Accept-Language` HTTP 头，以便测试全球化端点。
从技术上讲，这意味着向 Swagger 添加一个操作过滤器，使其能自动插入语言头，代码如下：

```csharp
public class AcceptLanguageHeaderOperationFilter : IOperationFilter
{
    private readonly List<IOpenApiAny>? supportedLanguages;

    public AcceptLanguageHeaderOperationFilter(
        IOptions<RequestLocalizationOptions> requestLocalizationOptions)
    {
        supportedLanguages = requestLocalizationOptions.Value.SupportedCultures?.Select(c =>
            new OpenApiString(c.TwoLetterISOLanguageName))
            .Cast<IOpenApiAny>().ToList();
    }

    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (supportedLanguages?.Any()?? false)
        {
        operation.Parameters??= new List<OpenApiParameter>();
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = HeaderNames.AcceptLanguage,
            In = ParameterLocation.Header,
            Required = false,
            Schema = new OpenApiSchema
            {
                Type = "string",
                Enum = supportedLanguages,
                Default = supportedLanguages.First()
            }
        });
        }
    }
}
```

在上述代码中，`AcceptLanguageHeaderOperationFilter` 通过依赖注入获取启动时定义的 `RequestLocalizationOptions` 对象，
并从中提取 Swagger 期望的支持语言格式。然后在 `Apply` 方法中，添加对应于 `Accept-Language` 头的新 `OpenApiParameter`。
通过 `Schema.Enum` 属性，使用在构造函数中提取的值提供支持语言列表。
此方法会为每个操作（即每个端点）调用，意味着参数会自动添加到每个端点。

现在将新过滤器添加到 Swagger：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddSwaggerGen(options =>
{
    options.OperationFilter<AcceptLanguageHeaderOperationFilter>();
});
```

像上述代码一样，对于每个操作，Swagger 会执行过滤器，过滤器会添加参数指定请求的语言。

假设存在以下端点：

```csharp
app.MapGet("/culture", () => 
    Thread.CurrentThread.CurrentCulture.DisplayName);
```

在上述处理程序中，只是返回线程的文化。此方法无参数，但添加上述过滤器后，Swagger UI 显示如下：

![Figure 9.1 – The Accept-Language header added to Swagger](/assets/images/minimal-apis/Figure_9.1_B17902.jpg)

可点击 “**Try it out**” 按钮从列表中选择值，再点击 “**Execute**” 调用端点：

![Figure 9.2 – The result of the execution with the Accept-Language HTTP header](/assets/images/minimal-apis/Figure_9.2_B17902.jpg)

这是选择意大利语作为语言请求的结果：Swagger 添加了 `Accept-Language` HTTP 头，
ASP.NET Core 用其设置当前文化，最终在路由处理程序中获取并返回文化显示名称。

此示例表明已正确为 Minimal API 添加全球化支持。
接下来进一步研究本地化，首先根据相应语言为调用者提供翻译资源。


## 使用资源文件

Minimal API 现在支持全球化，能根据请求切换文化，这意味着可向调用者提供本地化消息，如在传达验证错误时。
此功能基于资源文件（`.resx`），这是一种特殊的 XML 文件，包含表示必须本地化的消息的键值字符串对。

> 注意：
> 
> 这些资源文件与早期版本的.NET 中的文件相同。

### 创建和使用资源文件

使用资源文件可轻松将字符串与代码分离并按文化分组。
通常，资源文件放在名为 `Resources` 的文件夹中。
使用 Visual Studio 创建此类文件的步骤如下：

> 重要提示：
> 
> 遗憾的是，Visual Studio Code 不支持处理 `.resx` 文件。
> 更多信息可在 [https://github.com/dotnet/AspNetCore.Docs/issues/2501](https://github.com/dotnet/AspNetCore.Docs/issues/2501) 查看。

1、 在解决方案资源管理器中右键单击文件夹，选择 “**添加**”|“**新建项**”。

2、 在 “**添加新项**” 对话框窗口中搜索 “`Resources`”，选择相应模板并命名文件，如 `Messages.resx`：

![Figure 9.3 – Adding a resource file to the project](/assets/images/minimal-apis/Figure_9.3_B17902.jpg)

新文件会立即在 Visual Studio 编辑器中打开。

3、 在新文件中，首先从 “**访问修饰符**” 选项中选择 “**Internal**” 或 “**Public**”（根据所需代码可见性），
以便 Visual Studio 创建 C# 文件公开访问资源的属性：

![Figure 9.4 – Changing the Access Modifier of the resource file](/assets/images/minimal-apis/Figure_9.4_B17902.jpg)

更改此值后，Visual Studio 会向项目添加 `Messages.Designer.cs` 文件，并自动创建与资源文件中插入的字符串对应的属性。

资源文件必须遵循精确命名约定。包含默认文化消息的文件可任意命名（如示例中的 `Messages.resx`），
但提供相应翻译的其他 `.resx` 文件必须同名并指定所引用的文化（中性或特定）。所以有 `Messages.resx` 存储默认（英语）消息。

4、 由于要将消息本地化为意大利语，需创建另一个名为 Messages.it.resx 的文件。

> 注意：
> 
> 故意不创建法语文化的资源文件，以便了解 ASP.NET Core 如何查找本地化消息。

5、 现在可开始试验资源文件。打开 `Messages.resx` 文件，将 “`Name`” 设为 “**HelloWorld**”，“`Value`” 设为 “**Hello World!**”。

这样，Visual Studio 会在自动生成的 `Messages` 类中添加静态 `HelloWorld` 属性，以便根据当前文化访问值。

6、 为演示此行为，打开 `Messages.it.resx` 文件，添加同名（“`HelloWorld`”）项，将 “`Value`” 设为 “**Ciao mondo!**”。

7、 最后添加新端点展示资源文件用法：
```csharp
// using Chapter09.Resources;
app.MapGet("/helloworld", () => Messages.HelloWorld);
```

在上述路由处理程序中，简单访问静态 `Messages.HelloWorld` 属性，如前所述，该属性在编辑 `Messages.resx` 文件时自动创建。

运行 Minimal API 并执行此端点，根据在 Swagger 中选择的请求语言会得到以下响应：

| Accept-Language | Response     | Read from        |
|-----------------|--------------|------------------|
| en              | Hello World! | Messages.resx    |
| it              | Ciao mondo!  | Messages.it.resx |
| fr              | Hello World! | Messages.resx    |

当访问像 `HelloWorld` 这样的属性时，自动生成的 `Messages` 类内部使用 `ResourceManager` 查找相应本地化字符串。
首先查找包含请求文化的资源文件，若找不到，则查找父文化。
这意味着若请求文化是特定的，`ResourceManager` 会搜索中性文化；若仍找不到资源文件，则使用默认文件。

在示例中，使用 Swagger 只能选择英语、意大利语或法语作为中性文化。但如果客户端发送其他值会怎样呢？例如：

* 请求文化是 `it-IT`：系统搜索 `Messages.it-IT.resx`，然后找到并使用 `Messages.it.resx`。
* 请求文化是 `fr-FR`：系统搜索 `Messages.fr-FR.resx`，然后搜索 `Messages.fr.resx`，因两者均不可用，最终使用默认的 `Messages.resx`。
* 请求文化是 `de（德语）`：因不支持此文化，默认请求文化会被自动选中，字符串在 `Messages.resx` 文件中搜索。

> 注意：
> 
> 如果本地化资源文件存在但不包含指定键，则使用默认文件的值。

### 使用资源文件格式化本地化消息

还可使用资源文件格式化本地化消息。例如，向项目资源文件添加以下字符串：

| Name            | Value in Messages.resx | Value in Messages.it.resx |
|-----------------|------------------------|---------------------------|
| GreetingMessage | Hello, {0}!            | Ciao, {0}!                |

定义以下端点：

```csharp
// using Chapter09.Resources;
app.MapGet("/hello", (string name) =>
{
    var message = string.Format(Messages.GreetingMessage, name);
    return message;
});
```

在上述代码示例中，根据请求文化从资源文件获取字符串，且消息包含占位符，可使用传递给路由处理程序的名称创建自定义本地化消息。
执行端点会得到如下结果：

| Name  | Accept-Language | Response     | Read from        |
|-------|-----------------|--------------|------------------|
| Marco | en              | Hello Marco! | Messages.resx    |
| Marco | it              | Ciao Marco!  | Messages.it.resx |
| Marco | fr              | Hello Marco! | Messages.resx    |

能够创建带有在运行时用不同值替换的占位符的本地化消息是创建真正可本地化服务的关键。

开头提到在 Web API 中本地化的典型用例是在验证时提供本地化错误消息。接下来将看到如何向 Minimal API 添加此功能。

## 在验证框架中集成本地化

在 [第 6 章 “探索验证和映射”](/books/minimal-apis/chapter-06) 中，讨论了如何将验证集成到 Minimal API 项目中，
学习了使用 `MiniValidation` 库而非 `FluentValidation` 验证模型并向调用者提供验证消息，
还提到 `FluentValidation` 已默认提供标准错误消息的翻译。

然而，使用这两个库都可利用刚添加到项目中的本地化支持，以支持本地化和自定义验证消息。

### 使用 MiniValidation 本地化验证消息

使用 `MiniValidation` 库可在 Minimal API 中基于数据注释进行验证。参考**第 6 章**了解如何将此库添加到项目中。

重新创建 `Person` 类：

```csharp
public class Person
{
     [Required]
     [MaxLength(30)]
     public string FirstName { get; set; }
     [Required]
     [MaxLength(30)]
     public string LastName { get; set; }
     [EmailAddress]
     [StringLength(100, MinimumLength = 6)]
     public string Email { get; set; }
}
```

每个验证属性都可指定错误消息，可为静态字符串或对资源文件的引用。
看看如何正确处理 `Required` 属性的本地化。在资源文件中添加以下值：

| Name                    | Value in Messages.resx                  | Value in Messages.it.resx                 |
|-------------------------|-----------------------------------------|-------------------------------------------|
| FieldRequiredAnnotation | The field '{0}' is required             | Il campo '{0}' è obbligatorio             |
| FirstName               | First name                              | Nome                                      |
| LastName                | Last name                               | Cognome                                   |
| ValidationErrors        | One or more validation errors occurred. | Si sono verificati errori di validazione. |

希望当 `Required` 验证规则失败时，返回对应 `FieldRequiredAnnotation` 的本地化消息，且消息包含占位符，还需翻译属性名称。

有了这些资源，可更新 `Person` 类声明如下：

```csharp
public class Person
{
    [Display(Name = "FirstName", ResourceType = typeof(Messages))]
    [Required(ErrorMessageResourceName = "FieldRequiredAnnotation", 
        ErrorMessageResourceType = typeof(Messages))]
    public string FirstName { get; set; }
    //...
}
```

每个验证属性（如示例中的 `Required`）都公开属性，可指定使用的资源名称和包含相应定义的类类型。
注意名称是简单字符串，编译时不检查，若写错，只会在运行时出错。

接下来使用 `Display` 属性指定插入验证消息的字段名称。

> 注意：
> 
> 可在 GitHub 存储库（[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/blob/main/Chapter09/Program.cs#L97](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/blob/main/Chapter09/Program.cs#L97)）
> 查看带有本地化数据注释的 `Person` 类的完整声明。

现在重新添加第 6 章中的验证代码。不同之处在于现在验证消息将本地化：

```csharp
app.MapPost("/people", (Person person) =>
{
    var isValid = MiniValidator.TryValidate(person, out var errors);
    if (!isValid)
    {
        return Results.ValidationProblem(errors, title: Messages.ValidationErrors);
    }
    return Results.NoContent();
});
```

在上述代码中，`MiniValidator.TryValidate()` 方法返回的 `errors` 字典中的消息将根据请求文化进行本地化，
如前面章节所述。我们还在 `Results.ValidationProblem()` 调用中指定了标题参数，
因为我们也希望本地化这个值（否则，它将总是默认的 “One or more validation errors occurred”）。

如果我们更喜欢使用 `FluentValidation` 而不是数据注释，
我们知道从**第 6 章 “探索验证和映射”** 中可知它默认支持标准错误消息的本地化。
然而，使用这个库，我们也可以提供自己的翻译。在下一节中，我们将讨论实现此解决方案的方法。

### 使用 FluentValidation 本地化验证消息

使用 `FluentValidation`，我们可以完全将验证规则与模型分离。
如前所述，请参考**第 6 章**，了解如何将此库添加到项目中以及如何配置它。

接下来，重新创建 `PersonValidator` 类：

```csharp
public class PersonValidator : AbstractValidator<Person>
{
    public PersonValidator()
    {
        RuleFor(p => p.FirstName).NotEmpty().MaximumLength(30);
        RuleFor(p => p.LastName).NotEmpty().MaximumLength(30);
        RuleFor(p => p.Email).EmailAddress().Length(6, 100);
    }
}
```

如果我们没有指定任何消息，将使用默认消息。
让我们添加以下资源来自定义 `NotEmpty` 验证规则的消息：

| Name            | Value in Messages.resx                 | Value in Messages.it.resx                |
|-----------------|----------------------------------------|------------------------------------------|
| NotEmptyMessage | The field '{PropertyName}' is required | Il campo '{PropertyName}' è obbligatorio |

注意，在这种情况下，我们也有一个占位符，它将被属性名称替换。
然而，与数据注释不同，`FluentValidation` 使用具有名称的占位符来更好地识别其含义。

现在，我们可以在验证器中为例如 `FirstName` 属性添加此消息：

```csharp
RuleFor(p => p.FirstName).NotEmpty()
    .WithMessage(Messages.NotEmptyMessage)
    .WithName(Messages.FirstName);
```

我们使用 `WithMessage()` 来指定在前面规则失败时必须使用的消息，
然后添加 `WithName()` 调用来覆盖用于消息中 `{PropertyName}` 占位符的默认属性名称。

> 注意：
> 
> 你可以在 GitHub 存储库（[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/blob/main/Chapter09/Program.cs#L129](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/blob/main/Chapter09/Program.cs#L129)）
> 中找到带有本地化消息的 `PersonValidator` 类的完整实现。

最后，我们可以像在 **第 6 章** 中那样在端点中利用本地化的验证器：

```csharp
app.MapPost("/people", async (Person person, IValidator<Person> validator) =>
{
    var validationResult = await validator.ValidateAsync(person);
    if (!validationResult.IsValid)
    {
        var errors = validationResult.ToDictionary();
        return Results.ValidationProblem(errors, title: Messages.ValidationErrors);
    }
    return Results.NoContent();
});
```

与数据注释的情况一样，`validationResult` 变量将包含本地化的错误消息，
我们使用 `Results.ValidationProblem()` 方法将其返回给调用者（再次，带有标题属性的定义）。

> 提示：
> 
> 在我们的示例中，我们已经看到了如何使用 `WithMessage()` 方法为每个属性显式分配翻译。
> `FluentValidation` 还提供了一种替换其所有（或部分）默认消息的方法。
> 你可以在官方文档（[https://docs.fluentvalidation.net/en/latest/localization.html#default-messages](https://docs.fluentvalidation.net/en/latest/localization.html#default-messages)）中找到更多信息。

这就结束了我们使用资源文件进行本地化的概述。接下来，我们将讨论在处理旨在全球使用的服务时的一个重要主题：正确处理不同的时区。

## 为全球化的 Minimal API 添加 UTC 支持

到目前为止，我们已经为 Minimal API 添加了全球化和本地化支持，因为我们希望它能够被尽可能广泛的受众使用，而不受文化的限制。
但是，如果我们考虑到要面向全球受众，我们应该考虑与全球化相关的几个方面。全球化不仅涉及语言支持；
还有一些重要因素需要考虑，例如地理位置以及时区。

例如，我们的 Minimal API 可能在意大利运行，遵循 **中欧时间**（CET）（GMT+1），
而我们的客户端可能在世界各地使用浏览器执行单页应用程序，而不是移动应用程序。
我们可能还有一个包含我们数据的数据库服务器，它可能位于另一个时区。
此外，在某个时候，可能需要为全球用户提供更好的支持，因此我们可能需要将我们的服务迁移到另一个位置，这可能具有新的时区。
总之，我们的系统可能需要处理不同时区的数据，并且同一服务在其生命周期内可能会切换时区。

在这些情况下，理想的解决方案是使用 `DateTimeOffset` 数据类型，它包含时区信息，
并且 `JsonSerializer` 完全支持它，在序列化和反序列化过程中会保留时区信息。
如果我们总是能够使用它，就可以自动解决与全球化相关的任何问题，因为将 `DateTimeOffset` 值转换为不同时区非常简单。
然而，在某些情况下，我们无法处理 `DateTimeOffset` 类型，例如：

* 当我们在一个依赖于 `DateTime` 到处使用的遗留系统上工作时，
更新代码以使用 `DateTimeOffset` 不是一个选项，因为这需要太多的更改并且会破坏与旧数据的兼容性。
* 我们有一个数据库服务器，如 MySQL，它没有直接存储 `DateTimeOffset` 的列类型，所以处理它需要额外的努力，
例如使用两个单独的列，这会增加领域的复杂性。
* 在某些情况下，我们根本不关心发送、接收和保存时区信息 —— 我们只是想以一种 “_通用_” 的方式处理时间。

因此，在所有无法或不想使用 `DateTimeOffset` 数据类型的场景中，
处理不同时区的最佳和最简单方法之一是使用`协调世界时（UTC）`来处理所有日期：
服务必须假设它接收到的日期是 UTC 格式，并且另一方面，API 返回的所有日期也必须是 UTC 格式。

当然，我们必须以集中的方式处理这种行为；我们不想每次接收或发送日期时都记住要应用到 UTC 格式的转换。
著名的 _JSON.NET_ 库提供了一个选项来指定在处理 `DateTime` 属性时如何对待时间值，
允许它自动将所有日期视为 UTC 并在它们表示本地时间时将其转换为该格式。
然而，当前版本的 Microsoft `JsonSerializer` 在 Minimal API 中不包括这样的功能。
从第 2 章 “探索 Minimal API 及其优势” 中我们知道，我们不能更改 Minimal API 中的默认 JSON 序列化器，
但我们可以通过创建一个简单的 `JsonConverter` 来克服这种缺乏 UTC 支持的问题：

```csharp
public class UtcDateTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader reader, 
        Type typeToConvert, 
        JsonSerializerOptions options)
        => reader.GetDateTime().ToUniversalTime();

    public override void Write(Utf8JsonWriter writer, 
        DateTime value, 
        JsonSerializerOptions options)
        => writer.WriteStringValue(
            (value.Kind == DateTimeKind.Local? value.ToUniversalTime() : value)
                .ToString("yyyy'-'MM'-'dd'T'HH':'mm':'ss'.'fffffff'Z'"));
}
```

通过这个转换器，我们告诉 `JsonSerializer` 如何处理 `DateTime` 属性：

* 当从 JSON 读取 `DateTime` 时，使用 `ToUniversalTime()` 方法将值转换为 UTC。
* 当必须将 `DateTime` 写入 JSON 时，如果它表示本地时间（`DateTimeKind.Local`），
在序列化之前将其转换为 UTC —— 然后使用 `Z` 后缀进行序列化，该后缀表示时间是 UTC。

现在，在使用这个转换器之前，让我们添加以下端点定义：

```csharp
app.MapPost("/date", (DateInput date) =>
{
    return Results.Ok(new
    {
        Input = date.Value,
        DateKind = date.Value.Kind.ToString(),
        ServerDate = DateTime.Now
    });
});

public record DateInput(DateTime Value);
```

让我们尝试用例如格式为 `2022-03-06T16:42:37-05:00` 的日期调用它。我们将得到类似以下的结果：

```json
{
    "input": "2022-03-06T22:42:37+01:00",
    "dateKind": "Local",
    "serverDate": "2022-03-07T18:33:17.0288535+01:00"
}
```

输入日期，包含时区信息，已自动转换为服务器的本地时间（在这种情况下，服务器在意大利运行，如开头所述），
这也由 `dateKind` 字段证明。此外，`serverDate` 包含相对于服务器时区的日期。

现在，让我们将 `UtcDateTimeConverter` 添加到 `JsonSerializer`：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.Configure<Microsoft.AspNetCore.Http.JsonOptions>(options =>
{
    options.SerializerOptions.Converters.Add(new UtcDateTimeConverter());
});
```

通过这几行代码，我们将自定义的 `UtcDateTimeConverter` 添加到了 `JsonSerializerOptions` 的转换器列表中。

现在，如果我们再次使用相同的输入日期（例如格式为 `2022-03-06T16:42:37-05:00`）调用 `/date` 端点，会得到如下结果：

```json
{
    "input": "2022-03-06T21:42:37Z",
    "dateKind": "Utc",
    "serverDate": "2022-03-07T17:33:17Z"
}
```

可以看到，输入的日期现在被正确地转换为了协调世界时（UTC）格式，如 `input` 字段所示，其末尾带有 `Z` 后缀来表明是 UTC 时间。
同时，服务器端生成的 `serverDate` 也同样被转换为了 UTC 格式。

这样，我们就以一种简单的方式确保了 API 无论是接收还是返回日期时，都能以 UTC 格式来处理，
从而更好地应对来自不同时区客户端的请求，增强了整个应用程序在全球化场景下的适应性。

在实际应用中，可能还需要考虑其他一些相关因素，比如在数据库中存储日期时也统一采用 UTC 格式，
或者在客户端展示日期时，根据用户所在时区将 UTC 时间转换为对应的本地时间等。
但通过在 Minimal API 中添加这个 UTC 支持的基础机制，为后续进一步处理这些复杂情况奠定了良好的基础。

## 总结

在当今这个相互联系日益紧密的世界里，开发具备全球化和本地化支持功能的 Minimal API 是至关重要的。
ASP.NET Core 提供了创建能够响应用户所在文化、并依据请求语言提供相应翻译服务所需的全部功能，
其中包括本地化中间件、资源文件以及自定义验证消息等的使用，借助这些功能，我们能够打造出几乎可以支持所有文化的服务。

我们还探讨了在处理不同时区时可能出现的与全球化相关的问题，并展示了如何通过使用统一的协调世界时（UTC）日期时间格式来解决这些问题，
如此一来，无论客户端处于何种地理位置、属于哪个时区，我们的 API 都能够无缝运行。

在 [第 10 章 “评估和衡量 Minimal API 的性能”](/books/minimal-apis/chapter-10) 中，我们将探讨创建 Minimal API 的原因，
并分析与经典的基于控制器的方法相比，使用 Minimal API 在性能方面的优势。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/minimal-apis)
