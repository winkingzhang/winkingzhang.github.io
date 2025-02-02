---
title: "Modern Angular"
excerpt: "全面且深入地探讨了现代 Angular 框架的核心特性、应用场景与实践策略，是一本助力开发者高效构建前沿 Web 应用的实用指南。"
sitemap: false
permalink: /books/modern-angular/chapter-02
layout: single
classes: wide
sidebar:
  nav: "modern_angular"
---

## 独立的未来

**本章涵盖内容：**

* 无需 `NgModule` 即可使用 `Angular` 组件、指令和管道
* 不借助 `NgModule` 构建应用程序结构
* 独立组件的路由与延迟加载
* 将现有应用迁移为独立应用

在上一章中，我们了解了 `Angular` 的最新发展情况，并制定了本书的学习计划。
我们还创建了一个项目，且已经接触到了独立组件。
现在是时候探索现代 `Angular` 的新能力之一了，即无需 `NgModule`（通俗地称为独立模式）来构建应用程序，并了解这种方法的优点和缺点。
为此，我们首先需要更深入地探究团队纷纷摒弃 `NgModule` 的原因。

### 2.1 为什么要摒弃 NgModule？

如我们所知，在 Angular v14 之前，所有的 Angular 应用程序都需要使用 NgModule 才能运行。这是新开发者学习 Angular 时接触到的第一个概念，它是将 Angular 应用程序粘合在一起的粘合剂；相比之下，组件并非这种粘合剂。在这种情况下，NgModule 是框架的一个基本组成部分。那么，发生了什么变化呢？为什么会有如此巨大的改变？事实证明，并非所有人都对 NgModule 感到满意。

在继续探讨之前，我们需要了解关于模块和独立模式的几个关键点：

* `NgModule` 仍然存在并受到支持，且尚未被弃用。实际上，是否弃用还有待讨论。
* 独立构建块与 `NgModule` 的互操作性良好；
你可以有一个基于模块的项目，其中包含一些独立组件，反之亦然，并且你可以在独立组件中使用 `NgModule`。
* 目标是使 `NgModule` 成为可选的，而不是完全摒弃它们（至少目前是这样）。
* 开发者似乎更喜欢独立模式的设置，并且作为开发者，我们在未来遇到更多独立项目的可能性越来越大。
* 核心团队本身似乎也更倾向于独立模式，这使得未来弃用 `NgModule` 的可能性更大。

现在，让我们来探讨开发者不喜欢 `NgModule` 的所有原因。

#### 2.1.1 难学且难解释

`NgModule` 无论是自己学习，还是教授或解释给他人，都颇具难度。
为理解这一观点，让我们设想两种情景：一种是我们初次接触 `Angular`，另一种是我们维护一个 `Angular` 项目并迎接新成员加入。

##### 学习 Angular

我们是 `Angular` 的学习者：刚刚学完 `JavaScript`，读了一些关于 `TypeScript` 的内容，现在准备在 `Angular` 的世界里大展身手！
这时迎来了第一课：一个名为 `NgModule` 的概念。它们是什么呢？

