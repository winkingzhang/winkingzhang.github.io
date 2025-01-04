---
title: "掌握 Minimal APIs 技术"
excerpt: "使用 .NET 和 C# 编译、测试和快速开发 web api 原型应用"
sitemap: false
permalink: /books/minimal-apis/chapter-06
layout: single
classes: wide
sidebar:
  nav: "minimal_apis"
---


## 探索验证和映射

在本章中，我们将讨论如何在 Minimal API 中执行数据验证和映射，展示当前可用的功能、缺失的部分以及最有趣的替代方案。
了解这些概念将帮助我们开发更健壮和可维护的应用程序。

本章将涵盖以下主题：
* 处理验证
* 数据与 API 之间的映射

### 技术要求

要遵循本章中的描述，你需要创建一个 ASP.NET Core 6.0 Web API 应用程序。
请参考 [第 2 章 “探索最小 API 及其优势”](/books/minimal-apis/chapter-02) 中的技术要求部分，了解如何创建。

如果你使用控制台、shell 或 Bash 终端创建 API，请记住将工作目录更改为当前章节编号（Chapter06）。

本章中的所有代码示例都可以在本书的 GitHub 存储库中找到，地址为 <br/>
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter06](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter06)

### 处理验证

**数据验证**是任何有效软件中最重要的过程之一。
在 Web API 的背景下，我们执行验证过程以确保传递给我们端点的信息符合某些规则，
例如，一个 `Person` 对象必须定义 `FirstName` 和 `LastName` 属性，电子邮件地址有效，或者约会日期不在过去。

在基于控制器的项目中，我们可以直接在模型上执行这些检查（也称为模型验证），使用数据注解。
实际上，放置在控制器上的 `ApiController` 属性使模型验证错误在一个或多个验证规则失败时
自动触发 `400 Bad Request` 响应。
因此，在基于控制器的项目中，通常根本不需要显式执行模型验证：如果验证失败，我们的端点将永远不会被调用。

> **注意**：
> 
> `ApiController` 属性使用 `ModelStateInvalidFilter` 操作过滤器启用自动模型验证行为。

不幸的是，Minimal API 不提供内置的验证支持。
`IModelValidator` 接口和所有相关对象都不能使用。 因此，我们没有 `ModelState`；
如果存在验证错误，我们无法阻止端点的执行，并且必须显式返回 `400 Bad Request` 响应。

例如，看以下代码：

