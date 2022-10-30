---
title:  "Partial Content(206) 那些事"
header:
  teaser: "/assets/images/teaser-2.jpg"
categories:
  - web
tags:
  - http
  - PartialContent
  - CSharp
excerpt: >
  这是来自客户的一个真实而奇怪的问题，突然某一天他们的 Google Chome 和 Edge 访问网站时，
  许多图片都离奇消失了，但是又不是完全消失；支持人员一番尝试发现……
---

## 前言

说起来这是来自客户的一个真实而奇怪的问题，突然某一天他们的 Google Chome 和 Edge 访问网站时，
许多图片都离奇消失了，但是又不是完全消失；支持人员一番尝试发现：

- 换个非 Chome 内核的浏览器就好了，比如 Firefox
- 正常工作的图片都是 sprite 图的第一张
- 网络请求可以看到正常的记录，但是 Content-Length 是 0，并且这里的 Http StatusCode 是 `PartialContent (206)`

就这样开发团队发现只要把这里的 StatusCode 改成 OK (200) 就好了，一番操作上线，总算解决了线上问题。

但是，我的脑子里不自然的一直盘亘两个问题：

- 为什么 PartialCode (206) 不能工作，换一个 OK (200) 就能让图片正常工作了呢？
- 什么是 PartialCode (206) ，它到底该如何正确使用呢？

## 分析问题