嗯，官方文档的解释是：“`NgModule` 配置注入器和编译器，并帮助将相关内容组织在一起”（[https://angular.io/guide/ngmodules](https://angular.io/guide/ngmodules)）。

这并没有太大帮助。什么是注入器？什么是编译器？
结果是，要理解 `NgModule`，我们首先需要学习一堆其他概念，对于初学者来说，这些概念相当复杂。
那么现在该怎么办？我们可以查看 `main.ts` 文件，它显然是我们应用程序的第一个入口点。大致内容如下所示。

**清单 2.1 `NgModule` 设置中的 `main.ts` 文件**

```typescript
import { enableProdMode } from '@angular/core';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import { environment } from './environments/environment';

if (environment.production) {
  enableProdMode();
}

platformBrowserDynamic().bootstrapModule(AppModule)
 .catch(err => console.error(err));
```

**#1**：使用 `NgModule` 时，我们需要启用生产模式以禁用一些检查和断言。<br/>
**#2**：我们只能引导一个 `NgModule`，而非直接引导 `AppComponent`；然后，该模块将创建组件。<br/>

我们的应用程序及其所有构建块都在 `AppModule` 中声明，这是合理的，所以让我们看看它里面是怎么回事。

**清单 2.2 `AppModule`**

```typescript
@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    CommonModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

嗯，这有点奇怪。 什么是 `CommonModule`?
`bootstrap: [AppComponent]` 到底是什么意思？
为什么这个类是空的？当然，所有这些问题都有合理的答案，但它们给我们带来了更多的复杂性
。所以，本质上，目前我们只需要理解模块 “声明” 相关内容，让应用程序能够运行，然后继续学习我们的构建块。
这个过程使得学习路径有点不线性，我们稍后将不得不重新审视这些概念才能真正理解它们，这可能会让初学者望而却步。

##### 教授 Angular

现在让我们来探讨第二种情景：我们是经验丰富的资深开发者，团队中新加入了一名成员。
我们通过逐步解释项目来帮助他们融入团队。以下是我们可能从新成员那里听到的一些问题：

* “为什么应用程序是这样构建的？”
* “你们有共享模块吗？哦，你们有两个？”
* “为什么既有核心模块又有共享模块？”
* “为什么共享模块会在延迟加载模块中？”

当然，可能还会出现许多其他问题。我们花费大量时间管理 `NgModule`，
并且每个处理不同项目的团队对于 `NgModule` 应该如何构建都有自己的看法，是否应该有共享模块等等。
这些细微差别对新手来说都相当难以解释，而且每次有人加入一个新的（现有的）`Angular` 项目时，
都感觉像是一次全新的体验，所有东西都得重新解释一遍。

我们还有另一层困惑，因为 `TypeScript` 和 `JavaScript` 本身就有模块的概念：我们把导出某些内容的文件称为模块；
在 `NodeJS` 中，我们使用 `module.exports`，
而在 `TypeScript` 中，我们使用 `export` 和 `import` 语句来使用其他文件中的函数、类和变量。
为什么在此之上还有一个模块概念呢？这些概念之间的区别是什么？

在 `Angular` 中，`NgModule` 用于对功能进行分组，并将所有依赖项（如管道、指令和其他组件）集中在一个称为编译上下文的地方。
有人声称 `NgModule` 简化了编译过程并帮助我们构建应用程序，但重要的是要明白，这也可以通过其他方式实现
—— 这些方式不涉及高级框架概念，并且可能会产生代码结构不佳的结果。

正如我们所见，这个讨论涉及到 `Angular` 团队的主要目标之一 —— 使 `Angular` 以及用它构建的项目更容易上手。
如果我们还记得这是 `Angular` 团队的主要目标之一，仅本章中的这些情景就足以让至少使 `NgModule` 成为可选的这个想法，
对核心团队和更广泛的社区都极具吸引力。但是等等，还有更多！


#### 2.1.2 间接性与样板代码

我们编写代码时，往往会将其拆分成不同部分，然后根据需要进行组装。
例如，我们可以把一个大型函数拆分成多个小函数，给它们取合适的名字，并在大函数中必要时调用它们，这样能让大函数可读性更强。
这种方法也有助于实现相同功能的复用。
在 `Angular` 中，我们将代码拆分为其各种构建块：组件、指令、管道和可注入服务。
由于可注入服务与模板无关，我们通常会将其重点放在前三个构建块上。每个构建块通常都放在各自独立的文件中，以保持代码整洁。

所以，比如说，要在另一个组件中使用子组件，我们需要导入它。
然而，导入过程并非直接进行（比如，不能只是简单地将另一个组件导入到我们组件的文件中），而是涉及到更复杂的思维模式。

让我们来看一个来自我们人力资源管理系统（`HRMS`）应用程序的组件示例，
该组件在其模板中使用子组件，应用场景是一个显示单个员工卡片的员工列表。

**清单 2.3 在父组件中使用的组件**

```typescript
import { Component, Input } from '@angular/core';
import { Employee } from '<path-to-type>';

@Component({
  selector: 'app-employees-list',
  template: `
    <div class="employee-container">
      <app-employee *ngFor="let employee of employees" 
        [employee]="employee">
      </app-employee>
    </div>
  `
})
export class EmployeeListComponent {
  @Input() employees: Employee[];
}
```

如我们所见，这个组件接收一个员工列表作为输入，并在 `for` 循环中渲染 `app-employee` 组件。
显然，它使用了另一个组件，所以必须从某个地方导入它，对吧？
但如果我们查看这个组件文件的导入语句，不会看到对该组件的任何导入。
当然，如果我们之前使用过 `Angular`，就会明白这是怎么回事，因为某个模块在某个地方进行了导入。

**清单 2.4 包含两个组件的员工模块**

```typescript
import { EmployeeComponent } from '<path-to-component>';
import { EmployeeListComponent } from '<path-to-component>';

@NgModule({
  declarations: [EmployeeComponent, EmployeeListComponent]
})
export class EmployeeModule { }
```

这种方法使得确定一个组件的依赖关系变得更加困难。等等，情况可能更糟！
`EmployeeListComponent` 可能会使用来自完全不同模块的组件。
所以，如果我们导航到 `EmployeeModule`，然后我们需要找到导出该组件的模块，而这个模块可能有一个像 `SharedModule` 这样模糊的名称。
反过来，这个组件必须将该组件添加到导出数组中，以此类推。
现代开发者工具（比如 `Angular Language Service`：[https://mng.bz/j07z](https://mng.bz/j07z)）在一定程度上缓解了这个问题，
它们提供了 IDE 工具和扩展，帮助开发者直接从一个组件/指令/管道导航到模板，
但是如果我们试图构建我们的组件树、应用程序结构或数据流的心理模型，这些工具只会偶尔提供帮助。
仍然很难找到某个依赖项必须导入的位置，并且我们仍然会有不必要的复杂程度。

可注入服务的提供者也颇具挑战性。
`NgModule` 确实有一个 `providers` 数组，但随着新选项的出现，开发者越来越倾向于仅将服务标记为 `providedIn: 'root'`，
以避免与 `NgModule` 打交道，并更清楚地了解服务的使用时间和方式。
另一方面，未标记为 `providedIn: 'root'` 的服务必须存在于整体路由系统的子路由中，
但又不在路由上定义，这使得仅通过查看应用程序架构很难理解服务何时首次实例化。

基于我们已经讨论过的内容，很明显我们还需要编写大量的样板代码。
我们到处都需要相当标准的 `NgModule` 定义，以及一些相关的约定。
有些 `NgModule` 的代码可能会增长到数百行。
我们不会提供这样一个 `NgModule` 的示例。
如我们所见，`NgModule` 解决了一个似乎已经有解决方案的问题，同时还引入了更多的复杂性和样板代码。但这还不止如此。

#### 2.1.3 `NgModule` 的其他问题

虽然我们已经讨论了主要问题，但 `NgModule` 的实现还存在一些其他问题。

* **分散的代码库**：使用 `NgModule` 时，很容易出现相关的应用程序模块分散在完全不同的 `NgModule` 中的情况，
这意味着它们不能直接一起使用。这本身就是一个问题，为了缓解这个问题，
开发者想出了不同的解决方案，这又增加了复杂性，或者只能依赖于专门用于共享功能的模块。
* **共享模块**：为了对相关功能进行分组，许多项目采用了共享模块（`SharedModule`）的概念，
这是一种专门设计用于声明和导出可重用组件、指令和管道的模块。
虽然这解决了前一点中提到的问题，但它也带来了其他问题，比如模块文件过大或共享模块过于分散。
如果开发者决定创建多个共享模块，就会产生另一个问题，即何时应该创建这样的模块以及它应该包含什么。
“每个延迟加载的功能模块都应该有自己的共享模块吗？” 这是一个有多种答案的合理问题，每个答案都不能说完全错误。
结果就是，我们的代码库变得不一致。
* **循环依赖和更难调试**：在有许多 `NgModule` 的代码库中，很容易引入循环依赖，这可能会成为一个很难解决的问题。
调试能力也会在一定程度上受到限制，例如，错误信息会更难读懂，因为 `Angular` 应用程序中的所有内容都被包裹在另一层抽象中。

所有这些因素可能会让我们觉得需要一种替代 `NgModule` 的方案。
所以，让我们来探索如何减少对它们的依赖。
但我们不会直接讨论独立组件，而是先花点时间看看在独立模式兴起之前的几年里，广大社区提出的一些解决方案。


### 2.2 先前的解决方案

独立的 `Angular` 组件在 v14 版本中首次作为开发者预览功能推出，并在 v15 版本中稳定下来，这意味着相对较新。
那么，开发者是如何克服我们之前列出的那些挑战的呢？让我们探索一些先前的方法，
或许我们能找到一种更轻松地迁移到 v16 及更高版本中独立模式的途径。

#### 2.2.1 分层共享模块

如前所述，开发者在他们的应用程序中使用特定的 “可共享” `NgModule` 来帮助进行代码拆分和关注点分离已变得很常见。
当某些构建块与应用程序内的某个功能并不完全对应，但同时又在多个地方需要时，这些共享模块就很有用。
我们曾采用最原始的方法创建一个单一的共享模块并在各处导入它，但我们发现这是一种相当痛苦的方法。
所以，在独立模式出现之前的 `Angular` 代码库中，人们经常会看到的不是一个大型的共享模块，
而是在应用程序的功能层级上有多个共享模块，每个模块都包含与该功能相关的可重用组件，并且在顶部有一个稍大的共享模块，
其中包含应用程序范围的构建块。图 2.1 是一个关于我们所描述的方法在人力资源管理系统（`HRMS`）应用程序上如何工作的示意图。

![HRMS 模块关系图](/assets/images/modern_angular/2-1.png)

如我们所见，这个过程相当直接：那些大型子模块中的每一个都可以有自己的路由子模块，
并且为了在这些模块之间共享特定于某个功能的功能，会创建一个共享模块。
这种方法使一个中等规模的应用程序易于管理，并且在多个 `Angular` 项目中都运行良好。
然而，它很难扩展，而且众所周知很难重构：如果有东西需要移动，它可能会迅速变得一团糟，
组件和其他构建块难以在应用程序结构中找到自己的位置。
所以，对于涉及多个团队的更大规模场景，已经创建了其他方法。


#### 2.2.2 SCAM 方法介绍

`SCAM`，即单组件 `Angular` 模块（`Single Component Angular Modules`），
是一种基于为每个可声明对象创建单个模块的原则来构建应用程序结构的方法。
例如，假设我们有一个组件，它不属于某个功能模块或共享模块，而是有一个专门用于声明它以及导入其依赖项的单个 `NgModule`。
举例来说，如果回顾我们之前的示例（图 2.1），我们可以看到两个属于某个功能模块的组件：
`EmployeeComponent` 和 `EmployeeListComponent`，它们都在 `EmployeeModule` 中声明。
采用 SCAM 方法，我们会为这两个组件分别创建特定的模块，随后将它们导入到需要的其他地方。
这些特定的 SCAM 模块通常在组件文件中声明，作为可以导入该组件的起点。

**清单 2.5 作为 SCAM 的 `EmployeeComponent`**

```typescript
import { Component, Input } from '@angular/core';
import { Employee } from '<path-to-type>';
import { EmployeeComponent } from '<path-to-component-scam-module>';

@Component({
  selector: 'app-employee-list',
  template: `
    <div class="employees-container">
      <app-employee *ngFor="let employee of employees" 
        [employee]="employee">
      </app-employee>
    </div>
  `
})
export class EmployeeListComponent {
  @Input() employees: Employee[];
}

@NgModule({
  declarations: [EmployeeListComponent],
  imports: [EmployeeComponent]
})
export class EmployeeModule { }
```

**#1**：组件本身的声明 <br/>
**#2**：通过其自身的 SCAM 模块导入另一个组件 <br/>
**#3**：父组件的 SCAM 模块 <br/>

请注意，我们将这种方法命名为单组件 `Angular` 模块，并且对于指令和管道，我们也必须采用相同的方法。
值得注意的是，即使组件是路由组件（意味着某个路由指向它），我们在这些 SCAM 中也不添加路由，
并且这些模块仅作为声明组件/指令/管道和导入其依赖项的地方。

这种方法有几个好处：

* 更容易跟踪依赖关系：它们列在同一个文件中。
* 更容易在应用程序中重构和移动：我们可以直接获取这个文件并移动到任何其他地方。
* 更容易进行单元测试：我们只需要模拟这个组件的直接依赖项，并且可以使用非浅渲染模板（按原样渲染模板）。
* 更好的代码拆分：我们可以直接延迟加载组件本身，使应用程序包非常小且粒度更细。
* 更容易迁移到独立模式：随着独立构建块的兴起和迁移示意图的引入，SCAM 出现了一个新的潜在好处，我们将在本章后面的部分讨论；
简而言之，SCAM 使得运行示意图并轻松获得一个完全独立的应用程序变得非常容易。

当然，这些好处很不错，但 `NgModule` 的一些缺点仍然存在：

* **额外的样板代码**：虽然理解应用程序结构的思维负担减轻了，但样板代码实际上却增加了；我们现在为每样东西都创建了一个 `NgModule`。
* **仍然有点难以解释和上手**：新手仍然需要学习 `NgModule`，如果新开发者不知道 SCAM 模式，加入项目时的入门过程会更复杂一些。
* **其他模块仍然可能很大**：例如，我们仍然需要导入 `CommonModule` 才能使用诸如 `*ngIf` 或 `[ngClass]` 这样的指令，
并且可能会在我们的包中添加很多我们实际上并不使用的引用。

考虑到这些优点和缺点，开发者们一直在等待机会尝试摆脱 `NgModule`。接下来，让我们最终讨论编写 `Angular` 构建块的新独立方法。

### 2.3 不使用 NgModule 开发应用程序

在开始之前，让我们简要了解一下，当我们采用这种独立模式进行开发时，它会带来哪些优势：

* 我们可以完全以独立模式构建应用程序，就像我们在上一章中初始化的新版人力资源管理系统（HRMS）应用程序那样。
我们将学习在没有 `NgModule` 的情况下，所有这些是如何运作的。
* 独立模式具有向后兼容性，所以 `NgModule` 和独立组件可以（并且通常确实）在同一个代码库中共存。
* 许多第三方工具和库仍然没有从 `NgModule` 迁移过来，因此独立模式和模块之间具有完全的互操作性。我们将学习如何处理它们之间的连接。

我们将从在一个全新的项目中构建独立组件开始，因为我们已经有了独立的应用程序，
然后继续展示这个应用程序如何仍然与其他 `NgModule` 完全兼容，以及那些没有独立 API 的旧工具如何与独立构建块集成。
让我们开始吧！


#### 2.3.1 创建我们的第一个独立组件

在每个企业应用程序（以及许多非企业应用程序）中，用户的旅程都从一个起点开始：通常是登录页面。
让我们为我们的人力资源管理系统（HRMS）应用程序构建一个登录页面。我们将使用模板驱动表单（因为它们相当简单），
并且如果表单被认为有效，就进行 HTTP 调用。首先，让我们创建组件本身。

##### 创建组件

我们先在 `src/app` 目录下创建一个名为 `pages` 的文件夹，这样我们就可以把路由组件放在里面，包括第一个组件。
现在不用担心架构或文件夹结构：我们稍后会重构和整理这些内容。
我们将在这个文件夹中创建一个名为 `login.component.ts` 的文件，并把组件代码放在里面，如下所示。

**清单 2.6 独立的 `LoginComponent`**

```typescript
@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <h1>Login</h1>
      <form>
        <input type="text" name="email" placeholder="Email">
        <input type="password" name="password" placeholder="Password">
        <button type="submit">Login</button>
      </form>
    </div>
  `,
  standalone: true
})
export class LoginComponent {
  credentials = { email: '', password: '' };
}
```

这似乎是一个相当简单的组件，但有一个关键区别：
在它的元数据中，我们可以看到 `standalone: true`，这表明这个组件是独立的，不属于任何 `NgModule`。
因此，我们不能在任何 `NgModule` 中声明它，并且与这个组件相关的所有内容（例如，它导入的其他组件或指令）都在这个类中。

##### 将 `NgModule` 导入独立组件

一个很明显的问题是，为了使这个组件真正成为一个登录组件，我们需要为它添加一些交互性。
也就是说，当用户在输入框中输入内容时，我们希望凭证属性能够更新。
实现这一点的一种流行方法是使用 `ngModel` 指令，但问题在于：
这个组件不是任何 `NgModule` 的一部分，那么它如何导入 `FormsModule` 才能使用 `ngModel` 呢？
嗯，当我们将一个组件标记为独立时，我们可以在它的元数据中添加一个特殊的 `imports` 属性，该属性将列出该组件的所有依赖项。
让我们继续更新这个组件。

**清单 2.7 导入模块的独立组件**

```typescript
@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <h1>Login</h1>
      <form>
        <input type="text" name="email" placeholder="Email" 
          [(ngModel)]="credentials.email">
        <input type="password" name="password" placeholder="Password" 
          [(ngModel)]="credentials.password">
      </form>
      <button type="submit">Login</button>
    </div>
  `,
  standalone: true,
  imports: [FormsModule]
})
export class LoginComponent {
  credentials = { email: '', password: '' };
}
```

**#1**：使用 `ngModel` 指令 </br>
**#2**：通过组件自身的元数据将 `FormsModule` 直接导入到组件中 </br>

如我们所见，组件元数据上的 `imports` 属性的作用与在 `NgModule` 中完全相同：
它允许我们导入包含组件/指令/管道的 `NgModule`，这些组件/指令/管道是我们的组件运行所必需的。
我们之前讨论过独立组件与 `NgModule` 的互操作性，在这里我们看到了它的实际应用：
一个独立组件可以充当它自己的 `NgModule` 并与 `NgModule` 协同工作。
但是等等，还有更多内容。
现在让我们添加一条验证消息，只要有一个必填输入项未填写，该消息就会显示出来。

**清单 2.8 在独立组件中使用 `*ngIf`**

```typescript
template: `
  <div class="login-container">
    <h1>Login</h1>
    <form>
      <input type="text" name="email" placeholder="Email" 
        [(ngModel)]="credentials.email">
      <input type="password" name="password" placeholder="Password" 
        [(ngModel)]="credentials.password">
    </form>
    <button type="submit">Login</button>
    <span class="warning" *ngIf="!credentials.email ||!credentials.password">
      Please fill in all the required fields
    </span>
  </div>
`
```

##### 向独立组件中导入独立指令

现在，我们需要一种方法来让我们的组件知道 `*ngIf` 是什么。
当然，我们可以像之前处理 `FormsModule` 那样，把包含 `NgIf` 指令的 `CommonModule` 添加到我们组件的 `imports` 数组中，这样肯定是可行的。
但这会导致导入 `CommonModule` 中包含的所有其他组件和指令，比如 `JsonPipe` 或 `NgClass`，而在这个组件中我们并不需要它们。

那么我们该如何解决这个问题呢？`Angular` 团队已经为我们考虑到了：
从 v15 版本开始，`CommonModule` 中的所有实体都是独立的，所以我们可以不用导入整个模块，而是直接把我们需要的东西导入到组件中。
让我们把 `NgIf` 指令引入到登录组件中：

```typescript
imports: [FormsModule, NgIf]
```

现在，我们的组件将知道 `*ngIf` 的含义，与该指令选择器匹配，并让它运行，在用户填入必要的数据之前一直显示警告信息。
如果我们回忆一下之前关于 SCAM 方法的部分，这可能会让人感觉非常相似。
从本质上讲，我们做的事情和使用 SCAM 模块时一样，但增加了可以直接将之前与模块相关的元数据写入组件的能力。
这就是为什么我们说使用 SCAM 在实际上使得自动迁移到独立模式变得非常容易。

##### 在独立组件中提供服务

现在我们来关注最后一个问题：通过向身份验证 API 发起 HTTP 调用，将这个组件变成一个真正的登录组件。
为此，我们将在应用程序目录下创建一个 `services` 文件夹，并在其中添加一个 `auth.service.ts` 文件，内容如下所示。

**清单 2.9 身份验证服务**

```typescript
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  constructor(private http: HttpClient) {}

  login(credentials: { email: string, password: string }) {
    return this.http.post('/api/auth/login', credentials);
  }
}
```

这段代码相当通俗易懂；问题是，我们如何让登录组件知道这个服务呢？
和 `NgModule` 一样，我们可以将这个服务添加到独立组件的 `providers` 数组中，然后直接使用它。
下面的清单提供了包含 HTTP 调用的完整组件代码。

**清单 2.10 独立登录组件的最终版本**

```typescript
import { Component } from '@angular/core';
import { NgIf } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { AuthService } from '../services/auth.service';