```csharp
app.MapPost("/people", (Person person) =>
{
    return Results.NoContent();
});

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

如我们所见，即使 `Person` 参数不遵守验证规则，端点也会被调用。
只有一个例外：如果我们使用 **可空引用类型** 并且在请求中没有传递主体，我们实际上会得到 `400 Bad Request` 响应。
如第 2 章 “探索 Minimal API 优势” 中所述，在 .NET 6.0 项目中默认启用可空引用类型。

如果我们想接受一个 `null` 请求主体（如果有需要），我们需要将参数声明为 `Person?`。
但是，只要有主体，端点就会始终被调用。

因此，在 Minimal API 中，有必要在每个路由处理程序中执行验证，并在某些规则失败时返回适当的响应。
我们可以实现一个与现有属性兼容的验证库，以便我们可以使用经典的数据注释方法执行验证，
如下一节所述，或者使用第三方解决方案，如 `FluentValidation`，我们将在 “集成 `FluentValidation`” 部分看到。

### 使用数据注释执行验证

如果我们想使用基于数据注释的常见验证模式，我们需要依赖 **反射** 来检索模型中的所有验证属性并调用它们的 `IsValid` 方法，
这些方法由 `ValidationAttribute` 基类提供。

这种行为是对 ASP.NET Core 实际处理验证方式的简化。然而，这是基于控制器的项目中验证的工作方式。

虽然我们也可以在 Minimal API 中手动实现这种解决方案，但如果我们决定使用数据注释进行验证，
我们可以利用一个小而有趣的库，`MiniValidation`，它可在 GitHub（[https://github.com/DamianEdwards/MiniValidation](https://github.com/DamianEdwards/MiniValidation)）
和 NuGet（[https://www.nuget.org/packages/MiniValidation](https://www.nuget.org/packages/MiniValidation)）上获取。

>
> 重要提示：
> 
> 在撰写本文时，`MiniValidation` 在 NuGet 上作为预发布版本提供。
>

我们可以通过以下方式之一将此库添加到我们的项目中：

* 选项 1：如果你使用 Visual Studio 2022，右键单击项目并选择 “**管理 NuGet 包**” 命令以打开**包管理器界面**；
然后搜索 `MiniValidation`。确保选中 “**包括预发布**” 选项并点击 “**安装**”。
* 选项 2：如果你在 Visual Studio 2022 内部，打开包管理器控制台，
或者打开你的控制台、shell 或 Bash 终端，转到你的项目目录，并执行以下命令：<br/>
`dotnet add package MiniValidation --prerelease`

现在，我们可以使用以下代码验证一个 `Person` 对象：

```csharp
app.MapPost("/people", (Person person) =>
{
    var isValid = MiniValidator.TryValidate(person, out var errors);
    if (!isValid)
    {
        return Results.ValidationProblem(errors);
    }
    return Results.NoContent();
});
```

如我们所见，`MiniValidation` 提供的 `MiniValidator.TryValidate` 静态方法接受一个对象作为输入，
并自动验证在其属性上定义的所有验证规则。如果验证失败，它返回 `false` 并使用发生的所有验证错误填充 out 参数。
在这种情况下，因为我们负责返回适当的响应代码，我们使用 `Results.ValidationProblem`，
它产生一个 `400 Bad Request` 响应，带有一个 `ProblemDetails` 对象（如第 3 章 “使用最小 API” 中所述），
并且还包含验证遇到的问题。

现在，例如，我们可以使用以下无效输入调用端点：
```json
{
    "lastName": "MyLastName",
    "email": "email"
}
```
这是我们将获得的响应：
```json
{
    "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "errors": {
        "FirstName": [
            "The FirstName field is required."
        ],
        "Email": [
            "The Email field is not a valid e-mail address.",
            "The field Email must be a string with a minimum length of 6 and a maximum length of 100."
        ]
    }
}
```

通过这种方式，除了需要手动执行验证之外，我们可以在我们的模型上以与我们在以前版本的 ASP.NET Core 中习惯的相同方式实现使用数据注释的方法。
我们还可以自定义错误消息并通过创建继承自 `ValidationAttribute` 的类来定义自定义规则。

>
> 注意：
> 
> ASP.NET Core 6.0 中可用的验证属性的完整列表发布在： <br/>
> [https://docs.microsoft.com/dotnet/api/system.componentmodel.dataannotations](https://docs.microsoft.com/dotnet/api/system.componentmodel.dataannotations)
> 如果你有兴趣创建自定义属性，你可以参考：
> [https://docs.microsoft.com/aspnet/core/mvc/models/validation#custom-attributes](https://docs.microsoft.com/aspnet/core/mvc/models/validation#custom-attributes)

虽然数据注释是最常用的解决方案，但我们也可以使用所谓的流畅方法处理验证，
这种方法的好处是将验证规则与模型完全分离，如下一节所述。

### 集成 FluentValidation

在每个应用程序中，正确组织我们的代码都很重要。
对于验证也是如此。虽然数据注释是一种可行的解决方案，但我们应该考虑替代方案，以帮助我们编写更易于维护的项目。
这就是 `FluentValidation` 的目的，一个属于 .NET 基金会的库，
它允许我们使用带有 **lambda** 表达式的流畅接口构建验证规则。
该库可在 GitHub（[https://fluentvalidation.net](https://fluentvalidation.net)）
和 NuGet（[https://www.nuget.org/packages/FluentValidation](https://www.nuget.org/packages/FluentValidation)）上获取。
这个库可以用于任何类型的项目，但在与 ASP.NET Core 一起使用时，
有一个专门的 NuGet 包（[https://www.nuget.org/packages/FluentValidation.AspNetCore](https://www.nuget.org/packages/FluentValidation.AspNetCore)），
其中包含有助于集成它的有用方法。

> 注意：
>
> .NET 基金会是一个独立的组织，旨在支持围绕.NET 平台的开源软件开发和协作。
> 你可以在 [https://dotnetfoundation.org](https://dotnetfoundation.org) 上了解更多信息。

如前所述，使用这个库，我们可以将验证规则与模型分离，以创建一个更结构化的应用程序。
此外，`FluentValidation` 允许我们使用流畅的语法定义更复杂的规则，
而无需创建基于 `ValidationAttribute` 的自定义类。该库还原生支持标准错误消息的本地化。

那么，让我们看看如何将 FluentValidation 集成到 Minimal API 项目中。
首先，我们需要通过以下方式之一将此库添加到我们的项目中：

* 选项 1：如果你使用 Visual Studio 2022，右键单击项目并选择 “**管理 NuGet 包**” 
命令以打开**包管理器界面**。然后搜索 `FluentValidation.DependencyInjectionExtensions` 并点击 “**安装**”。
* 选项 2：打开包管理器控制台，如果你在 Visual Studio 2022 内部，
或者打开你的控制台、shell 或 Bash 终端，转到你的项目目录，并执行以下命令 <br/>
`dotnet add package FluentValidation.DependencyInjectionExtensions`

现在，我们可以重写 `Person` 对象的验证规则，并将它们放在一个 `PersonValidator` 类中：

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

`PersonValidator` 继承自 `AbstractValidator<T>`，这是 `FluentValidation` 提供的一个基类，
包含我们定义验证规则所需的所有方法。例如，我们流畅地说我们有一个针对 `FirstName` 属性的规则，
即它不能为空且最大长度为 30 个字符。

下一步是在服务提供程序中注册验证器，以便我们可以在路由处理程序中使用它。
我们可以通过一个简单的指令执行此任务：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddValidatorsFromAssemblyContaining<PersonValidator>();
```

