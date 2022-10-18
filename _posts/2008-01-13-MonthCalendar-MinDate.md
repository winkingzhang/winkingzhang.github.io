---
title:  "为什么 MonthCalendar.MinDate 是 01/01/1753 ?"
header:
  teaser: "/assets/images/teaser.jpg"
categories:
  - CSharp
tags:
  - MonthCalendar
---

查看MSDN上的 `MonthCalendar.MinDate` 的说明，发现属性默认值为 `01/01/1753`，
很是不解——为什么最小日期是1753年1月1日，找了许多文档都没有找到合理解释，
恰好今天看到ms monthcal的部分代码，里面的一段代码注释恰好就说明了这个问题。

这段代码里面定义并解释了两个名词
- 新纪元（Epoch）：宇宙的最开始（这真是个恐怖的词句！，好在还有个附注——这个就是MS支持的最早日期啦）；
- 世界末日（Armageddon）：圣经解释是“哈米吉多顿，世界末日善恶决战的战场”，说它是宇宙末日貌似没有问题的咯。

然后就道出了两个概念的值，这里 `Epoch` 就是 `1752年9月14日` ，原因是英美历史上 `1572年9月14日` 的前一天是 `1572年9月2日`，
而Armageddon就是4位记年的最大值（9999年12月31日），看看这里，这位大哥还挺逗，居然考虑到了10k年的问题，）。

```c
//
//  Epoch = the beginning of the universe (the earliest date we support)
//  Armageddon = the end of the universe (the latest date we support)
//
//  Epoch is 14-sep-1752 because that's when the Gregorian calendar
//  kicked in.  The day before 14-sep-1752 was 2-sep-1752 (in British
//  and US history; other countries switched at other times).
//
//  Armageddon is 31-dec-9999 because we assume four digits for years
//  is enough.  (Oh no, the Y10K problem)
//
const SYSTEMTIME c_stEpoch      = { 1752,  9, 0, 14,  0,  0,  0,   0 };
const SYSTEMTIME c_stArmageddon = { 9999, 12, 0, 31, 23, 59, 59, 999 };
```