@Component({
  selector: 'app-login',
  template: `
    <div class="login-container">
      <h1>Login</h1>
      <form>
        <input type="text" name="email" placeholder="Email"
          [(ngModel)]="credentials.email">
        <input type="password" name="password" placeholder="Password" 
          [(ngModel)]="credentials.password">
      </form>
      <button type="submit" (click)="submit()">Login</button>
      <span class="warning" 
        *ngIf="!credentials.email ||!credentials.password">
        Please fill in all the required fields
      </span>
    </div>
  `,
  standalone: true,
  imports: [FormsModule, NgIf],
  providers: [AuthService]
})
export class LoginComponent {
  credentials = { email: '', password: '' };

  constructor(private authService: AuthService) {}

  submit() {
    if (this.credentials.email && this.credentials.password) {
      this.authService.login(this.credentials).subscribe();
    }
  }
}
```

我们可能会注意到，虽然这个组件是有功能的，但它目前的作用还不大，因为它没有被路由，也没有在应用程序的任何地方建立连接；它只是单独存在。
接下来，让我们通过路由将它连接到我们的应用程序中，看看它实际运行的效果。

#### 2.3.2 独立组件的路由与依赖项提供

我们之前已经讨论过在 `NgModule` 中路由是如何工作的，也了解了在独立模式设置中路由应该如何定义。
现在，让我们弄清楚如何连接我们的可注入依赖项（DI），并使我们的组件可路由。

首先，让我们回顾一下 DI 在 `Angular` 中是如何工作的。
我们不会深入探讨，因为下一章专门用于全面理解 DI，但我们会研究两个核心原则：如何提供依赖项以及如何注入它。
我们从后者开始，以便更好地理解整个过程。

在 `Angular` 中注入依赖项归根结底是定义某个东西的令牌，并告诉框架我们希望在某个地方使用它。
如清单 2.10 所示，我们通过添加构造函数参数来定义我们想要的东西：`private authService: AuthService`。
看到这个后，`Angular` 会（以某种方式）为我们获取该服务的一个实例。

但它怎么知道在哪里找到实际的实例呢？清单 2.9 表明 `AuthService` 本质上只是一个类，并且可能有多个实例。
它还接收一个 `Angular` 的 `HttpClient` 实例，那么这个类现在怎么知道从哪里获取那个实例呢？
嗯，这里就是等式的第二部分发挥作用的地方：提供者（`providers`）。
提供者允许我们为可以注入的实体定义值。
从本质上讲，它们是我们对 `Angular` 说 “亲爱的 `Angular`，如果你看到这个令牌（例如，我们示例中的 `AuthService` ），
请注入这个值（那个特定的实例）” 的地方。

在使用 `Angular` v13 或更早版本时，我们知道有两种提供这些值的方法：
一是添加到某些 `NgModule` 或组件的 `providers` 数组中，二是将服务标记为 `providedIn: 'root'`。
在清单 2.10 中，我们将 `AuthService` 直接添加到了 `LoginComponent` 的 `providers` 中，这样就完成了操作。
这确实是最简单的方法，但它只是为当前阶段设计的，当应用程序规模扩大时并不适用。
它还引发了一些问题，特别是在模块化应用程序的情况下，如表 2.1 所示。

| 问题                             | 无独立 API 的答案                         | 有独立 API 的答案                                    |
|--------------------------------|-------------------------------------|------------------------------------------------|
| 我是否需要在每个组件中提供每个依赖项？            | 要么这样做，要么将服务标记为 `providedIn: 'root'` | 如果有独立 API，我们可以使用它                              |
| 我如何导入在其他模块中提供的服务（没有特殊的独立 API）？ | 直接将模块导入到组件中                         | 使用 `src/main.ts` 文件中的 `importProvidersFrom` 函数 |
| 我如何导入 Angular 内置依赖项？           | 导入整个模块                              | 使用现有的独立 API                                    |


在 “有独立 API 的答案” 这一列中，我们定义了 “现有独立 API” 的概念，以及一个名为 `importProvidersFrom` 的函数。
让我们来了解一下这个函数试图为我们的应用程序提供 `HttpClient` 的作用。
我们可以记住，这些依赖项存在于 `@angular/common/http` 包中，名称为 `HttpClientModule`。
另外，在上一章中，我们在 `src/app/app.config.ts` 文件中遇到了一个带有 `providers` 属性的配置对象。
这就是在独立的 `Angular` 应用程序中提供依赖项的地方。
所以，我们将把 `HttpClientModule` 的提供者添加到这个数组中。
为了做到这一点，我们将使用 `importProvidersFrom` 函数。
让我们打开 `src/app/app.config.ts` 文件，我们会看到提供的路由。
在它们旁边，我们将添加新的导入。

**清单 2.11 从其他模块添加提供者**

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    importProvidersFrom(HttpClientModule)
  ]
};
```

