---
title: VisualStyleRenderer造成GDI泄漏分析
author: Wenqing Zhang
date: 2008-01-10
category: CSharp
layout: post
tags: C# VisualStyleRenderer GDI
---

今天下午突然接到一个很奇怪的bug，发现在打开Theme的环境下，show OverflowTip时会导致每次1个GDI的泄漏，
拿到示例，跟了许久，居然没有发现任何线索，只能通过GDIUsage查到泄漏了相当多的GDI Region，
很郁闷，按照新的设计，应该不会有明显的GDI泄漏问题，即使有也可能只是开发时的“手误”，
只要打开GDI的Trace工具，肯定就可以发现，结果什么都没有，难道Trace工具也会错？


只好再查，发现问题只发生在Theme打开的环境，恰好这时看到代码中有一段关于Theme的处理，
于是抱着试试的心理，封了这段处理（代码3-11行），意外竟然发现不再有GDI Leak了，显然就是这里有鬼了。

```csharp
public override Region CreateRegion()
{
    if (this.UseSystemVisualStyle
        && Application.RenderWithVisualStyles
        && VisualStyleRenderer.IsElementDefined(VisualStyleElement.ToolTip.Standard.Normal))
    {
        VisualStyleRenderer render = new VisualStyleRenderer(VisualStyleElement.ToolTip.Standard.Normal);

        return render.GetBackgroundRegion(DefaultGraphics, this.Bounds);
    }
    else
        return new Region(this.Bounds);
}
```

但是这里很简单，基本上没有什么复杂的，构造了个 `VisualStyleRenderer` ，
调用 `GetBackgroundRegion` 获得一个 `Region` ，于是再次把这段代码打开，GDI又泄漏了，
嗯，既然外表看来没有问题，那只能去查查他的家底了。

打开Reflector，找到 `VisaulStyleRenderer.GetBackgroundRegion` ，
发现这里是通过uxTheme的API—— `GetThemeBackgroundRegion` 来获取一个GDI Region Object，
然后通过 `System.Drawing.Region.FromHrgn` 获得 `Region` 并返回；
或许你也发现了，这里的GDI Region Object没有释放！就是说就是这个函数导致了GDI泄漏。

```csharp
[SuppressUnmanagedCodeSecurity]
public Region GetBackgroundRegion(IDeviceContext dc, Rectangle bounds)
{
    if (dc == null)
    {
        throw new ArgumentNullException("dc");
    }
    if ((bounds.Width < 0) || (bounds.Height < 0))
    {
        return null;
    }
    IntPtr hrgn = IntPtr.Zero;
    using (WindowsGraphicsWrapper wrapper = new WindowsGraphicsWrapper(dc, TextFormatFlags.PreserveGraphicsTranslateTransform | TextFormatFlags.PreserveGraphicsClipping))
    {
        HandleRef hdc = new HandleRef(wrapper, wrapper.WindowsGraphics.DeviceContext.Hdc);
        this.lastHResult = SafeNativeMethods.GetThemeBackgroundRegion(new HandleRef(this, this.Handle), hdc, this.part, this.state, new NativeMethods.COMRECT(bounds), ref hrgn);
    }
    if (!(hrgn != IntPtr.Zero))
    {
        return null;
    }
    return Region.FromHrgn(hrgn);
}
```


分析到此，已经可以得出是MS在这里的bug了。
为了再次验证，我写了个简单的Demo，当每次点击Button后，就会发现系统资源里多一个GDI Object：

```csharp
public partial class Form1 : Form
{
    public Form1()
    {
        InitializeComponent();

        this.Text = "Current GDI Object Number: " + GetGuiResources(Process.GetCurrentProcess().Handle, 0).ToString();
    }

    private void button1_Click(object sender, EventArgs e)
    {
        if (Application.RenderWithVisualStyles
            && VisualStyleRenderer.IsElementDefined(VisualStyleElement.ToolTip.Standard.Normal))
        {
            VisualStyleRenderer render = new VisualStyleRenderer(VisualStyleElement.ToolTip.Standard.Normal);
            using (Graphics g = this.CreateGraphics())
            {
                using (Region region = render.GetBackgroundRegion(g, new Rectangle(0, 0, 100, 100)))
                {
                    g.FillRegion(Brushes.Red, region);
                }
            }
        }

        GC.Collect();
        GC.WaitForPendingFinalizers();

        this.Text = "Current GDI Object Number: " + GetGuiResources(Process.GetCurrentProcess().Handle, 0).ToString();
    }

    [DllImport("User32.dll")]
    private static extern int GetGuiResources(IntPtr hProcess, uint uiFlags);
}
```

下面是运行结果截图：
![GDI Leak](/images/visualstylerenderer/o_gdi leak.jpg)

OK，到此为止，确定这是MS VisualStyleRenderer导致GDI leak了。
面对bug，该怎么解决呢？既然这个接口会导致bug，我们只能提供一个相似功能的函数，
来作为短期替代方案，待MS修复bug之后再恢复现在code。

下面是解决方案：
```csharp
public override Region CreateRegion()
{
    if (this.UseSystemVisualStyle
        && Application.RenderWithVisualStyles
        && VisualStyleRenderer.IsElementDefined(VisualStyleElement.ToolTip.Standard.Normal))
    {
        VisualStyleRenderer render = new VisualStyleRenderer(VisualStyleElement.ToolTip.Standard.Normal);

        // MS VisualStyleRenderer.GetBackgroundRegion() would cause GDI leak, so I write a same logic
        // but clean the bug.
        // If MS have fixed this bug, please take care to rollback if possible.
        // ---------------------------------------------------------------
        //return render.GetBackgroundRegion(DefaultGraphics, this.Bounds);
        return SafeGetVisualStyleRendererBackgroundRegion(render, DefaultGraphics, this.Bounds)
    }
    else
        return new Region(this.Bounds);
}

private static Region SafeGetVisualStyleRendererBackgroundRegion(
    VisualStyleRenderer render, 
    Graphics g, 
    Rectangle bounds)
{
    IntPtr hrgn = IntPtr.Zero;
    IntPtr hdc = g.GetHdc();

    try
    {
        UnsafeNativeMethods.GetThemeBackgroundRegion(render.Handle, hdc,
            render.Part, render.State, new NativeMethods.RECT(bounds), ref hrgn);
        if (hrgn == IntPtr.Zero)
        {
            return null;
        }

        return Region.FromHrgn(hrgn);
    }
    finally
    {
        if (hdc != IntPtr.Zero)
        {
            g.ReleaseHdc(hdc);
        }
        if (hrgn != IntPtr.Zero)
        {
            UnsafeNativeMethods.DeleteObject(hrgn);
        }
    }
}
```
