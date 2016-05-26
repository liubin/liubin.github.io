---
layout: post
title: "如何使用 git diff 来对比Office文档？"
date: 2016-04-26 12:51:35 +0800
comments: true
categories: 
---

我们知道用 `git diff` 命令比较文本文件还是比较容易的，Office文档的修改能比较么？

答案是可以的。下面我们就来看一下如何对Office文档进行比较。例子为在OS X上使用tika来将Office文本化之后再进行比较。


- 安装tika

```
brew install tika
```
​
这可能需要一些时间，如果你的网速不快的话。

- 创建脚本

然后编辑一条用于比较Office文档的命令

```
# 文件名 ~/bin/diffoffice

#!/bin/sh
tika -t "$1"
```

并将文件设置为可执行权限

```
chmod +x ~/bin/diffoffice
```

- 编辑 git 配置文件

编辑 `~/.gitconfig` 文件

添加如下内容：


```
[diff "office"]
	binary = true
	textconv = ~/bin/diffoffice
```


- 编辑代码仓库的属性文件

在代码仓库的根目录下，创建 `.gitattributes` 文件，其内容如下：

```
*.pptx diff=office
*.docx diff=office
*.xlsx diff=office
```


上面操作完成之后，就可以对Office的“二进制文件”进行对比了。



```
$ git diff keynote.pptx

... ...

index 68a6a9a..bf108f9 100644
--- a/20160423-docker-monitoring/keynote.pptx
+++ b/20160423-docker-monitoring/keynote.pptx
@@ -175,7 +175,7 @@ SaaS
 采集
 存储
 展示
-报警（动作）
+报警（事件处理）
 
```