`importProvidersFrom` 函数接受一个模块（或多个模块；我们可以提供多个），提取它们的提供者，并将其传输到我们想要使用的地方。
这个函数非常简单，因为它仅用于提供独立模式和 `NgModule` 之间的互操作性。

重要的是要理解，此方法仅在应用程序初始化上下文或在 `bootstrapApplication` 函数中的某些路由提供者中起作用
（我们将在下一节中更多地讨论路由提供者）。 接下来，我们应该意识到这适用于任何 `NgModule`；
所以，如果我们使用的一些第三方 `Angular` 库尚未迁移到独立模式，我们仍然可以使用 `importProvidersFrom` 来继续充分使用该库。
最后，在处理一些 `Angular` 内置模块时，我们实际上可以放弃 `importProvidersFrom`。
我们强调 “一些”，是因为并非所有 `Angular` 内置 API 都已从 `NgModule` 迁移。
但是，正如我们在路由（`provideRouter`函数）中看到的，大量的 `Angular` 模块已经迁移到了独立 API，包括 `HttpClient`。
这是我们真正希望在应用程序的 `app.config.ts` 文件中提供 `HttpClient` 的方式：

**清单 2.12 从其他模块添加提供者**

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient()
  ]
};
```

我们可以注意到 `provideSomething` 的命名约定。
这种模式现在被许多库作者广泛采用，所以我们要做好准备，在各种代码库中遇到这样的函数。
另外，如果我们是库的作者，以类似的方式导出提供者被认为是一种良好的实践。

那么，我们如何提供 `AuthService` 呢？
嗯，我们可以直接将它添加到这个根级别的提供者数组中，但我们不这么做，而是简单地将它标记为 `providedIn: 'root'`，
这样 `app.config.ts` 文件就不会被改动。

最后，我们需要为登录组件创建一条路由，使它成为我们应用程序第一个可用的功能。
我们很快会发现，在这方面实际上没有什么变化。

**清单 2.13 使用 `HttpClient` 独立 API**

```typescript
import { Routes } from '@angular/router';
import { LoginComponent } from './pages/login.component';

