---
title:  "hash codes"
header:
  teaser: "/assets/images/teaser-2.jpg"
categories:
  - CSharp
tags:
  - hash
  - Object
  - HashCode
  - CSharp
excerpt: >
    本文介绍 C# 中 `GetHashCode` 方法，涉及哈希表原理、`Vector3D` 类的实现及性能优化等多方面内容。
---

## 前言
你可能已经知道在 .NET 世界中，`Object` 类中有一个名为 `GetHashCode()` 的方法，
并且每个 C# 类（直接或间接）都继承自 `Object` 类，该类仅公开了 3 个虚方法。
`ToString` 和 `Equals` 的用途相当明显，但是第三个方法 `GetHashCode` 呢？

当我在技术讨论中向一些社区成员询问 `GetHashCode` 时，我通常会得到以下这类回答：
1. 我不清楚（至少他们不知道自己不清楚什么）
2. 它为对象返回一个 ID
3. 它返回一个唯一值，我可以将其用作键
4. 它可用于测试两个对象是否相等

上述四种回答都不完全正确。

Hash code (哈希码) 不是一个 ID，它不会返回唯一值。
留意到该方法的签名，`GetHashCode` 返回一个 `Int32` 类型的值，
该类型 “仅有” 约 42 亿个可能的值，而潜在的不同对象数量可能是无限的，所以肯定会有一些对象的哈希码相同。

哈希码也不能用于测试两个对象是否相等。原因同上，具有相同哈希码的两个对象不一定相等。
不过，反过来是成立的：相等的两个对象应该具有相同的哈希码（或者至少应该如此，如果 `Equals` 和 `GetHashCode` 被正确实现的话）。

## 哈希表
那么，`GetHashCode` 有什么用呢？简单的答案是，它主要用于一个目的：
当对象用作哈希表中的键时，充当哈希函数，正如该函数的字面意思所示。
嗯，但是什么是哈希表呢？也许这个术语听起来不熟悉，但实际上你可能已经使用过一个了，
它就是 `Dictionary<TKey, TValue>` 类，该类表示一个键值对，其内部以某种方式使用了哈希表。
另一个是 `HashSet<TValue>`，顾名思义，这个集合内部使用了哈希表。

互联网上有很多优秀的文章，但我会尽力做一个简短的介绍，因为这不是一本教科书，所以描述可能不是完全精确。

哈希表是一种将值与键关联的数据结构。它可以根据给定的键查找值，并且平均时间复杂度为 O (1)；换句话说，
通过键查找条目的时间不依赖于哈希表中的条目数量。一般的原则是根据键的哈希码将条目放置在固定数量的 “存储桶” 中。

向哈希表添加条目看起来像如下伪代码：
```csharp
var hashCode = key.GetHashCode();
var bucketIndex = hashCode % bucketLength;
buckets[bucketIndex].Insert(value);
```
存储桶的数量是固定的，而每个存储桶可以有多个条目。不同实现处理这种情况的方式不同，但一种常见的方法是使用链表。

通过键获取（更好的说法是搜索）条目是按以下方式完成的：
```csharp
var hashCode = key.GetHashCode();
var bucketIndex = hashCode % bucketLength;
return buckets[bucketIndex].Find(key);
```

这就是哈希表如此高效的原因，如你所见，找到包含给定键的存储桶非常快，并且不依赖于条目的数量。
当你找到存储桶时，其中通常只有很少的条目，所以搜索它们也很快。

但是，要使其正常工作，首先必须满足一个重要条件：

> 条目必须均匀分布在存储桶中。

例如，如果你有 2000 个条目，而哈希表有 1000 个存储桶，最好的情况是将 2000 个条目均匀分布到每个存储桶中，
平均只有 2 个条目，但是如果所有条目最终都在同一个存储桶中，你会立即找到正确的存储桶，
但必须遍历所有条目才能找到正确的条目。这就完全没有意义，也着实太疯狂了。

## 实现 `GetHashCode`

假设有一个表示三维位置的类，名为 `Vector3D`，毫无疑问它包含三个属性，省略所有方法部分，它看起来应该是这样：
```csharp
public class Vector3D(double x, double y, double z)
{
    public double X { get; } = x;
    public double Y { get; } = y;
    public double Z { get; } = z;
}
```

根据上述讨论，很容易得出结论，有两种情况需要手动实现 `GetHashCode`：
* 该对象将作为哈希表中的键
* 该对象重写了 `Equals` 并最终需要保持一致性

因此，我们将有两种不同的策略来解决 `GetHashCode` 的问题，让我们继续下一部分。

### 组合哈希以适应 `Equals`