`AddValidatorsFromAssemblyContaining` 方法自动注册在包含指定类型的程序集中派生自 `AbstractValidator` 的所有验证器。
特别是，此方法注册验证器并使其可通过依赖注入通过 `IValidator<T>` 接口访问，
而 `IValidator<T>` 接口又由 `AbstractValidator<T>` 类实现。如果我们有多个验证器，
我们可以使用此单个指令注册它们。我们也可以轻松地将我们的验证器放在外部程序集中。

现在一切都已就绪，记住在 Minimal API 中我们没有自动模型验证，我们必须以以下方式更新我们的路由处理程序：

```csharp
app.MapPost("/people", async (Person person, IValidator<Person> validator) =>
{
    var validationResult = await validator.ValidateAsync(person);
    if (!validationResult.IsValid)
    {
        var errors = validationResult.ToDictionary();
        return Results.ValidationProblem(errors);
    }
    return Results.NoContent();
});
```

我们在路由处理程序参数列表中添加了一个 `IValidator<Person>` 参数，
所以现在我们可以调用它的 `ValidateAsync` 方法来针对输入的 `Person` 对象应用验证规则。
如果验证失败，我们提取所有错误消息并使用通常的 `Results.ValidationProblem` 方法将它们返回给客户端，
如前一节所述。

总之，让我们看看如果我们尝试使用之前的输入调用端点会发生什么：

```json
{
    "lastName": "MyLastName",
    "email": "email"
}
```

我们将得到以下响应：
```json
{
    "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "errors": {
        "FirstName": [
            "'First Name' non può essere vuoto."
        ],
        "Email": [
            "'Email' non è un indirizzo email valido.",
            "'Email' deve essere lungo tra i 6 e 100 caratteri. Hai inserito 5 caratteri."
        ]
    }
}
```

如前所述，`FluentValidation` 提供标准错误消息的翻译，所以这是在意大利系统上运行时得到的响应。
当然，我们可以使用典型的流畅方法完全自定义消息，通过将 `WithMessage` 方法链接到验证器中定义的验证方法。
例如，看以下代码：

```csharp
RuleFor(p => p.FirstName)
    .NotEmpty()
    .WithMessage("You must enter a first name.");
```

我们将在 [第 9 章 “利用全球化和本地化”](/books/minimal-apis/chapter-09) 中更详细地讨论本地化。