export const routes: Routes = [
  { path: '/login', component: LoginComponent }
];
```

当然，这种对可路由独立组件的简单注册，只在我们有一个简单的、急切加载的组件时才有效。
如果我们想要延迟加载一条路由会怎样呢？
我们现在没有模块了，所以不能使用旧方法。
现在让我们直接讨论延迟加载组件。

### 2.4 延迟加载组件

在开始之前，让我们简要回顾一下在独立模式出现之前，`Angular` 应用程序中是如何实现延迟加载的。
由于所有组件和其他构建块都被打包在 `NgModule` 中，我们不能直接延迟加载组件
（实际上，某种程度上我们可以，但不能通过路由，而我们这里严格讨论的是路由），
相反，我们延迟加载 `NgModule`，然后这些 `NgModule` 再加载它们自己的路由模块。
最后，它加载我们想要的组件（或者甚至可能是另一个具有相同过程的延迟加载模块）。

#### 2.4.1 使用 `NgModule` 进行延迟加载

在本节中，我们将向应用程序添加延迟加载组件；
具体来说，我们想要开始实现应用程序的 “员工” 功能，并添加一个注册页面。
让我们首先研究如何使用 `NgModule` 来实现前者。
图 2.2 说明了 `Angular` 如何通过 `NgModule` 延迟加载组件。

![延迟加载组件](/assets/images/modern_angular/2-2.png)

如图所示，这是一个相当复杂的过程。
例如，如果 `EmployeesRoutingModule` 实际上又延迟加载了另一个 `NgModule` 等等，这个过程可能会变得更加困难。
此外，即使我们有一个想要延迟加载的组件，我们也需要为它创建一个模块（如果它在逻辑上不属于现有的某个功能模块）。
使用 SCAM 方法可以缓解这个问题，但在其他场景中我们会受到很大限制。
当然，无论哪种方式，我们都会得到一堆我们想要避免的样板代码。

#### 2.4.2 延迟加载单个独立组件

现在，我们将探索相同的过程在独立组件中是如何工作的。 让我们创建一个注册组件，并使其可路由。
在 `src/app/pages` 目录中，创建一个名为 `registration.component.ts` 的文件，并添加 `RegistrationComponent`。
我们希望在用户首次导航到我们的应用程序时显示登录页面，并且还有一个单独的 `/register` 路由，它将延迟加载 `RegistrationComponent`。
以下清单提供了实现此功能所需的代码。

**清单 2.14 直接延迟加载组件**

```typescript
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path:'register',
    loadComponent: () => {
      return import('./pages/registration.component')
        .then((m) => m.RegistrationComponent);
    }
  }
];
```

请注意，在路由对象上添加了特定的 `loadComponent` 选项。
此选项接受一个函数，该函数将导入模块（在这种情况下，我所说的模块是指包含代码的文件，而不是 `NgModule`）并加载该特定组件。
在这种情况下，该过程比图 2.3 中描述的要简单得多，主要是完全跳过了加载任何 `NgModule`。

![延迟加载独立组件](/assets/images/modern_angular/2-3.png)

有两个重要点需要注意：

* **组件本身没有变化**：新的路由方法在任何方面都不会影响我们编写组件的方式。
从本质上讲，当我们将一个可路由组件迁移为独立组件时，我们只需要将其代码标记为 `standalone: true`。
* **路由对象没有变化**：除了使用新的 `loadComponent` 选项之外，与我们之前定义路由的方式没有任何不同；
我们可以添加守卫、解析器、路由或查询参数等等。

到目前为止，我们已经延迟加载了单个组件。
虽然这是一个有效的用例，但在现实生活中，我们可能希望延迟加载一组组件，
这在以前是通过延迟加载整个包含其所有可路由组件的 `NgModule` 来实现的。
让我们看看如何在没有 `NgModule` 的情况下实现相同的事情。

#### 2.4.3 延迟加载多个独立组件

在开始之前，让我们先理解为什么我们希望具备同时延迟加载多个路由的能力。
这种场景之所以重要，有以下三个原因：

* **代码管理**：如果我们持续添加更多延迟加载的组件，`app.routes.ts` 文件可能最终会变得难以管理，有可能包含上千行甚至更多的代码
—— 这样就不太容易进行导航和排查问题。
* **功能分组**：其次，我们可能认为从逻辑上讲属于同一组的多个组件
（例如，`EmployeeList`、`EmployeeDetails`、`CreateEmployee` 和 `EditEmployee`）
可以很容易地一起进行延迟加载，而不会对性能造成显著影响。
以前，这很难实现，因为这些组件很可能已经属于某个功能 `NgModule`，并一起被延迟加载。
有了独立组件模式，它们可以位于任何位置并一起被加载。
* **文件夹结构优化**：最后但同样重要的是，我们可能希望文件夹结构能更好地反映应用程序整体以及路由中的层次结构，
因此需要多个 `routes.ts` 文件，这些文件存在于包含特定功能相关代码的子文件夹中。
同样，独立组件模式无需借助 `NgModule` 就能解决所有这些问题。

现在让我们创建第一个功能模块，并完善应用程序的 “员工” 部分。
如前所述，它将包含允许用户查看员工列表、特定员工详细信息以及创建和编辑员工的组件。

在 `src/app/pages` 文件夹中，我们将创建一个名为 `employees` 的目录，该目录将包含所有与员工相关的代码。
在这个文件夹中，我们将创建所有提到的组件。
这些组件目前还不可路由，因为我们不想将它们放在 `app.routes.ts` 中，而是希望为它们设置单独的路由配置文件。
让我们在同一个目录中创建一个名为 `employees.routes.ts` 的文件，并将以下清单中的代码放入其中。

**清单 2.15 添加功能级路由**

```typescript
export const routes: Routes = [
  { path: 'list', component: EmployeeListComponent },
  { path: 'details/:id', component: EmployeeDetailsComponent },
  { path: 'create', component: CreateEmployeeComponent },
  { path: 'edit', component: EditEmployeeComponent }
];
```

如我们所见，添加这样的功能路由或子路由与直接在 `app.routes.ts` 中添加它们完全没有区别。
然而，这些路由目前尚未与我们的根路由连接。
我们可以通过添加一个带有 `loadChildren` 的路由来完成此操作，在这个路由文件中直接导入路由，无需任何中间的 `NgModule`。

**清单 2.16 延迟加载单个和多个独立组件**

```typescript
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path:'registration',
    loadComponent: () => {
      return import('./pages/registration.component')
        .then((m) => m.RegistrationComponent);
    }
  },
  {
    path: 'employees',
    loadChildren: () => {
      return import('./pages/employees/employees.routes')
        .then((m) => m.routes);
    }
  }
];
```

同样，与 `NgModule` 设置相比，唯一的关键区别在于 `loadChildren` 函数导入的是路由，而不是导入一个会再导入路由模块的模块。
`loadChildren` 函数的工作方式与之前讨论的相同，并且实现了相同的结果，只是方式更加直接。
这个过程的思维模型也比使用 `NgModule` 更简单。

![从路由延迟加载独立组件](/assets/images/modern_angular/2-4.png)

最后，我们想要解决的最后一个问题是能够仅为某些路由提供一些依赖项。
例如，`EmployeeService` 仅在与我们应用程序中员工功能相关的组件中可用。

#### 2.4.4 仅为特定路由提供依赖项

使用 `NgModule` 时，我们可以通过将某些服务和其他依赖注入（DI）令牌添加到特定延迟加载模块的 `providers` 数组中，
从而在这些模块中提供它们。这种方法有两个好处：
一是能够延迟加载服务，使应用程序更加细粒度；
二是可以为不同的模块（路由）提供同一服务的不同实例。
最后这一点对使用状态管理库（如 `NgRx` 和 `NgXS`）的开发者特别有用，
这些库允许他们仅将应用程序状态的某些部分提供给特定路由， 并使其与周围的代码库更加隔离。
这既能简化应用程序状态（我们会有几个较小的状态对象，当用户访问应用程序的那些部分时才会被加载，而不是一个庞大的对象），
又有助于避免状态加载过早的情况。例如，可能会在不需要的时候进行一些 HTTP 请求，从而导致数据过时或不必要的服务器负载。

在独立组件的场景中，我们没有 `NgModule` 来提供新的依赖项，所以我们需要寻找基于独立组件的解决方案。
我们可能已经注意到一种模式：路由定义的行为常常和 `NgModule` 类似（但仅限于延迟加载方面）。
事实证明，这次也没有什么不同：现在可以在路由定义中添加一个 `providers` 数组。

为了实现这一点，让我们在 `src/app/services` 目录中创建一个 `EmployeeService`，并在其中编写一些负责 HTTP 调用的代码。
实际代码并不重要，所以我们暂时省略它；只要它没有被标记为 `providedIn: 'root'` 就行：

```typescript
@Injectable()
export class EmployeeService {}
```

接下来，我们要去 `app.routes.ts` 文件，就是我们连接延迟加载员工路由的地方。
我们可以按照以下清单修改特定的路由定义。

**清单 2.17 向延迟加载的路由添加提供者**

```typescript
{
  path: 'employees',
  providers: [EmployeeService],
  loadChildren: () => {
    return import('./pages/employees/employees.routes')
      .then((m) => m.routes);
  }
},
```

这段代码将使 `EmployeeService` 仅在 `employees.routes.ts` 中的路由中可用，
这意味着只有在该配置文件中路由的组件才能访问 `EmployeeService` 的这个特定实例。
例如，我们可以通过导航到 `LoginComponent` 并通过构造函数注入 `EmployeeService` 来轻松检查这一点。
如果我们在浏览器中检查应用程序，肯定会看到以下错误：
`“NullInjectorError: No provider for EmployeeService!”` 
以及一些更多的解释性细节。
我们现在已经成功地将这个服务隔离到了我们路由的一个子集中。

#### 2.4.5 向另一个组件延迟加载组件

到目前为止，我们讨论了通过路由延迟加载组件（例如，有一条路由仅在用户导航到它时才会加载其组件）。
但如果我们想要更细粒度的控制呢？让我们设想以下场景：
我们正在构建 `EmployeeListComponent`，并在其中放置一个员工表格，
表格将包含每个员工的全名、年龄、职位等数据，以及一个操作列，允许我们编辑或删除员工。
当用户点击 “删除” 按钮时，我们希望向他们显示一个确认对话框，询问他们是否确定要删除该员工。
但问题在于：确认对话框将是一个独立的、可复用的组件（我们可能希望在多个页面上都有这样的确认逻辑），
并且我们不希望应用程序在用户点击 “删除” 之前下载该组件的代码。
毕竟，该组件在那之前是不可见的，在真正需要使用它之前拥有这段代码是没有意义的。

在这种情况下，我们不能依赖通过路由进行延迟加载。
从某种意义上说，我们可以使用命名路由器出口（named router outlets）（[https://mng.bz/yowp](https://mng.bz/yowp)）
和辅助路由（secondary router）（[https://mng.bz/M1nQ](https://mng.bz/M1nQ)）来实现这一点。
然而，在这里，我们不想要一个单独的可路由组件；我们只想要另一个可以纯粹按需加载和显示的组件。
这项任务相当容易完成，尤其是对于独立组件。让我们首先创建 `EmployeeListComponent`。
将清单 2.18 中的代码添加到 `src/app/pages/employees/employee-list.component.ts` 文件中。

**清单 2.18 `EmployeeListComponent`**

```typescript
@Component({
  selector: 'app-employee-list',
  template: `
    <h2>Employees List</h2>
    <table>
      <thead>
        <tr>
          <th>Full Name</th>
          <th>Position</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let employee of employees$ | async">
          <td>{{ employee.firstName }} {{ employee.lastName }}</td>
          <td>{{ employee.position }}</td>
          <td>
            <button (click)="showConfirmationDialog()">Delete</button>
          </td>
        </tr>
      </tbody>
    </table>
  `,
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf, NgComponentOutlet]
})
export class EmployeeListComponent {
  employees$ = this.employeeService.getEmployees();
  isConfirmationOpen = false;

