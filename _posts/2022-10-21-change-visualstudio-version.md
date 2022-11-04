---
title:  "Visual Studio各版本解决方案和工程文件之间的转换"
header:
  teaser: "/assets/images/teaser-1.jpg"
categories:
  - CSharp
tags:
  - VS2010
  - VisualStudio
  - CSharp
excerpt: >
    Visual Studio 差不多每两年就有一个大版本发布，因此解决方案和工程文件版本也比较多，
    而且低版本是无法直接打开高版本的解决方案和工程文件的，但可以通过手动修改来缓解这些问题。
---

## 问题描述

最近有一个遗留项目需要重新编译，浏览了一下代码，发现主体代码是 VS2010 的。
从解决方案文件来看，部分已经识别是 VS2015 的；
从工程文件来看，大部分 C# 工程是 VS 2010 的，但部分 C++ 代码已经被切换到了 VS2015。

因此强制使用 VS2010 打开并编译看到如下错误：

```
错误	1	error MSB8008: 指定的平台工具集(v140)未安装或无效。请确保选择受支持的 PlatformToolset 值。
	C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0\Platforms\Win32\Microsoft.Cpp.Win32.Targets
```

当然，这里安装 VS2015 之后编译肯定是可以的，因为 `MSBUILD` 能够自动适配系统安装的编译器。
但是，为了兼容考虑必须选择 VS2010 做编译，因此需要想办法降级解决方案和工程文件的版本。

## 解决方案文件版本

解决方案的格式比较简单，它就是一个纯文本，在前几行描述格式版本和支持的 Visual Studio 版本，
整理了一下，如下表格：

| Visual Studio 版本     | 解决方案格式（头部）                                                                                                                                                                              |
|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Visual Studio 2022` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio Version 17 <br/> VisualStudioVersion = 17.2.32505.173 <br/> MinimumVisualStudioVersion = 10.0.40219.1 |
| `Visual Studio 2019` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio Version 16 <br/> VisualStudioVersion = 16.0.28701.123 <br/> MinimumVisualStudioVersion = 10.0.40219.1 |
| `Visual Studio 2017` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio 15 <br/> VisualStudioVersion = 15.0.27703.2035 <br/> MinimumVisualStudioVersion = 10.0.40219.1        |
| `Visual Studio 2015` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio 14 <br/> VisualStudioVersion = 14.0.25123.0 <br/> MinimumVisualStudioVersion = 10.0.40219.1           |
| `Visual Studio 2013` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio 2013 <br/> VisualStudioVersion = 12.0.30501.0 <br/> MinimumVisualStudioVersion = 10.0.40219.1         |
| `Visual Studio 2012` | Microsoft Visual Studio Solution File, Format Version 12.00 <br/> # Visual Studio 2012                                                                                                  |
| `Visual Studio 2010` | Microsoft Visual Studio Solution File, Format Version 11.00 <br/> # Visual Studio 2010                                                                                                  |

因此，想要 VS2010 识别解决方案，只需对前几行做如下修改：
* 将 Format Version 12.00 改为 **11.00**；
* 将 # Visual Studio 14 改为 **2010**；
* 删除 VisualStudioVersion = 14.0.25123.0 或者将值改为 **10**。

## 工程文件版本

工程文件和解决方案文件一样，都是纯文本文件，但稍微区别的是工程文件采用比较严谨的 XML 作为主体，方便 `MSBUILD` 来统一处理，
因此可以使用任意文本编辑器打开， 前两行类似如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="14.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
```

这里 `ToolsVersion` 就是关键标记，整理了一下对于 Visual Studio 对于工程文件的定义如下表：

|  Visual Studio 版本    | 工程文件 ToolsVersion 定义 |
|----------------------|----------------------|
| `Visual Studio 2022` | ToolsVersion="17.0"  |
| `Visual Studio 2019` | ToolsVersion="16.0"  |
| `Visual Studio 2017` | ToolsVersion="15.0"  |
| `Visual Studio 2015` | ToolsVersion="14.0"  |
| `Visual Studio 2013` | ToolsVersion="12.0"  |
| `Visual Studio 2012` | ToolsVersion="4.0"   |
| `Visual Studio 2010` | ToolsVersion="4.0"   |

因此，想要 VS2010 识别工程文件，需要替换 ToolsVersion 这个特性
* 将 ToolsVersion="14.0" 改为 **ToolsVersion="4.0"**；


#### VC 项目文件

对于 C++ 项目来说还有一个额外的配置，就是 `PlatformToolset`。
Platform Toolset 允许项目以不同版本的 Visual C++ 库和编译器为目标。

PlatformToolset 与 Visual Studio 版本的对应关系如下：

|  Visual Studio 版本    | PlatformToolset 版本 |
|----------------------|--------------------|
| `Visual Studio 2022` | v143               |
| `Visual Studio 2019` | v142               |
| `Visual Studio 2017` | v141               |
| `Visual Studio 2015` | v140               |
| `Visual Studio 2013` | v120               |
| `Visual Studio 2012` | v110               |
| `Visual Studio 2010` | v100               |

因此，想要 VS2010 识别工程文件，需要替换 ToolsVersion 这个特性
* 将 `<PlatformToolset>v140</PlatformToolset>` 替换成 `<PlatformToolset>v100</PlatformToolset>`，一般来说都是4个，取决于工程中的编译平台的定义。

## 结论

虽然上面的方法可以降低源代码对 Visual Studio 版本的要求，但是并不能保证代码一定能够从高版本平滑切换到低版本，
实际代码中还可能有很多高版本支持的功能需要调整，因此还需要大量的编译和测试工作。
