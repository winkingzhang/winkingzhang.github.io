---
title:  "Nodejs使用并信任自签名SSL证书"
header:
  teaser: "/assets/images/teaser.jpg"
categories:
  - nodejs
tags:
  - node
  - ssl
---

SSL证书是客户端和服务器安全通信的保障，但是有点小贵，而且在开发服务器上有点大材小用。
这就引出了自签名证书这个话题了，本篇我们将分成四个段落来详细阐述：
- 如何创建自签名证书
- 在nodejs中使用自签名证书
- 客户端验证和错误处理
- 如何信任自签名证书


### 如何创建自签名证书

创建自签名证书其实很简单，只需要有 `openssl` 命令行工具就ok了

{: class="alert-info" }
如何安装openssl，可以参考 [官方文档](https://github.com/openssl/openssl#build-and-install)

以下是我的示例

```bash
$ openssl req -x509 -nodes -days 365 \
    -newkey rsa:4096 \
    -keyout selfsigned.key \
    -out selfsigned.crt
```

对应输出如下：

![openssl命令行输出](/assets/images/nodejs/openssl.png)

正如以上示例和输出，`openssl` 会根据配置参数提出一些问题并要求输入，完成之后就自动生成了产物
- `selfsigned.crt`, 证书主体，客户端可见
- `selfsigned.key`, 证书私钥，通常需要安全的保存在server端

需要注意的是，这里建议指定RSA 4096位秘钥，以确保安全；`-days 365` 是证书有效期，可以根据需要调整

### 在nodejs中使用自签名证书

##### 在https模块中使用自签名证书

众所周知，在node开启https需要 `require('https')`，
然后调用 `https.createServer(options, callback)` 来启动https服务

```javascript
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('selfsigned.key', 'utf8'),
  cert: fs.readFileSync('selfsigned.crt', 'utf8')
};

https.createServer(options, function (req, res) {
  res.writeHead(200);
  res.end("hello world over HTTPS\n");
}).listen(8443);
```

这里需要注意的是 `fs.readFileSync` 不同于 `fs.readFile`，`fs.readFileSync` 是一个同步函数，
会阻塞当前进程知道函数返回（或者出错），在这里实际上是读取必不可少的配置数据，因此能够接受。

{: class="alert-success" }
在命令行执行 `node app.js` （这里app.js是上述代码所在文件）以启动https服务

可以看到如下 `curl` 测试结果
![node https server](/assets/images/nodejs/node-https-app.png)


##### 在expressjs中使用自签名证书

本质上来说在expressjs中使用自签名正式和https模块是没有区别的，
但是封装还是带来了一些微小的差别，代码如下：

```javascript
const fs = require('fs');
const http = require('http');
const https = require('https');

const options = {
  key: fs.readFileSync('selfsigned.key', 'utf8'),
  cert: fs.readFileSync('selfsigned.crt', 'utf8')
};

const express = require('express');
const app = express();

// your express configuration here

const httpServer = http.createServer(app);
const httpsServer = https.createServer(options, app);

httpServer.listen(8080);
httpsServer.listen(8443);
```

### 客户端验证和错误处理

正如上文 `curl` 测试结果图上所示，默认是抛出一个错误的，关键的信息片段如下

> `curl: (60) SSL certificate problem: self signed certificate`

意思是说，我发现这个证书是自签名证书，`curl` 无法验证它的有效性。
简单的场景，我们可以使用 `-k` 参数来忽略这个错误，告诉 `curl` 跳过验证。
当然，也可以通过 `--cacert` 参数来指定根证书以验证服务端证书，而自签名证书根证书就是自身，自然可以通过的。


 接下来看看在nodejs客户端如何和https服务处理，首先，尝试最基本的用法：
```javascript
https.get('https://localhost:8443')
```
不出意外，我们收到 `self signed certificate` 错误

![self signed certificate](/assets/images/nodejs/node-https-client-err.png)

和上面的 `curl` 类似，只需要传入根证书（自签名证书的根证书就是它自己）就可以通过校验，示例如下：
```javascript
let options = url.parse('https://localhost:8443');
options['ca'] = fs.readFileSync('selfsigned.crt','utf8');
https.get(options)
// Output: 'GET / HTTP/1.1\r\nHost: localhost:8443\r\nConnection: close\r\n\r\n'
```

至此我们肯定会想，那我们为什么不能像Google，Bing那样不用特殊操作就受浏览器信任，而我们的自签名证书却不行呢？

这就涉及到SSL证书的两个重要概念，分别是根证书和中间证书，它们有点抽象，我们举个例子：

购房贷款这事基本上都清楚，一般银行都需要资产证明或者固定收入证明，这里拿固定收入证明做例子:

- 如果贷款银行不关心工作单位仅仅关注资产情况，那么就只需要拿着工资所在银行开具的流水或者存款证明并加盖银行签章，提交贷款申请就可以了，显然，这里银行就像一个根证书，它签发的证明，收入证明就是可信的，即使拿着甲银行的证明到乙银行也是可信的（根证书机构互信）。
- 如果贷款银行要求公司出具薪资证明，那么我们就得到公司财务出具薪资证明并加盖公司签章，然后再去银行提交贷款申请，这时，公司就可以理解成中间证书机构，它是由工商机构认证的单位，而工商机构和银行之间是互信的（根证书机构互信），因此，银行就能信任公司签发的薪资证明。

![certificate process](/assets/images/nodejs/certificate-proc.png)


### 如何信任自签名证书

上面我们提到了根证书，而我们的机器本身就信任了一些知名的根证书，对于我们的自签名证书，如果也能够加入到这些受信任的根证书一起，那么整个系统就自动信任了。

光说不练可不行，我们来实际操作一下，这里因为系统的设计不同，导致它们的方法也有所不同，下面列举几个常见系统的方法。

##### Ubuntu/Debian

```bash
$ sudo apt install ca-certificates -y
$ cp selfsigned.crt /usr/local/share/ca-certificates/
$ sudo update-ca-certificates --verbose
```

{: class="alert-info" }
注意：这里拷贝的文件后缀是 `crt`，并且确保文件的第一行是 `-----BEGIN CERTIFICATE-----`。

##### CentOS

```bash
$ sudo yum install -y ca-certificates
$ sudo cp selfsigned.crt /usr/share/pki/ca-trust-source/anchors/
$ sudo update-ca-trust force-enable
$ sudo update-ca-trust extract
```

##### Windows

```bash
> certutil -addstore -f "ROOT" selfsigned.crt
```

##### Mac OS X

```bash
$ sudo security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain "selfsigned.crt"
```

如果想将自签名证书在全系统级别受信任，需要将上面的证书目标路径从 `~/Library/Keychains/login.keychain` 替换成 `/Library/Keychains/login.keychain`。

{: class="alert-info" }
注意：在Mac下，浏览器和 `curl` 会自动信任新增的自签名证书，但许多编程语言并没有默认集成 `keychain`，因此不能自动通过自签名证书。