  constructor(private readonly employeeService: EmployeeService) {}

  showConfirmationDialog() {
    this.isConfirmationOpen = true;
  }
}
```

如我们所见，我们有了表格，但还没有对话框。我们可以通过三个步骤向其中添加一个动态加载的组件：

1. 在一个新文件 `src/app/shared/components/confirmation-dialog.component.ts` 中创建 `ConfirmationDialogComponent`。 
2. 添加一个动态导入函数调用，以异步方式导入该组件。 
3. 通过异步管道将得到的 `Promise` 传递给 `ngComponentOutlet` 指令。

让我们用代码实现，然后看看它是如何工作的。
我们将在 `src/app` 目录下创建一个名为 `shared` 的新文件夹，
我们将把应用程序的可复用组件放在这里，包括 `ConfirmationDialogComponent`。
以下清单提供了必要的代码。

**清单 2.19 `ConfirmationDialogComponent`**

```typescript
@Component({
  selector: 'app-confirmation-dialog',
  template: `
    <dialog [open]="isConfirmationOpen">
      Are you sure you want to perform this action?
      <button (click)="isConfirmationOpen = false">Cancel</button>
      <button (click)="isConfirmationOpen = false">Confirm</button>
    </dialog>
  `,
  standalone: true
})
export class ConfirmationDialogComponent {
  @Input() isConfirmationOpen = true;
}
```

如我们所见，这个组件非常简单，功能不多，但目前够用了。
接下来，我们需要在用户点击 “删除” 按钮时动态导入这个组件。
我们通过将 `showConfirmationDialog` 方法设为异步函数，
并将组件加载到一个属性中以便后续使用其模板来实现这一点。

**清单 2.20 按需导入组件**

```typescript
export class EmployeeListComponent {
  employees$ = this.employeeService.getEmployees();
  isConfirmationOpen = false;
  confirmDialog: any = null;