这只是一个关于如何使用 `FluentValidation` 定义验证规则并在 Minimal API 中使用它们的快速示例。
这个库允许许多更复杂的场景，在官方文档（[https://fluentvalidation.net](https://fluentvalidation.net)）中有全面的描述。

现在我们已经看到了如何向我们的路由处理程序添加验证，了解如何使用此信息更新 Swagger 创建的文档也很重要。


### 向 Swagger 添加验证信息

无论选择哪种解决方案来处理验证，重要的是使用 `ProducesValidationProblem` 方法在端点声明后
更新 OpenAPI 定义，以指示处理程序可以产生验证问题响应：

```csharp
app.MapPost("/people", (Person person) =>
{
    //...
})
.Produces(StatusCodes.Status204NoContent)
.ProducesValidationProblem();
```

通过这种方式，将为 Swagger 添加一个新的 `400 Bad Request` 状态码的响应类型，如下所示：

![Figure 6.1 – The validation problem response added to Swagger](/assets/images/minimal-apis/Figure_6.1_B17902.jpg)

此外，Swagger UI 底部显示的 JSON 模式可以显示相应模型的规则。
使用数据注释定义验证规则的一个好处是它们会自动反映在这些模式中：

![Figure 6.2 – The validation rules for the Person object in Swagger](/assets/images/minimal-apis/Figure_6.2_B17902.jpg)

不幸的是，使用 `FluentValidation` 定义的验证规则不会自动显示在 Swagger 的 JSON 模式中。
我们可以通过使用 `MicroElements.Swashbuckle.FluentValidation` 克服此限制，这是一个小库，
通常可在 GitHub（[https://github.com/microelements/MicroElements.Swashbuckle.FluentValidation](https://github.com/microelements/MicroElements.Swashbuckle.FluentValidation)）
和 NuGet（[https://www.nuget.org/packages/MicroElements.Swashbuckle.FluentValidation](https://www.nuget.org/packages/MicroElements.Swashbuckle.FluentValidation)）上获取。
在按照之前介绍其他 NuGet 包的相同步骤将其添加到我们的项目后，
我们只需调用 `AddFluentValidationRulesToSwagger` 扩展方法：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddFluentValidationRulesToSwagger();
```

通过这种方式，Swagger 中显示的 JSON 模式将反映验证规则，就像数据注释一样。
然而，值得记住的是，在撰写本文时，这个库不支持 `FluentValidation` 中所有可用的验证器。
如需更多信息，我们可以参考该库的 GitHub 页面。

这结束了我们对 Minimal API 中验证的概述。
在下一节中，我们将分析每个 API 的另一个重要主题：如何正确处理数据与我们的服务之间的映射。


### 数据与 API 之间的映射

当处理可以被任何系统调用的 API 时，有一个黄金法则：_我们永远不应将内部对象暴露给调用者。_
如果我们不遵循这种解耦思想，并且由于某种原因需要更改我们的内部数据结构，我们可能会破坏所有与我们交互的客户端。
内部数据结构和用于与客户端对话的对象必须能够彼此独立发展。

这种对话要求就是**映射**如此重要的原因。
我们需要将一种类型的输入对象转换为不同类型的输出对象，反之亦然。
通过这种方式，我们可以实现两个目标：

* 在不引入对调用者公开的契约的破坏性更改的情况下发展我们的内部数据结构
* 更改用于与客户端通信的对象的格式，而无需更改内部处理这些对象的方式

换句话说，映射意味着通过从源复制和转换对象的属性到目标，将一个对象转换为另一个对象。
然而，映射代码很枯燥，测试映射代码更加枯燥。
尽管如此，我们需要充分理解这个过程至关重要，并努力在所有场景中采用它。

那么，考虑以下对象，它可能代表使用 **Entity Framework Core** 存储在数据库中的一个 `Person` 对象：

```csharp
public class PersonEntity
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
    public string City { get; set; }
}
```

我们已经设置了获取人员列表或检索特定人员的端点。

第一个想法可能是直接将 `PersonEntity` 返回给调用者。
以下高度简化的代码足以让我们理解这个场景：

```csharp
app.MapGet("/people/{id:int}", (int id) =>
{
    // 在实际应用程序中，这个实体可能会从数据库中检索，检查具有给定 ID 的人是否存在。
    var person = new PersonEntity();
    return Results.Ok(person);
})
.Produces(StatusCodes.Status200OK, typeof(PersonEntity))
```

如果我们需要修改数据库的模式，例如添加实体的创建日期，会发生什么情况？
在这种情况下，我们需要用一个新属性更改 `PersonEntity`，以映射相关日期。
然而，调用者现在也会得到这个信息，这可能是我们不想暴露的。
相反，如果我们使用所谓的 **数据转换对象（DTO）** 来公开人员，这个问题就会变得多余：

```csharp
public class PersonDto
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
    public string City { get; set; }
}
```

这意味着 API 应返回 `PersonDto` 类型的对象而非 `PersonEntity`，需在两者之间进行转换。
乍一看这似乎是代码重复，因为两个类属性相同。

但如果考虑到 `PersonEntity` 可能因数据库需求增加新属性或结构因新语义改变（调用者不应知晓），
映射的重要性就显而易见了。例如，将城市存储在单独表中并通过 `Address` 属性公开，
或者出于安全原因不想公开确切出生日期而只公开年龄。
使用专门的 DTO，可轻松更改模式和更新映射，而无需触及实体，更好地分离关注点。
当然，映射可以是双向的。在示例中，需将 `PersonEntity` 转换为 `PersonDto` 后返回给客户端，
也可将来自客户端的 `PersonDto` 转换为 `PersonEntity` 存入数据库。
我们讨论的所有解决方案都适用于这两种情况。

我们可以手动执行映射，也可采用提供此功能的第三方库。
接下来分析这两种方法的优缺点。

#### 手动执行映射

如前所述，映射本质上是将源对象属性复制到目标对象属性并进行转换。
手动执行是最简单有效的方法。

使用此方法需自行处理所有映射代码。
需要一个方法接受输入对象并转换为输出对象，若类包含复杂属性需递归映射。
建议使用扩展方法以便在需要时轻松调用。

此映射过程的完整示例可在 GitHub 存储库中找到：
[https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter06](https://github.com/PacktPublishing/Minimal-APIs-in-ASP.NET-Core6/tree/main/Chapter06)

这种解决方案能保证最佳性能，因为我们显式编写了所有映射指令，无需依赖自动系统（如反射）。
然而，手动方法存在一个缺点：每次在实体中添加必须映射到 DTO 的属性时，都需要更改映射代码。
另一方面，一些方法可以简化映射，但代价是会产生性能开销。
在下一节中，我们将研究一种使用 `AutoMapper` 的方法。

#### 使用 AutoMapper 进行映射

`AutoMapper` 可能是 .NET 中最著名的映射框架之一。
它使用流畅的配置 API，结合基于约定的匹配算法，将源值匹配到目标值。
与 `FluentValidation` 一样，该框架是 .NET 基金会的一部分，
可在 GitHub（[https://github.com/AutoMapper/AutoMapper](https://github.com/AutoMapper/AutoMapper)）
或 NuGet（[https://www.nuget.org/packages/AutoMapper](https://www.nuget.org/packages/AutoMapper)）上获取。
同样，在这种情况下，我们有一个特定的 NuGet 包（[https://www.nuget.org/packages/AutoMapper.Extensions.Microsoft.DependencyInjection](https://www.nuget.org/packages/AutoMapper.Extensions.Microsoft.DependencyInjection)），
它简化了在 ASP.NET Core 项目中的集成。

让我们快速了解一下如何将 `AutoMapper` 集成到 Minimal API 项目中，并展示其主要功能。
该库的完整文档可在 [https://docs.automapper.org](https://docs.automapper.org) 上找到。

像往常一样，首先要做的是按照前几节中使用的相同说明将库添加到我们的项目中。
然后，我们需要配置 `AutoMapper`，告诉它如何执行映射。
有几种方法可以执行此任务，但推荐的方法是创建继承自库提供的 `Profile` 基类的类，并将配置放入构造函数中：

```csharp
public class PersonProfile : Profile
{
    public PersonProfile()
    {
        CreateMap<PersonEntity, PersonDto>();
    }
}
```

这就是我们开始所需的全部：一个简单的指令，指示我们要将 `PersonEntity` 映射到 `PersonDto`，无需任何其他细节。
我们已经说过 `AutoMapper` 是基于约定的。
这意味着，默认情况下，它会将源和目标中具有相同名称的属性进行映射，同时在必要时自动转换为兼容类型。
例如，源上的一个 `int` 属性可以自动映射到目标上具有相同名称的 `double` 属性。
换句话说，如果源和目标对象具有相同的属性，则不需要任何显式的映射指令。
然而，在我们的情况下，我们需要执行一些转换，所以我们可以在 `CreateMap` 之后流畅地添加它们：

```csharp
public class PersonProfile : Profile
{
    public PersonProfile()
    {
        CreateMap<PersonEntity, PersonDto>()
         .ForMember(dst => dst.Age, opt =>
                opt.MapFrom(src => CalculateAge(src.BirthDate)))
         .ForMember(dst => dst.City, opt =>
                opt.MapFrom(src => src.Address.City));
    }

    private static int CalculateAge(DateTime dateOfBirth)
    {
        var today = DateTime.Today;
        var age = today.Year - dateOfBirth.Year;
        if (today.DayOfYear < dateOfBirth.DayOfYear)
        {
            age--;
        }
        return age;
    }
}
```

使用 `ForMember` 方法，我们可以指定如何映射目标属性 `dst.Age` 和 `dst.City`，使用转换表达式。
我们仍然不需要显式地映射 `Id`、`FirstName` 或 `LastName` 属性，因为它们在源和目标中都存在这些名称。

现在我们已经定义了映射配置文件，我们需要在启动时注册它，以便 ASP.NET Core 可以使用它。
与 `FluentValidation` 一样，我们可以在 `IServiceCollection` 上调用一个扩展方法：

```csharp
builder.Services.AddAutoMapper(typeof(Program).Assembly);
```

通过这行代码，我们会自动注册指定程序集中包含的所有配置文件。
如果我们向项目中添加更多配置文件，例如为每个要映射的实体创建一个单独的 `Profile` 类，我们不需要更改注册指令。

这样，我们现在就可以通过依赖注入使用 `IMapper` 接口了：

```csharp
app.MapGet("/people/{id:int}", (int id, IMapper mapper) =>
{
    var personEntity = new PersonEntity();
    //...
    var personDto = mapper.Map<PersonDto>(personEntity);
    return Results.Ok(personDto);
})
.Produces(StatusCodes.Status200OK, typeof(PersonDto))
```

在从数据库（例如使用 Entity Framework Core）检索到 `PersonEntity` 之后，
我们在 `IMapper` 接口上调用 `Map` 方法，指定结果对象的类型和输入类。
通过这行代码，`AutoMapper` 将使用相应的配置文件将 `PersonEntity` 转换为 `PersonDto` 实例。

有了这个解决方案，映射现在更容易维护，因为只要我们在源和目标上添加具有相同名称的属性，
我们就根本不需要更改配置文件。此外，`AutoMapper` 支持列表映射和递归映射。
所以，如果我们有一个必须映射的实体，例如 `PersonEntity` 类上的 `AddressEntity` 类型的属性，
并且相应的配置文件可用，转换将再次自动执行。

这种方法的缺点是性能开销。`AutoMapper` 通过在运行时动态执行映射代码来工作，所以它在底层使用反射。
配置文件在第一次使用时创建，然后缓存以加快后续映射。
然而，配置文件总是动态应用的，所以操作的成本取决于映射代码本身的复杂性。
我们只看到了 `AutoMapper` 的一个基本示例。
这个库非常强大，可以管理相当复杂的映射。
然而，我们需要小心不要滥用它，否则，我们可能会对应用程序的性能产生负面影响。


### 总结

验证和映射是在开发 API 时需要考虑的两个重要功能，以构建更健壮和可维护的应用程序。
Minimal API 不提供执行这些任务的内置方法，所以了解如何添加对此类功能的支持很重要。
我们已经看到我们可以使用数据注释或 `FluentValidation` 执行验证，
并了解如何向 Swagger 添加验证信息。
我们还讨论了数据映射的重要性，并展示了如何利用手动映射或 `AutoMapper` 库，描述了每种方法的优缺点。

在下一章中，我们将讨论如何将 Minimal API 与**数据访问层**集成，
例如展示如何使用 Entity Framework Core 访问数据库。



<br/><br/><br/><br/>
&gt;  [返回扉页](/books/minimal-apis)
