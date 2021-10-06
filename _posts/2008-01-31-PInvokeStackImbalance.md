---
title: MDA提示PInvokeStackImbalance异常的分析和解决
author: Wenqing Zhang
date: 2008-01-31
category: CSharp
layout: post
tags: C# PInvoke MDA
---

### 问题描述
昨天接到一个bug，说在某一台机器上发现程序在使用IDE调试时会看到一个异常，并显示如下异常信息：
> A call to PInvoke function 'UnsafeNativeMethods.GetThemeBackgroundRegion' has unbalanced the stack.
> This is likely because the managed PInvoke signature does not match the unmanaged target signature.
> Check that the calling convention and parameters of the PInvoke signature match the target unmanaged signature.

### 重现和分析
bug来了，首当其冲的事情是重现，于是按照Tester给的描述，准备重现bug——打开系统Theme，
在VS环境下按F5启动，然后就应该看到异常；但是无论如何也没有办法重现，
没办法，只好去找Tester了，并针对bug提出了一些问题，发现仅仅在他的机器上能重现这个bug，
于是检查他的机器环境，发现没有安装.NET 2.0 SP1，而其他人都安装了，看来这里有点问题，
但是什么问题呢，除了MS没人知道，于是只好到MSDN里面搜索了一下这条异常消息，
发现是由于PInvoke声明和本地API调用栈不匹配，激活了托管调试助手（MDA），
依据这个解释那么应该是PInvoke的声明有问题了，于是找到PInvoke声明和对应的CPP声明，如下：
```csharp
[DllImport("uxtheme.dll", CharSet = CharSet.Auto)]
public static extern int GetThemeBackgroundRegion(IntPtr hTheme, IntPtr hdc,
     int iPartId, int iStateId, [In] RECT pRect, ref IntPtr pRegion);

[StructLayout(LayoutKind.Sequential)]
public struct RECT
{
    public int Left;
    public int Top;
    public int Right;
    public int Bottom;
}
```
对应Win32 API的C++接口声明如下：
```cpp
HRESULT GetThemeBackgroundRegion(HTHEME hTheme, HDC hdc,
    int iPartId, int iStateId, LPCRECT pRect, HRGN* pRegion);

typedef struct _RECT { 
    LONG left; 
    LONG top; 
    LONG right; 
    LONG bottom; 
} RECT, *PRECT, *LPCRECT;
```

这里貌似没有看到什么问题，但是在Tester的机器上确实发现问题了，没办法，
只好征求Tester在他的环境上调试了，按照bug重现步骤，果然看到了异常提示，
而且确实在这个PInvoke调用的时候MDA发出的，也就是说肯定声明有问题，但是在哪里呢？

因为这一段声明是从MS里面看到的，就是说和MS的是一样的，于是尝试使用MS的代码替换我的实现，
然后按照bug步骤，发现没有异常提示了，看来我们很接近问题的点了，于是一点一点的对比我们的PInvoke和MS的。
```csharp
[DllImport("uxtheme.dll", CharSet = CharSet.Auto)]
public static extern int GetThemeBackgroundRegion(HandleRef hTheme, HandleRef hdc, 
    int iPartId, int iStateId, [In] RECT pRect, ref IntPtr pRegion);

[StructLayout(LayoutKind.Sequential)]
public class RECT
{
    public int left;
    public int top;
    public int right;
    public int bottom;
}
```

细心的你应该发现问题了，这里MS对RECT的声明是**class**，而我使用的是**struct**！
显然当我把struct改成class后，就没有异常了，那么这里又是问什么呢？
这里我们的回到cpp的声明上，MS对API GetThemeBackgroundRegion的声明里的第五个参数是LPCRECT，
这里LPCRECT意味着这里传入的是RECT的指针，所以PInvoke对应这里声明必须也是指针或者等价类型
（例如void*、IntPtr、ref参数、引用类型），显然这里有了一个低级错误，
因为把RECT声明成struct，而没有使用ref或指针作为传入参数。
都知道struct是值类型，所以默认In参数是将值拷贝到调用函数的，
显然如果Marshal在这里处理会决定这个PInvoke行为，如果Marshal按照指针传入参数到本地API，
调用肯定没有问题，但是如果PInvoke不做任何处理，显然这里RECT的数据就会被截断成指针传入，
导致错误，而PInvoke在这里做了检测，发现了这个问题，通过MDA作出了提示，
当然实际的执行确实按照正确的指针传入，就是说，这里PInvoke没有失败，
因为Marshal在后台默默的做了很多工作。