  constructor(private readonly employeeService: EmployeeService) {}

  async showConfirmationDialog() {
    this.confirmDialog = await import(
      '../../shared/components/confirmation-dialog.component'
    ).then((m) => m.ConfirmationDialogComponent);
    this.isConfirmationOpen = true;
  }
}
```

`confirmDialog` 属性将保存对 `ConfirmationDialogComponent` 类的引用
（重要的是，是类本身，而不是它的实例 —— 实例尚未创建）。
如我们所见，每当点击 “删除” 按钮时，我们想要的组件将在那时动态加载，而不会更早。
最后，最后一步将是在视图中渲染此组件。我们可以使用 `ngComponentOutlet` 指令来实现这一点。
所以，我们在 `EmployeeListComponent` 的模板底部添加一行代码：

```html
<ng-container *ngComponentOutlet="confirmDialog"></ng-container>
```

由于这是一个对话框组件，我们可以将它放在模板中的任何位置，不会有太大区别。
如果我们想在模板中的特定位置加载该组件，我们可以将其放在那个确切的位置。
但这里发生了什么呢？看，`ngComponentOutlet`指令接收一个组件类的引用，
并将其渲染到 `ng-container` 中，我们把该指令放在这个 `ng-container` 里。
在这种情况下，最初，`confirmDialog` 属性为 `null`，所以不会进行渲染，`ConfirmationDialogComponent` 也尚未加载。
当用户点击 “删除” 按钮时，我们加载组件类并将其放入一个属性中，这会触发重新渲染，并动态且延迟地显示我们的确认对话框。
与在 `NgModule` 中声明的组件相比，延迟加载独立组件的优势在于无需导入整个模块或将其作为输入传递给 `ngComponentOutlet`。

理想情况下，我们希望有一种方法与动态创建的组件实例进行通信。
我们可以通过几种方式做到这一点，包括将输入传递给子组件或获取该组件的引用并对其进行操作。
我们将在第 4 章研究此类操作的示例，在那里我们将探索组件（不一定是独立组件）的新功能。

### 2.5 迁移与常见陷阱

到目前为止，我们已经研究了在一个空应用程序中从头开始创建和使用独立组件的情况。
我们还了解到可以在基于 `NgModule` 的应用程序中轻松创建和使用独立组件。
但如果我们想要将现有的整个应用程序迁移为完全使用独立组件呢？迁移完成后可能会遇到哪些问题呢？
让我们研究几种可以采用的方法，并直接了解这些问题。

#### 2.5.1 手动迁移

大型项目可能有多个相互交织的 `NgModule`，很难一次性（甚至自动）轻松拆解。
如果代码库处于活跃状态（有多个开发者提交新代码），采用渐进式方法可能很有吸引力：
正如我们所记得的，独立组件、指令和管道与 `NgModule` 完全互操作，这意味着我们可以挑选要首先转换的组件。
这种确定可以通过四个步骤完成：

1. 采用将所有新组件、指令和管道编写为独立组件的规则。
2. 选择我们想要转换的组件，并将它们标记为 `standalone: true`。
将该组件从 `NgModule` 的 `declarations` 数组移动到 `imports` 数组
（独立组件在原地声明，不能在 `NgModule` 中重新声明）。
如果该组件也在另一个 `NgModule` 中使用，将其添加到 `exports` 数组中（它可能已经在那里了）。
以项目可管理的规模进行迭代操作。
3. 逐个移除 `NgModule`，直到只剩下 `AppModule`。
这意味着我们将不得不手动将新的独立组件导入到需要它们的地方，直到所有构建错误消失。
4. 当只剩下 `AppModule` 时，移除它，并将提供者移动到 `main.ts` 中，
同时切换到新的独立提供者（如`provideRouter`、`provideHttpClient`等）。

这种方法在小型项目上可能或多或少能顺利进行，对于没有太多 `NgModule` 的中型项目也许也行得通。
然而，对于大型项目，这种方法可能耗时太长，这就是为什么我们可以采用 SCAM 方法来简化它。


#### 2.5.2 使用单组件 `Angular` 模块（SCAM）

我们已经讨论过 SCAM 方法；现在让我们看看它如何帮助我们过渡到独立模式。
我们将采用上一节手动迁移中的大部分步骤，但会做一些修改：

1. 同样，将所有新的构建块都编写为独立的。
2. 选择我们希望首先转换的组件，但不是将它们变成独立组件，而是为它们创建一个 SCAM 模块，并将该组件保留在其中。
该模块不会存在于任何其他 `NgModule` 中。在项目团队觉得最方便的时候进行此操作。
3. 当所有组件、指令和管道都已转换为 SCAM 模块（在这些 SCAM `NgModule` 之外），
我们将只剩下功能 `NgModule`（例如我们 HRMS 应用程序示例中的`EmployeeModule`、`WorkModule`等）。
一旦完成此操作，我们就可以开始将所有组件、指令和管道标记为独立的，并移除它们的 SCAM 模块。
这个过程应该很容易，因为模块将在同一个文件中。然后，我们需要找到对 SCAM 模块的引用，并直接用组件替换它。
我们可以通过简单的查找和替换来做到这一点。虽然这个过程看起来有点令人生畏，但实际上可以使用基本的 IDE 功能一次性完成。
4. 完成步骤 3 后，我们可以开始移除功能模块，并将它们的可声明组件/指令/管道标记为独立的，并将已经是独立的（以前的 SCAM 模块）导入其中。
5. 现在，移除 `AppModule` 并使用独立的提供者 API。

这种方法虽然步骤更多，但可能更直观，并且主要部分可以一次性完成。
它还将结构简化到一个程度，使得独立迁移示意图可以安全地完成其工作。接下来让我们讨论这个主题。

#### 2.5.3 使用 `schematic` 命令进行迁移
在 `Angular` v16 版本中，`Angular` 团队引入了一种特殊的 `schematic`（架构指令），它有助于将基于传统 `NgModule` 的应用程序自动转换为完全独立的应用程序。这项任务可以通过几种方式完成。让我们从最基本的方式开始：立即运行命令，步骤如下：

1. 确保我们要升级的应用程序是 v16 版本。
2. 在应用程序的根文件夹中打开一个终端窗口，并运行 `ng g @angular/core:standalone`。
3. 将会出现三个选项： “将所有组件、指令和管道转换为独立的”、
“移除不必要的 `NgModule` 类” 以及“使用独立 API 引导应用程序”。
根据 `Angular` 文档（[https://mng.bz/aV8j](https://mng.bz/aV8j)），我们应该按此顺序运行它们，所以让我们选择第一个选项。
4. 第一个选项会将所有组件/指令/管道转换为独立的，添加它们需要导入的任何内容（例如，在其模板中使用的指令），
并使它们被导入而不是在其 `NgModule` 中声明。运行应用程序以确保在此步骤中没有出现问题。
5. 接下来，再次运行相同的命令并选择 “移除不必要的 `NgModule` 类”。
此步骤将移除所有不属于应用程序结构的模块（例如，SCAM 模块），使我们得到一个基本为独立模式的项目。
同样，让我们运行应用程序并检查一切是否正常。
6. 最后，再次运行该命令并选择 “使用独立 API 引导应用程序”。
此步骤将移除`AppModule`，并在`main.ts`中使用`bootstrapApplication`函数。

现在，这种迁移并非万无一失，在大多数情况下，我们还需要做一些额外的工作：

1. 首先，我们应该搜索任何剩余的 `NgModule` 并手动移除它们（通常，剩余的 `NgModule` 数量不会很多）。
2. 接下来，我们应该查看 `main.ts` 文件并使用独立路由（在自动迁移后，我们很可能已经移除了剩余的路由模块）。
3. 然后，我们应该检查我们的第三方依赖项，通常通过 `importProvidersFrom` 函数来提供这些依赖项。
如果你知道这些依赖项的较新版本有独立 API，考虑升级这些包并用新的独立 API 替换 `NgModule` 定义。
理想情况下，我们根本不希望使用 `importProvidersFrom` 函数。
4. 接下来，你可能会注意到 HTTP 客户端是通过以下行提供的：<br/>`provideHttpClient(withInterceptorsFromDi())`<br/>
我们已经见过这个 API，但还没有讨论过 `withInterceptorsFromDi()`。我们将在下一章讨论它。
5. 最后，如果我们有代码维护工具，如代码检查器、代码格式化器设置或其他工具，
我们应该运行它们以确保代码库保持纯净并修复任何出现的问题。

#### 2.5.4 处理循环依赖

使用 `schematic` 迁移时，我们可能会遇到一个难以预见的问题：
结果可能是我们的一些组件存在循环依赖。
当组件递归地相互渲染时，就会出现这种情况。
例如，在我们的 HRMS 应用程序中，我们可能有一个 `EmployeeCardComponent`，用于显示员工的简要信息；
一个 `ProjectCardComponent`，用于显示有多个员工参与的项目的简要信息；
在 `ProjectCardComponent` 中，我们可能会渲染一系列 `EmployeeCardComponent`，
每个 `EmployeeCardComponent` 会展开显示该员工正在参与的项目，这就形成了递归。

在 `NgModule` 中，这不是问题，因为它们可能都在同一个模块中声明，或者从一个共享模块中获取彼此的引用；
然而，在独立组件中，它们必须相互导入，这将导致以下错误：

```text
ReferenceError: Cannot access 'ProjectCardComponent' before initialization
```

虽然这个问题很棘手，但相当容易解决：我们只需要在相互导入组件时使用 `forwardRef` 函数。
例如，`ProjectCardComponent` 的元数据可能如下所示。

**清单 2.21 使用 `forwardRef` 避免循环依赖问题**

```typescript
@Component({
  standalone: true,
  imports: [
    forwardRef(() => EmployeeCardComponent)
  ]
})
export class ProjectCardComponent {}
```

根据 `Angular` 文档，`forwardRef` 函数旨在允许开发者访问尚未定义的引用；
换句话说，当 `ProjectCardComponent` 注册时，它会注意到在实际初始化时将使用某个引用。
另一个组件 `EmployeeCardComponent` 将像往常一样定义，无需 `forwardRef` ，这样就避免了循环依赖。

如我们所见，独立构建块为我们提供了新的功能，可简化我们的代码并减少样板代码。
这个新特性在各地的 `Angular` 开发者中非常受欢迎，越来越多的团队直接使用独立组件启动他们的新项目。
接下来，我们将更深入地探讨如何使用新的依赖注入方式来简化组件之间的共享功能。


### 2.6 读者练习

正如上一章所提到的，我鼓励你按照本书中的示例和步骤来构建这个 HRMS 项目。
虽然跟着做非常有益，但我也鼓励你自己编写一些代码，以便在实际环境中测试你在特定章节中获得的知识。
以下是在现有示例应用程序中可以完成的一些事情：

* 实现我们创建的功能中的其他组件（`EmployeeList`、`EmployeeDetails`、`CreateEmployee` 和 `EditEmployee`），
并为它们添加一些业务逻辑。
* 创建其他功能（工作和招聘功能，目前先创建空组件，因为我们将在后续章节中逐步为它们添加更多功能），并对独立组件使用延迟加载实践。
* 按照 2.5.3 节中描述的步骤，将迁移架构指令应用于现有项目。
迁移可用于将现有应用完全迁移到独立模式，或者进行实验以了解该架构指令的工作方式以及可能出现的问题；
在后一种情况下，可以随意丢弃自动化所做的任何更改。


## 小结

* `NgModule` 存在诸如样板代码过多、复杂度增加以及学习曲线陡峭等突出问题。
* 单组件 `Angular` 模块（SCAM）可以缓解基于 `NgModule` 的应用程序中的这些问题。
* 从 v14 版本开始，`Angular` 提供了独立的组件、指令和管道，在它们的元数据中标记为 `standalone: true`。
* 现在可以使用 `main.ts` 文件中的特殊 `bootstrapApplication` 函数，在完全没有 `NgModule` 的情况下构建应用程序。
* 独立组件、管道和指令可以通过其元数据中的 `imports` 数组导入其他独立构建块。
* 独立构建块可以通过将模块直接导入组件或通过 `importProvidersFrom` 函数与 `NgModule` 完全互操作。
* 现在独立组件可以直接延迟加载路由，而不必延迟加载整个模块。
* 现有应用程序可以通过手动方式或使用 `Angular` 团队提供的架构指令迁移到独立模式；SCAM 方法极大地简化了这个过程。


<br/><br/><br/><br/>
&gt;  [返回扉页](/books/modern-angular)