好吧，让我们揭示 `Vector3D` 的 `Equals` 方法是做什么的，
假设这是一个奇怪的像素世界，最小单位是一个像素（1x1x1 的方块内认为是一个地方），对于 x、y、z 使用 `Ceiling` 操作进行近似相等判断，以下是代码：

```csharp
static bool IsCeilingEqual(double value1, double value2) => 
    Math.Ceiling(value1).Equals(Math.Ceiling(value2));
public override bool Equals(object other) =>
    other is Vector3D p
    && IsCeilingEqual(p.X, X)
    && IsCeilingEqual(p.Y, Y)
    && IsCeilingEqual(p.Z, Z);
```
在这种情况下，表达式 `new Vector3D(1,1,1) == new Vector(.5,.5,.5)` 将为 `true`，
换句话说，它们两个的哈希码也应该相同。

```csharp
public override int GetHashCode() => ComputeHashCodeByCombine(this);
private static int ComputeHashCodeByCombine(Vector3D v)
    => HashCode.Combine(Math.Ceiling(v.X), Math.Ceiling(v.Y), Math.Ceiling(v.Z));
```

`HashCode.Combine` 使用一个带有 4 字节种子的静态变量作为起始值，
因此在应用程序重新启动后组合结果会不同，另外它的顺序是不可互换的！

还有另一种通过内部持有 `HashCode` 实例并使用其 `Add` 方法混合不同部分来组合哈希的变体，例如：
```csharp
public override int GetHashCode() => ComputeHashCodeByHashCodeAdd(this);
private static int ComputeHashCodeByHashCodeAdd(Vector3D v)
{
    var hash = new HashCode();
    foreach (var i in new[]{ v.X, v.Y, v.Z })
    {
        hash.Add(Math.Ceiling(i));
    }
    return hash.ToHashCode();
}
```

然而，这种变体看起来有点冗长（这也是为什么首先想到 `HashCode.Combine` 的原因），还有其他好的解决方案吗？

是的，由于引入了新的 `Tuple`，我们可以使用匿名元组快速计算哈希码，例如：

```csharp
public override int GetHashCode() => ComputeHashCodeByHashCodeAdd(this);
private static int ComputeHashCodeByTuple(Vector3D v)
    => (Math.Ceiling(v.X), Math.Ceiling(v.Y), Math.Ceiling(v.Z)).GetHashCode();
```

从字面上看，最后一种方法看起来相当清晰，但实际上，这三种方法都以某种方式基于 `HashCode.Combine`。

### 手动计算哈希以提高性能
如前所述，使用 `HashCode.Combine`（或其变体）易于阅读 / 维护，但会损失一些性能。让我们深入一点。

嗯，要计算哈希码，通常是将被视为相等的那些值组合起来，并添加一些质数来改善分布。

以下是一个示例，受到 **Jon Skeet** 在 **StackOverflow** 上的回答的启发：
```csharp
public override int GetHashCode() => ComputeHashCodeManual(this);
private static int ComputeHashCodeManual(Vector3D v)
{
    unchecked // 允许算术溢出，数字将“环绕”
    {
        int hashcode = 1031; // 从一个质数开始
        hashcode = hashcode * 8653 ^ Math.Ceiling(v.X).GetHashCode();
        hashcode = hashcode * 8653 ^ Math.Ceiling(v.Y).GetHashCode();
        hashcode = hashcode * 8653 ^ Math.Ceiling(v.Z).GetHashCode();
        return hashcode;
    }
}
```

我不太确定为什么使用质数很重要，但它确实重要；如果你比我更擅长数学，也许你可以试着解释一番！
当然，你可以使用与上述不同的值，只要确保它们是质数就可以。

几乎完成了，但稍等一下，我们可以重构这个逻辑以避免一行又一行的重复（如你所见的第 8、9、10 行）？

当然，我们可以使用 LINQ 的 `Aggregate` 方法对值进行聚合，它类似于从一个可枚举集合中减少每个项，例如：

```csharp
public override int GetHashCode() => ComputeHashCodeManual(this);
private static int ComputeHashCodeManual(Vector3D v)
{
    unchecked // 允许算术溢出，数字将“环绕”
    {
        return new[] { v.X, v.Y, v.Z }
           .Aggregate(1031, (c, i) => c * 8653 ^ Math.Ceiling(i).GetHashCode());
    }
}
```

## 结论
哈哈，让我用最后几句话来简化我的长篇大论：
- `GetHashCode` 需要符合自定义的 Equals。
- 如果不考虑性能要求，使用 `HashCode.Combine`（或其变体）。
- 采用一个质数，然后使用异或和乘法与质数相结合的方式以获得高性能。

