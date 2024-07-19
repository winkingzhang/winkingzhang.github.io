---
title:  "如何在 C# 中重复字符串"
header:
  teaser: "/assets/images/teaser.jpg"
categories:
  - CSharp
tags:
  - String
  - Span
  - Linq
  - CSharp
excerpt: >
    本文介绍了如何将字符串重复n次，包括使用StringBuilder, Linq, Array, Span 等多种方法。
---

当我用 C# 进行单元测试时，经常需要准备大量很长的字符串作为输入，这很费劲。
但是，在 JavaScript 或 Python 中做类似的事情很容易，例如这样:

```javascript
"test_string".repeat(5);
// output: test_stringtest_stringtest_stringtest_stringtest_string
```
```python
s = "test_string"
print(s * 5)
# output: test_stringtest_stringtest_stringtest_stringtest_string
print(s[5:] * 3)
# output: stringstringstring
```

不幸的是，C# 没有类似的函数 - 甚至 .NET 8 依然没有 - 所以我们只能自己写一个。
以下是一些如何在 C# 中重复字符串 n 次的几种方法。

>
> 文中所引用的源代码均托管在我的个人 GitHub 仓库里：<br/> 
> [https://github.com/winkingzhang/dotnet-repeat-string](https://github.com/winkingzhang/dotnet-repeat-string)
>

## 使用 `StringBuilder`

```csharp
public static string RepeatByStringBuilder(this string text, uint n)
{
    return new StringBuilder(text.Length * (int)n)
        .Insert(0, text, (int)n)
        .ToString();
}
```
为此，我们实例化一个新 `StringBuilder` 对象，初始容量设置为输入字符串的长度乘以重复的次数 `n`。
然后我们使用 `StringBuilder` 的方法 `Insert` 将字符串从开头位置（索引 0）开始插入 `n` 次。
最后，我们使用 `ToString` 方法返回字符串。

这种方法效率尚可，胜在简单直接。


## 使用 Linq

```csharp
public static string RepeatByLinq(this string text, uint n)
{
    return string.Concat(Enumerable.Repeat(text, (int)n));
}
```

这里我们使用 `System.Linq` 命名空间中的方法 `Enumerable.Repeat` 重复字符串 `n` 次，
然后使用该 `string.Concat` 方法将重复的字符串连接在一起。

这是重复字符串的一种非常直接的方法，但它并不是最有效的方法，因为它会创建一个 `IEnumerable<string>` 对象。

## 使用 `Array`

```csharp
public static string RepeatByArray(this string text, uint n)
{
    var arr = new char[text.Length * (int)n];
    for (var i = 0; i < n; i++)
    {
        text.CopyTo(0, arr, i * text.Length, text.Length);
    }
    return new string(arr);
}
```
现在事情变得有趣了，这里要使用 `for` 循环来重复字符串 `n` 次。
首先使用 `char[]` 数组存储重复的字符串，然后使用 `CopyTo` 方法将字符串复制到数组的指定位置处。

这是一种相对复杂的字符串重复的方法，但更有效率，尤其是当 `n < 100` 时。

但是，等等，既然它采用了更多的内存空间来加速性能，那以此类推我们可以通过 `Array.Fill<T>` 方法来改进它

```csharp
public static string RepeatByArrayFill(this string text, uint n)
{
    var arr = new string[(int)n];
    Array.Fill(arr, text);
    return string.Concat(arr);
}
```

## 使用 `Span`

```csharp
// CSharp 7.2 above
public static string RepeatBySpan(this string text, uint n)
{
    var textAsSpan = text.AsSpan();
    var span = new Span<char>(new char[textAsSpan.Length * (int)n]);
    for (var i = 0; i < n; i++)
    {
        textAsSpan.CopyTo(span.Slice((int)i * textAsSpan.Length, textAsSpan.Length));
    }
    return span.ToString();
}
```
`Span` 是 C# 7.2 中引入的一项新功能，允许将任意内存的连续区域表示为一等对象。
这在处理大量数据时非常有用，因为它可以更有效地使用内存以提高性能。

这里使用 `Span<char>` 来抽象字符串在内存中的存储形式，本质上是在处理堆栈上的连续内存，而不是字符串。这就是这种方法更高效的原因。
我们将原始字符串的内存片段复制到这个新的 `Span<char>` 的片段上。事实证明，这种方法非常高效，因为它不会在堆上分配内存。

## 性能比较

到此为止，我们有几种实现方式，让我们做一个基准测试，期望能找出最快的一种实现（目前测试在 Apple M2 MBP 和联想 Legion Y9000P 上进行）。

```
BenchmarkDotNet v0.13.12, macOS Sonoma 14.5 (23F79) [Darwin 23.5.0]
Apple M2 Pro, 1 CPU, 12 logical and 12 physical cores
.NET SDK 8.0.302
  [Host]     : .NET 8.0.6 (8.0.624.26715), Arm64 RyuJIT AdvSIMD
  DefaultJob : .NET 8.0.6 (8.0.624.26715), Arm64 RyuJIT AdvSIMD
| Method                  | Source              | N    | Mean           | Error         | StdDev        | Gen0     | Gen1     | Gen2     | Allocated |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 1    |      22.586 ns |     0.4437 ns |     0.5611 ns |   0.0181 |        - |        - |     152 B |
| RepeatByLinq            | Hello World!        | 1    |       8.189 ns |     0.0644 ns |     0.0571 ns |   0.0048 |        - |        - |      40 B |
| RepeatByArray           | Hello World!        | 1    |      10.847 ns |     0.0673 ns |     0.0562 ns |   0.0124 |        - |        - |     104 B |
| RepeatByArrayFill       | Hello World!        | 1    |       9.420 ns |     0.0376 ns |     0.0294 ns |   0.0038 |        - |        - |      32 B |
| * RepeatBySpan          | Hello World!        | 1    |      10.844 ns |     0.0867 ns |     0.0677 ns |   0.0124 |        - |        - |     104 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 10   |      61.587 ns |     0.8528 ns |     0.7121 ns |   0.0745 |        - |        - |     624 B |
| RepeatByLinq            | Hello World!        | 10   |      49.517 ns |     0.4064 ns |     0.3393 ns |   0.0391 |        - |        - |     328 B |
| RepeatByArray           | Hello World!        | 10   |      46.391 ns |     0.9445 ns |     0.8835 ns |   0.0688 |        - |        - |     576 B |
| RepeatByArrayFill       | Hello World!        | 10   |      58.089 ns |     0.2114 ns |     0.1650 ns |   0.0468 |        - |        - |     392 B |
| * RepeatBySpan          | Hello World!        | 10   |      45.400 ns |     0.8819 ns |     0.7818 ns |   0.0688 |        - |        - |     576 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 100  |     481.105 ns |     3.8141 ns |     3.1850 ns |   0.6332 |   0.0010 |        - |    5296 B |
| RepeatByLinq            | Hello World!        | 100  |     485.196 ns |     7.6006 ns |     7.1096 ns |   0.3176 |        - |        - |    2664 B |
| RepeatByArray           | Hello World!        | 100  |     389.557 ns |     7.6172 ns |     6.7525 ns |   0.6270 |        - |        - |    5248 B |
| RepeatByArrayFill       | Hello World!        | 100  |     516.305 ns |     5.8944 ns |     5.2253 ns |   0.4120 |        - |        - |    3448 B |
| * RepeatBySpan          | Hello World!        | 100  |     360.148 ns |     3.0571 ns |     2.7100 ns |   0.6270 |        - |        - |    5248 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 1000 |   4,021.073 ns |    59.2613 ns |    49.4859 ns |   6.2103 |        - |        - |   52096 B |
| RepeatByLinq            | Hello World!        | 1000 |   4,314.573 ns |    65.3837 ns |    54.5984 ns |   3.1052 |        - |        - |   26064 B |
| RepeatByArray           | Hello World!        | 1000 |   3,375.894 ns |    14.0973 ns |    11.7719 ns |   6.1913 |        - |        - |   52048 B |
| RepeatByArrayFill       | Hello World!        | 1000 |   4,890.302 ns |    96.4937 ns |    94.7697 ns |   4.0588 |        - |        - |   34048 B |
| * RepeatBySpan          | Hello World!        | 1000 |   3,242.421 ns |    25.4595 ns |    22.5692 ns |   6.1913 |        - |        - |   52048 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 1    |     171.625 ns |     3.0466 ns |     2.7007 ns |   0.4418 |   0.0019 |        - |    3696 B |
| RepeatByLinq            | The (...)og.  [900] | 1    |       8.150 ns |     0.0921 ns |     0.0862 ns |   0.0048 |        - |        - |      40 B |
| RepeatByArray           | The (...)og.  [900] | 1    |     158.566 ns |     0.9380 ns |     0.7323 ns |   0.4361 |        - |        - |    3648 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 1    |       9.446 ns |     0.1320 ns |     0.1102 ns |   0.0038 |        - |        - |      32 B |
| RepeatBySpan            | The (...)og.  [900] | 1    |     158.091 ns |     1.5203 ns |     1.2696 ns |   0.4361 |        - |        - |    3648 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 10   |   1,343.146 ns |    26.3687 ns |    23.3752 ns |   4.3087 |   0.2689 |        - |   36096 B |
| RepeatByLinq            | The (...)og.  [900] | 10   |   1,553.162 ns |    30.1964 ns |    44.2615 ns |   2.1534 |        - |        - |   18064 B |
| RepeatByArray           | The (...)og.  [900] | 10   |   1,344.261 ns |     8.6756 ns |     6.7733 ns |   4.3011 |        - |        - |   36048 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 10   |     773.897 ns |    11.5882 ns |    10.2726 ns |   2.1591 |        - |        - |   18128 B |
| RepeatBySpan            | The (...)og.  [900] | 10   |   1,377.670 ns |    13.2442 ns |    12.3886 ns |   4.3011 |        - |        - |   36048 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 100  |  40,207.800 ns |   557.9090 ns |   465.8792 ns | 333.5571 | 333.5571 |  95.2148 |  360159 B |
| RepeatByLinq            | The (...)og.  [900] | 100  |  35,458.880 ns |   324.1271 ns |   253.0571 ns | 143.3105 | 143.3105 |  53.5889 |  198812 B |
| RepeatByArray           | The (...)og.  [900] | 100  |  41,028.174 ns |   660.0815 ns |   834.7917 ns | 333.2520 | 333.2520 |  95.2148 |  360111 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 100  |  29,060.898 ns |   475.4133 ns |   421.4415 ns | 260.6812 | 260.6812 |  43.4570 |  180877 B |
| RepeatBySpan            | The (...)og.  [900] | 100  |  40,898.484 ns |   773.1716 ns |   685.3965 ns | 333.7402 | 333.7402 |  95.0928 |  360111 B |
|------------------------ |-------------------- |----- |---------------:|--------------:|--------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 1000 | 308,376.300 ns | 3,170.6771 ns | 2,647.6590 ns | 669.4336 | 667.9688 | 665.5273 | 3603238 B |
| RepeatByLinq            | The (...)og.  [900] | 1000 | 267,982.010 ns | 5,272.7026 ns | 9,372.2235 ns | 511.2305 | 504.8828 | 500.4883 | 2861879 B |
| RepeatByArray           | The (...)og.  [900] | 1000 | 316,297.470 ns | 2,105.5426 ns | 1,643.8692 ns | 668.4570 | 667.9688 | 665.5273 | 3600480 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 1000 | 202,907.132 ns | 3,826.7030 ns | 5,238.0342 ns | 572.5098 | 572.5098 | 214.3555 | 1808211 B |
| RepeatBySpan            | The (...)og.  [900] | 1000 | 317,738.614 ns | 5,167.9774 ns | 4,834.1292 ns | 668.4570 | 667.9688 | 665.5273 | 3600480 B |

```
```
BenchmarkDotNet v0.13.12, Windows 11 (10.0.22631.3880/23H2/2023Update/SunValley3)
12th Gen Intel Core i7-12700H, 1 CPU, 20 logical and 14 physical cores
.NET SDK 8.0.303
  [Host]     : .NET 8.0.7 (8.0.724.31311), X64 RyuJIT AVX2
  DefaultJob : .NET 8.0.7 (8.0.724.31311), X64 RyuJIT AVX2
| Method                  | Source              | N    | Mean             | Error          | StdDev         | Gen0     | Gen1     | Gen2     | Allocated |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 1    |        19.113 ns |      0.4035 ns |      0.4485 ns |   0.0121 |        - |        - |     152 B |
| RepeatByLinq            | Hello World!        | 1    |         6.164 ns |      0.1237 ns |      0.1157 ns |   0.0032 |        - |        - |      40 B |
| RepeatByArray           | Hello World!        | 1    |        10.534 ns |      0.1746 ns |      0.1633 ns |   0.0083 |        - |        - |     104 B |
| RepeatByArrayFill       | Hello World!        | 1    |        12.770 ns |      0.1091 ns |      0.1021 ns |   0.0025 |        - |        - |      32 B |
| * RepeatBySpan          | Hello World!        | 1    |        10.460 ns |      0.0960 ns |      0.0898 ns |   0.0083 |        - |        - |     104 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 10   |        59.964 ns |      0.3641 ns |      0.3228 ns |   0.0497 |        - |        - |     624 B |
| RepeatByLinq            | Hello World!        | 10   |        46.269 ns |      0.3449 ns |      0.3057 ns |   0.0261 |        - |        - |     328 B |
| RepeatByArray           | Hello World!        | 10   |        45.578 ns |      0.2459 ns |      0.1920 ns |   0.0459 |        - |        - |     576 B |
| RepeatByArrayFill       | Hello World!        | 10   |        57.397 ns |      0.4272 ns |      0.3335 ns |   0.0312 |        - |        - |     392 B |
| * RepeatBySpan          | Hello World!        | 10   |        44.148 ns |      0.6019 ns |      0.5335 ns |   0.0459 |        - |        - |     576 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 100  |       426.229 ns |      7.6544 ns |     11.2197 ns |   0.4215 |   0.0038 |        - |    5296 B |
| RepeatByLinq            | Hello World!        | 100  |       432.568 ns |      1.1930 ns |      1.0576 ns |   0.2122 |        - |        - |    2664 B |
| RepeatByArray           | Hello World!        | 100  |       373.965 ns |      6.4629 ns |      5.3968 ns |   0.4182 |        - |        - |    5248 B |
| RepeatByArrayFill       | Hello World!        | 100  |       479.134 ns |      3.8107 ns |      2.9751 ns |   0.2747 |        - |        - |    3448 B |
| * RepeatBySpan          | Hello World!        | 100  |       345.615 ns |      4.2018 ns |      3.9304 ns |   0.4182 |   0.0038 |        - |    5248 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | Hello World!        | 1000 |     3,869.509 ns |     70.9931 ns |     66.4070 ns |   4.1428 |        - |        - |   52096 B |
| RepeatByLinq            | Hello World!        | 1000 |     4,370.279 ns |     86.2628 ns |     99.3403 ns |   2.0676 |        - |        - |   26064 B |
| RepeatByArray           | Hello World!        | 1000 |     3,447.800 ns |     67.0250 ns |     62.6952 ns |   4.1313 |   0.3738 |        - |   52048 B |
| RepeatByArrayFill       | Hello World!        | 1000 |     4,553.790 ns |     32.4700 ns |     30.3724 ns |   2.7084 |        - |        - |   34048 B |
| * RepeatBySpan          | Hello World!        | 1000 |     3,322.508 ns |     22.3966 ns |     20.9498 ns |   4.1313 |   0.3738 |        - |   52048 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 1    |       195.531 ns |      3.6825 ns |      3.0751 ns |   0.2944 |   0.0010 |        - |    3696 B |
| RepeatByLinq            | The (...)og.  [900] | 1    |         6.106 ns |      0.1164 ns |      0.1089 ns |   0.0032 |        - |        - |      40 B |
| RepeatByArray           | The (...)og.  [900] | 1    |       184.165 ns |      1.0751 ns |      1.0057 ns |   0.2906 |   0.0010 |        - |    3648 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 1    |        12.569 ns |      0.1770 ns |      0.1478 ns |   0.0025 |        - |        - |      32 B |
| RepeatBySpan            | The (...)og.  [900] | 1    |       187.762 ns |      2.3320 ns |      2.1814 ns |   0.2906 |   0.0010 |        - |    3648 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 10   |     1,364.000 ns |     13.3304 ns |     12.4693 ns |   2.8725 |        - |        - |   36096 B |
| RepeatByLinq            | The (...)og.  [900] | 10   |     1,567.080 ns |     19.6918 ns |     15.3741 ns |   1.4381 |        - |        - |   18064 B |
| RepeatByArray           | The (...)og.  [900] | 10   |     1,405.505 ns |     13.6193 ns |     12.7395 ns |   2.8648 |   0.1907 |        - |   36048 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 10   |       748.462 ns |     11.4568 ns |      9.5670 ns |   1.4400 |        - |        - |   18128 B |
| RepeatBySpan            | The (...)og.  [900] | 10   |     1,418.811 ns |     12.4327 ns |     11.0213 ns |   2.8648 |   0.1907 |        - |   36048 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 100  |   111,793.479 ns |  2,168.6215 ns |  2,581.5896 ns | 111.0840 | 111.0840 | 111.0840 |  360133 B |
| RepeatByLinq            | The (...)og.  [900] | 100  |    64,249.943 ns |  1,273.1289 ns |  1,466.1371 ns |  55.5420 |  55.5420 |  55.5420 |  180083 B |
| RepeatByArray           | The (...)og.  [900] | 100  |   111,735.743 ns |  1,928.2633 ns |  1,803.6987 ns | 111.0840 | 111.0840 | 111.0840 |  360085 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 100  |    55,911.211 ns |  1,080.2581 ns |  1,244.0268 ns |  55.5420 |  55.5420 |  55.5420 |  180867 B |
| RepeatBySpan            | The (...)og.  [900] | 100  |   113,400.479 ns |  1,778.2817 ns |  1,576.4004 ns | 111.0840 | 111.0840 | 111.0840 |  360085 B |
|------------------------ |-------------------- |----- |-----------------:|---------------:|---------------:|---------:|---------:|---------:|----------:|
| RepeatByStringBuilder   | The (...)og.  [900] | 1000 | 1,016,601.237 ns |  8,227.9402 ns |  7,696.4204 ns | 998.0469 | 998.0469 | 998.0469 | 3600432 B |
| RepeatByLinq            | The (...)og.  [900] | 1000 |   380,526.263 ns |  5,745.3800 ns |  5,374.2321 ns | 276.3672 | 276.3672 | 276.3672 | 1800198 B |
| RepeatByArray           | The (...)og.  [900] | 1000 | 1,027,307.370 ns | 11,481.7296 ns | 10,740.0170 ns | 998.0469 | 998.0469 | 998.0469 | 3600384 B |
| * RepeatByArrayFill     | The (...)og.  [900] | 1000 |   557,133.607 ns |  4,842.4693 ns |  4,529.6488 ns | 499.0234 | 499.0234 | 499.0234 | 1808216 B |
| RepeatBySpan            | The (...)og.  [900] | 1000 | 1,037,758.529 ns | 15,399.8181 ns | 14,404.9994 ns | 998.0469 | 998.0469 | 998.0469 | 3600384 B |

```

### 总结

根据上面的基准报告，很容易总结 
- `RepeatBySpan` 对于短字符串重复是效率相对最高的。
- `RepeatByArrayFill` 对于长字符串重复是效率相对最高的。