好了，bug算是修了，但是为什么这个只在没有安装.NET 2.0 SP1的机器上才有呢，估计只有ms才能解释吧。

### 结论
**托管调试助手 (MDA)** 是调试辅助程序，它与公共语言运行库 (CLR) 结合工作以提供关于运行时状态的信息。
这些助手生成关于无法通过其他方式捕获的运行时事件的信息性消息。
可以使用 MDA 隔离在托管代码和非托管代码之间转换时发生的难以发现的应用程序 Bug。

### 补充资料

#### 启用和禁用 MDA
可使用注册表项、环境变量和应用程序配置设置，启用和禁用 MDA。要使用应用程序配置设置，就必须启用注册表项或环境变量。

##### 使用注册表项启用和禁用 MDA
可通过在 Windows 注册表中添加 HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\MDA 子项启用 MDA。

请将下面的示例复制到一个名为“MDAEnable.reg”的文本文件中，执行 MDAEnable.reg 文件，以在计算机上启用 MDA：
```
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework]
"MDA"="1"
```

若要禁用 MDA，请将下面的示例复制到一个名为“MDADisable.reg”的文本文件中，然后运行 MDADisable.reg 文件
```
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework]
"MDA"="0"
```

默认情况下，即使未添加该注册表项，有些 MDA 也会在运行附加到调试器的应用程序时启用。
PInvokeStackImbalance 和 InvalidApartmentStateChange 即属于此类助手。
可以按上面的描述运行 MDADisable.reg 文件来禁用这些助手。


##### 使用环境变量启用和禁用助手

还可以通过环境变量 COMPLUS_MDA 控制 MDA 激活。环境变量将重写注册表项。
该字符串是区分大小写的分号分隔的 MDA 名称的列表或其他特殊控制字符串。
在托管或非托管调试器下启动将默认启用一组 MDA。
这是通过在环境变量或注册表项的值前面隐式附加调试器下默认启用的分号分隔的 MDA 的列表来实现的。
特殊控制字符串如下：
- `0` -- 停用所有 MDA。
- `1` - 从应用程序名称.mda.config 读取 MDA 设置。
- `managedDebugger` -- 显式激活在调试器下启动托管可执行文件时隐式激活的所有 MDA。
- `unmanagedDebugger` -- 显式激活在调试器下启动非托管可执行文件时隐式激活的所有 MDA。

如果存在冲突的设置，则最近的设置重写先前的设置：
- `COMPLUS_MDA=0` 禁用所有 MDA，包括在调试器下隐式启用的那些 MDA。
- `COMPLUS_MDA=gcUnmanagedToManaged` 启用 gcUnmanagedToManaged 以及在调试器下隐式启用的任何 MDA。
- `COMPLUS_MDA =0;gcUnmanagedToManaged` 启用 gcUnmanagedToManaged，但是禁用那些将在调试器下隐式启用的 MDA。


##### 使用特定于应用程序的配置设置启用和禁用 MDA
可以在应用程序的 MDA 配置文件中单独地启用、禁用和配置某些助手。

若要使用用于配置 MDA 的应用程序配置文件，则必须设置 MDA 注册表项或 COMPLUS_MDA 环境变量。
应用程序配置文件通常与应用程序的可执行文件 (.exe) 位于同一目录。
该文件名采用应用程序名称.mda.config 的形式；例如，notepad.exe.mda.config。
在应用程序配置文件中启用的助手可能包含专门设计用于控制该助手的行为的属性或元素。
下面的示例演示如何启用和配置封送 MDA。

```xml
<mdaConfig>
  <assistants>
    <marshaling>
      <methodFilter>
        <match name="*"/>
      </methodFilter>
      <fieldFilter>
        <match name="*"/>
      </fieldFilter>
    </marshaling>
  </assistants>
</mdaConfig>
```

这些设置启用和配置 Marshaling MDA，对于应用程序中每个托管到非托管的转换，
该 MDA 发出描述正在被封送处理为非托管类型的托管类型的信息。Marshaling MDA 具有更大的灵活性，
能够分别筛选 <methodFilter> 和 <fieldFilter> 子元素中提供的方法和结构字段的名称。

有关特定于每个单独的 MDA 的设置的更多信息，请参见该 MDA 的文档。
