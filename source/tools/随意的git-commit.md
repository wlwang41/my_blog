---
title: "随意的git commit"
date: 2014-10-27 14:40
---

有一种比较好的commit信息的组织格式叫做 **50/72 formatting**，总结起来也就是下面这几点:

- 第一行简要的表明这次commit干了什么，就好像给别人下达某种命令，不超过50字符
- 第二行是空行
- 一些补充信息，不超过72字符，并控制好单行长度

原文如下:

    Short (50 chars or less) summary of changes

    More detailed explanatory text, if necessary.  Wrap it to about 72 characters or so.
    In some contexts, the first line is treated as the subject of an email and the rest of the text as the body.
    The blank line separating the summary from the body is critical (unless you omit the body entirely);
    tools like rebase can get confused if you run the two together.

    Further paragraphs come after blank lines.

    - Bullet points are okay, too

    - Typically a hyphen or asterisk is used for the bullet, preceded by a
      single space, with blank lines in between, but conventions vary here

写好commit信息可以加速code review的速度，并且帮助其他人或者自己更好的维护代码，也便于代码回滚.

本来我一直老老实实写commit的log的，后来发现大家都是写奇奇怪怪的log，也就染上了随意写commit log的习惯。

那么问题来了，就算要随便写commit log，那怎么才能写得高贵冷艳让人一眼看不出来是瞎写的呢?

偶然的机会看到了这么一个网站: [http://whatthecommit.com/index.txt](http://whatthecommit.com/index.txt)，它能随机生成commit log

那么要想随便写commit log就只需要下面这行命令就行啦：

    git commit -m "`curl -s http://whatthecommit.com/index.txt`"