上一小节简单描述了现象，接下来贴一下示例的问题代码吧，以便后续分析。
当然，也可以看我的 [github repo - partial-content-reveal](https://github.com/winkingzhang/partial-content-reveal)
```csharp
[HttpGet("sprite-images", Name = "GetPartialContentSpriteImages")]
public async Task GetSpriteImages([FromQuery] int? index, CancellationToken token = default)
{
    Response.StatusCode = StatusCodes.Status206PartialContent;

    var imageStream = Assets.GetImageStream();
    var (offset, length) = Assets.GetImageOffsetAndLength(index);

    var imageBytes = ArrayPool<byte>.Shared.Rent(length);
    imageStream.Seek(offset, SeekOrigin.Begin);
    var readLength = await imageStream.ReadAsync(imageBytes.AsMemory(0, length), token);

    Response.ContentType = MediaTypeNames.Application.Octet; // application/octec, default binary
    Response.ContentLength = readLength;

    await StreamCopyOperation.CopyToAsync(new MemoryStream(imageBytes), Response.Body, readLength,
        bufferSize: 64 * 1024, token);

    _logger.LogInformation("Response sprite image by {Index}, size is {SizeInBytes} kB",
        index,
        readLength / 1000);

    await Response.CompleteAsync();
}
```

这段代码并没有特别的地方，总结起来就是这么几件事情：
1. 将 Http StatusCode 设置成 PartialContent (206)
2. 从一个 blob 里找到对应的图片的部分（同时记录位置，长度等等信息）
3. 将 Http ContentType 设置成 `application/octec` （其实这也是默认的，标记这是一个 binary 流）
4. 将上面读到的图片的 binary 数据流写入到 Response Body里面

前文也提到过，修复的方案就是把 Http StatusCode 改成 OK (200) 就好了，看来是浏览器端对 206 response
做了特殊处理，但究竟是做了什么呢？让我们看一下 Http StatusCode RFC-9110 的一段话，原文是这样的：

> A client MUST inspect a 206 response's Content-Type and Content-Range field(s)
> to determine what parts are enclosed and whether additional requests are needed.

简单翻译一下，就是说：

> 客户端必须检查 206 响应的 Content-Type 和 Content-Range 字段来决定封装的部分内容是什么，以及是否需要额外的请求。

也就是说，一旦返回 206，那么浏览器就应该检查 Content-Type，并以此来决定自己的后续行为，
这里代码中指定的 Content-Type 是 `application/octec`， 这表示发送了一段未知格式的二进制流，
显然，不同浏览器采用了不同的策略，Firefox 猜出来这是图片并直接显示出来，Chrome 当然也是猜出来是图片的，
但上面代码没有指定 Content-Range，因此中断了后续的行为，导致除了第一张图之外的所有图片不显示。

到此你可能也看出来，另一个可能的修复方案了，就是讲 Content-Type 指定为 `image/jpeg`，
究其原因，因为此时根据 Content-Type 已知是图片，无论什么 Http StatusCode 浏览器都能够显示出图片，甚至可以是一小部分图片。

这么解释似乎已经能够为客户问题感官定论了，但终究还有一点点缺憾，
PartialContent (206) 到底是什么，用来解决什么问题，这些还需要一点篇幅慢慢展开。

## 探究 206

#### 定义

首先，摘选一段 RFC-9110 / RFC-7233 的片段来看看标准是如何定义 206 的

> The 206 (Partial Content) status code indicates that the server is successfully fulfilling a range request
> for the target resource by transferring one or more parts of selected representation.

从这个定义，我理解是这样的

> 206 Partial Content 表示服务器已正确返回请求，资源内容可以是不完整但必须符合请求中 Range 条件。 

Ok，显然这个设计非常高效而且可控的解决了早期 HTTP 客户端下载中断、恢复，甚至分片同时下载的需求。 

### 深入 206

#### 流程介绍

首先，让我们捋一下涉及 206 的下载请求的整个流程，这里我简单按顺序列举下面这几步：
1. 客户端向服务器发送下载资源的请求。
2. 服务器向客户端返回，通过响应头 Accept-Ranges 来告知这个资源可以接受分片。
3. 客户端再次向服务器发送请求，附带 Range 请求头来告知期望下载的片段。
4. 服务器按以下情况返回数据：
    - 如果 Range 合法，服务器返回 206 Partial Content 以及分片的内容，并在 Content-Range 里表明当前内容的分片信息。
    - 如果 Range 不合法 （例如：超过范围），服务器返回 416 Request Range Not Satisfiable，并在 Content-Range 里表明当前内容的可用的分片信息。

接下来，看一下上文提到的几个关键请求头和响应头

###### # Accept-Ranges: bytes

这是服务器向客户端返回的响应头，表示所请求的资源可以分片下载。它的值是后续 Range 请求的单位，通常是 `bytes`。

###### # Range: (start)-(end)

这个请求头是客户端告诉服务器期望下载的片段范围，注意这里的 `start` 和 `end` 都是以 0 为起始并包含在内的，它们两个可省略其中任意一个，分别代表不同的意思：
- 如果省略 `end`，那么服务器将返回从 `start` 位置开始的后续所有数据。
- 如果省略 `start`， 那么服务器将返回从起始位置直到 `end` 位置的所有数据，换言之，此时 `end` 和数据的总大小是一致的。

###### # Content-Range: bytes (start)-(end)/(total)

这个响应头通常随着 206 一起返回，这里的 `start` 和 `end` 代表数据在当前资源的起始和结束位置。
和 `Range` 头一样，它们都是以 0 为起始的。`total` 表示的是整个资源可分片的有效大小范围。

###### # Content-Range: */(total)

和上面一样但格式不同，通常随着 416 一起返回。`total` 表示的是整个资源可分片的有效大小范围。

###### # Content-Type: multipart/byteranges; boundary=SEPARATES

当上文流程第3步的 Range 头包含多个合法范围的时候，第4步的返回时会从服务器返回这个响应头，
用来表示响应是多个片段，使用字符串 `SEPARATES` 作为分隔，每一段又有自己的 `Content-Length` 和 `Content-Type`。

#### 动手逐步实现

为了探究实现细节，这里定义了一个特殊的 Web API Controller Action。
因为会手动处理 Response Stream，这里返回值 Task，
站在总流程的角度，刚开始只需要考虑通用 Content-Type 是 `image/jpg`， 并在结束之前 Flush Response 流就可以了。

代码看起来就像这样：

```csharp
[HttpGet("images", Name = "GetPartialContentImages")]
public async Task GetImages(CancellationToken token = default)
{
    // 让我们从简单的图片格式开始，实际中这里需要根据需求决定
    Response.ContentType = MediaTypeNames.Image.Jpeg;

    // TODO: 这里加入处理逻辑

    // 这里结束 Response
    await Response.CompleteAsync();
}
```

接下来要考虑的是处理请求头里的 Range，注意这里 Range 内部可能会有多段的情况要处理，逻辑上这里会是如下示意的三条分支：

- Range 请求头存在且只有一个 range => 将返回一个片段数据
- Range 请求头存在且有多个 range => 将返回多个片段数据
- Range 请求头不存在，或者存在但没有任何 range => 将返回完整数据

示例代码如下：
```csharp
var rangeHeader = Request.GetTypedHeaders().Range;
switch (rangeHeader?.Ranges.Count)
{
    case 1:
        // single part
        break;
    case > 1:
        // multiple parts
        break;
    default:
        // no range in header
        break;
}
```

###### 单一部分内容响应

让我们从简单的单一部分内容响应开始，此时已知 Range 请求头只有一个 range，
接下来要验证这个 range 的合法性，简单的判断就是不能小于 0， 不能大于资源的长度：

```csharp
// 这里模拟读取资源的整个数据流
var resourceStream = Assets.Img10_Jpg!;

var firstRange = rangeHeader.Ranges.First();
var rangeStart = firstRange.From ?? 0;
var rangeEnd = firstRange.To ?? resourceStream.Length;
var length = (int)(rangeEnd - rangeStart);

if (rangeStart < 0 || rangeEnd > resourceStream.Length)
{
    Response.StatusCode = StatusCodes.Status416RangeNotSatisfiable;
    Response.Headers.ContentRange = $"*/{resourceStream.Length}";
    
    _logger.LogWarning("The request {Range} is invalid, it should be {From} to {To}",
        firstRange,
        0,
        resourceStream.Length);
    break; //switch (rangeHeader?.Ranges.Count) case 1:
}
```

接下来就是可以正常发送单一部分内容的数据了，先要设置好 StatusCode，这里有一点 Spec 上不明确的但推荐实现的点，
就是如果 `end` 刚好达到了资源的末尾，那么应该将 StatusCode 设置成 200 ，语义化更明确。

```csharp
Response.StatusCode = rangeEnd == resourceStream.Length
    ? StatusCodes.Status200OK
    : StatusCodes.Status206PartialContent;
```
接下来从资源数据流读取对于 range 上的片段，缓存起来，注意我这里偷懒只用放内存，
实际生产需要斟酌一下哦，另外，这里推荐使用 ArrayPool 来优化内存操作，避免大量临时对象产生。
```csharp
var imageBytes = ArrayPool<byte>.Shared.Rent(length);
var readLength = await resourceStream.ReadAsync(
    imageBytes.AsMemory((int)rangeStart, length), token);
```
准备好数据就设置 Content-Range 和 Content-Length，并把内存数据拷贝到 Http Response数据流里：
```csharp
Response.Headers.ContentRange =
    $"bytes {rangeStart}-{rangeStart + readLength}/{resourceStream.Length}";
Response.ContentLength = readLength;

await StreamCopyOperation.CopyToAsync(new MemoryStream(imageBytes),
    Response.Body,
    readLength,
    bufferSize: 64 * 1024,
    token);

_logger.LogInformation("Respond a single {Range} with {ContentRange}",
    firstRange,
    $"bytes {rangeStart}-{rangeStart + readLength}/{resourceStream.Length}");
```

###### 多段部分内容响应

多段部分内容比较复杂，为了演示，这里尽量简化，仅仅考虑合并重叠 range 的场景。
一开始同样需要逐个验证这些 range 的合法性，简单的判断就是不能小于 0， 不能大于资源的长度；
和单一段的方法不同，这里为了方便后续合并操作，直接先逐个处理了一下 range

```csharp
// 这里模拟读取资源的整个数据流
var resourceStream = Assets.Img10_Jpg!;

var normalizedRanges = rangeHeader.Ranges.Select(r =>
        new RangeItemHeaderValue(r.From ?? 0, r.To ?? resourceStream.Length))
    .ToList();
if (normalizedRanges.Any(r =>
        r.From! < 0 || r.To! - r.From! > resourceStream.Length))
{
    Response.StatusCode = StatusCodes.Status416RangeNotSatisfiable;
    Response.Headers.ContentRange = $"*/{resourceStream.Length}";
    _logger.LogWarning("The request {Ranges} are invalid, it should be {From}
        normalizedRanges.Select(r => 
            r.From! < 0 || r.To! - r.From! > resourceStream.Length).ToList(),
        0,
        resourceStream.Length);
    break;
}
```

接着可以合并 range，按照多个段处理（这里忽略合并后只有一个的场景哈），直接利用 `MultipartContent`
来快速处理（实在不想过于细节，手动实现内存拷贝啦），核心逻辑如下：

```csharp
Response.StatusCode = StatusCodes.Status206PartialContent;
var content = new MultipartContent("multipart/byteranges", "THIS_STRING_SEPARATES");
foreach (var range in MergeRanges(normalizedRanges))
{
    var rangeStart = range.From!.Value;
    var rangeEnd = range.To!.Value;
    var length = (int)(rangeEnd - rangeStart);
    var imageBytes = ArrayPool<byte>.Shared.Rent(length);
    var readLength = await resourceStream.ReadAsync(
        imageBytes.AsMemory((int)rangeStart, length), token);
    var contentPart = new ByteArrayContent(imageBytes);
    contentPart.Headers.ContentType = MediaTypeHeaderValue.Parse(MediaTypeNames.Image.Jpeg);
    contentPart.Headers.ContentLength = readLength;
    content.Add(contentPart);
}
await content.CopyToAsync(Response.Body, token);
```

###### 完整内容响应

此时没有 Range 请求头或者请求头数据为空，直接返回整个资源，示例如下：

```csharp
var resourceStream = Assets.Img10_Jpg!;
var length = (int)resourceStream.Length;
var imageBytes = ArrayPool<byte>.Shared.Rent(length);
var readLength = await resourceStream.ReadAsync(
    imageBytes.AsMemory(0, length), token);
Response.StatusCode = StatusCodes.Status206PartialContent;
Response.ContentLength = readLength;
Response.Headers.ContentRange =
    $"bytes {0}-{readLength}/{resourceStream.Length}";
await StreamCopyOperation.CopyToAsync(new MemoryStream(imageBytes),
    Response.Body,
    readLength,
    bufferSize: 64 * 1024,
    token);
```

#### ASP.NET Core 对 Partial Content 支持

###### StaticFileMiddleware

ASP.NET Core 预制了很多组件，其中利用率很高频的就是这个 `StaticFileMiddleware` 组件了，
它可以快速将指定的目录（默认为 `wwwroot`）当初一个静态站点来使用，使用起来也很方便，大抵就是 Configure 方法中一句话：

```csharp
app.UseStaticFiles()
```

如果在 `wwwroot` 下放一个相对比较大一点的视频，然后放一个 html 页面并使用 video 标签引用这个视频，
通过浏览器开发者工具的网络 tab 就可以看到如下图类似的结果，这里 Http 状态码就可能是 206，并且同时
看到对应的响应头 `Accept-Range` 和 `Content-Range`。

![StaticFileMiddleware](/assets/images/http-partialcontent/staticfilemiddleware.png)


###### FileStreamResult

如果想要在 Controller 中返回支持 206，`FileStreamResult` 就派上用场了，当然，实际使用的时候，
不需要自己手动实例化一个 `FileStreamResult` 对象，通常可以调用 `Controller` 上的 `File(...)` 方法，
并传入合适的参数就 ok，示例如下，注意代码中的参数 `enableRangeProcessing` 是 `true`，表示期望自动处理 Range 请求。

```csharp
public class VideoController : Controller
{
    private readonly IWebHostEnvironment environment;

    public VideoController(IWebHostEnvironment environment)
    {
        this.environment = environment;
    }
    
    // GET
    [HttpGet, Route("videos/video.mp4")]
    public IActionResult Index()
    {
        var video = environment
            .WebRootFileProvider
            .GetFileInfo("video.mp4")
            .CreateReadStream();

        return File(video, "video/mp4", enableRangeProcessing: true);
    }
}
```

这样，就可以得到和上文 `StaticFileMiddleware` 类似的结果啦。

## 验证和测试 206

#### 测试服务器是否支持

简而言之，要想知道服务器是否支持 206，只需看一下任意服务器的资源的响应头，利用 `curl` 很容易做到：
```bash
$ curl -I https://martinfowler.com/microservices/ms-article.png
```
这个请求是 Martin Fowler 介绍微服务的一张插图，可以看到网站的响应类似
```http
HTTP/1.1 200 OK
Date: Sun, 18 Sep 2022 01:07:07 GMT
Server: Apache
Strict-Transport-Security: max-age=31536000; includeSubDomains;
Last-Modified: Mon, 10 Sep 2022 19:45:37 GMT
ETag: "61039-5ebcd0918da40"
Accept-Ranges: bytes
Content-Length: 397369
Content-Security-Policy-Report-Only: default-src https: 'unsafe-inline' 'unsafe-eval'; report-uri https://b3ceba9babf02086c0dca962bbbd1cda.report-uri.io/r/default/csp/reportOnly
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Type: image/png
```
从 Http 规范可以知道判断是否支持 206，要看响应里是否有 `Accept-Ranges: bytes`，从上面的结果恰好有，这就说明这个站点是支持的。

#### 发送 Range 请求头

`curl` 命令支持设置请求头，需要指定 `--header “KEY: VALUE”` 作为参数，继续用上面的插图为例，如下代码示例分成两段来下载文件，然后合并到一起得到完整图片。

```bash
$ curl --header "Range: bytes=0-200000" https://martinfowler.com/microservices/ms-article.png -o part1.png
$ curl --header "Range: bytes=200001-" https://martinfowler.com/microservices/ms-article.png -o part2.png
$ cat part1.png part2.png >> final.png
```

另外，也可以用 `curl` 的 `-r 0-200000` 来替代手动设置 Header。

值得一提的是，由于图片和视频的格式特殊性，通常获取的部分是可以预览的，例如上面的 `part1.png` 就可以看到原图上半部分大约一半多一点的图。


#### 使用 `HTTP request` 来发送请求

现在很多工具都支持 `HTTP request` 来简单定义请求，并直接执行，例如 Jetbrains 系列都可以创建 HTTP Request 文件，在 IDE 里执行这些请求。

```
# response first part
GET https://martinfowler.com/microservices/ms-article.png
Range: bytes=0-200000

###

# response second part
GET https://martinfowler.com/microservices/ms-article.png
Range: bytes=200001-

```


## 引用和参考
- [206 Partial Content on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/206)
- [206 Partial Content in RFC-7233](https://rfc-editor.org/rfc/rfc7233#section-4.1)
- [206 Partial Content in RFC-9110](https://httpwg.org/specs/rfc9110.html#status.206)