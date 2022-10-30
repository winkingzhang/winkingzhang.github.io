---
title:  "解决 git “unsafe repository” 问题"
header:
  teaser: "/assets/images/teaser.jpg"
categories:
  - git
tags:
  - git
excerpt: >
  前段时间不小心删掉了自己的一个 github repo，找了很久在一个备用U盘里发现有备份，
  开心的从U盘备份拷到机器上准备推到github上，然而，习惯性的运行 `git status` 时却收到了 unsafe repository 的错误
---

前段时间不小心删掉了自己的一个 github repo，找了很久在一个备用U盘里发现有备份，
开心的从U盘备份拷到机器上准备推到github上，然而，习惯性的运行 `git status` 时却收到了错误：

![git status error](/assets/images/git_unsafe_repo/git-unsafe-repo.png)

当然，其他的 git 命令也是提示错误。然后网上搜索了一下，这个错误也可能是这样的
![git pull error](/assets/images/git_unsafe_repo/git-unsafe-repo-1.png)


### 问题分析

为了搞清楚问题，我做了两个试验：
- 找另一个本地存在的 git repo，执行 `git pull` 和 `git status`，命令，工作正常
- 找另一个本地存在的 git repo，拷贝到另一个位置，再进去执行 `git pull` 和 `git status`，命令，工作正常
- 在一个新的目录下执行 `git clone` 一个全新干净的 github repo，也可以可以正常使用 `git status`， `git add`， `git commit`。 

有点抓瞎，看来不是我想象的备份拷贝操作造成的，只要祭出网络搜索，
结果还真在 [Stackoverflow 帖子](https://stackoverflow.com/questions/71901632/fatal-error-unsafe-repository-home-repon-is-owned-by-someone-else)
发现了一些端倪。这里提到 git 在 `2.35.2` 修复了一个安全漏洞，而我本地的 git版本是 `2.37.1`。显然我肯定也是受它累了。

然后我仔细看了一下这个安全漏洞详细描述，发现了这一段话：

> This version changes Git’s behavior when looking for a top-level .git directory to stop when its directory traversal changes ownership from the current user.

翻译过来就是
> 该版本改变了对顶层 .git 目录的递归检索的规则，当发现目录所有权不是当前用户就立即停止

嗯，那我来看看出错的目录的所有权是啥样子的

![directory ownership](/assets/images/git_unsafe_repo/git-unsafe-repo-owner.png)

这里看起来很奇怪，通常应该是用户名，但是这里看到的只是一个 SID，这说明当前的系统不能识别这个 owner，
也就是说，显然这里目录的所有者不是当前用户。而前面测试的三种情况都是我自己创建的目录，肯定 owner 是当前用户的。

知道了原因，修复起来就容易啦。走起……


### 解决思路

最直接的方法就是在目录权限对话框里直接把 owner 设置成我自己（也就是当前用户），如下图

![change owner](/assets/images/git_unsafe_repo/git-unsafe-repo-change-owner.png)

这对于少数几个目录出问题，手动修复是毫无问题的。但是，当有很多目录的时候，我们就得想一下是否可以批量处理。
一个折中的办法是将这些目录的父级目录的 owner 设置成当前用户。

对于程序员来说，这个手动方法有点 low，对于 Windows 有一个自带工具可以快速改变文件的 owner， 它就是 `takeown.exe`。

```batch
> TAKEOWN /F . /R /D Y
```

这里有三组参数需要注意：

- /F: 指定要改变 owner 的文件或者目录，也可以用 `*` 来通配所有文件。这里使用 `.` 来表示当前目录
- /R: 自动子目录递归
- /D: 自动应答命令行提问，这里 `Y` 表示自动接受


当然，这里也可以使用命令行错误提示里的方法来绕过，但是这只是 workaround，不推荐使用，仅当无法修改 owner 时的一种备用方案。
```
git config --global --add safe.directory *
```

PS：Mac Osx 和 Linux 解决方法：
```bash
$ sudo chown -R $USER .
```
