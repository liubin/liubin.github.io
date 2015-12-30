---
layout: post
title: "开始新Blog"
date: 2015-12-30 18:47:07 +0800
comments: true
categories: 
---

以前的[Blog](http://liubin.org)是基于Wordpress的，最近一直打算换到GitHub上。

# 为什么使用GitHub作为Blog？

原来的Blog是基于Wordpress的，托管在dreamhost上，dm估计很多人已经不熟悉了，取而代之的可能则是linode。

dreamhost现在有几个问题，第一是贵，要$200一年，第二是速度现在显得有点慢，最后一个原因就是维护起来稍显麻烦。

而且，现在Markdown如此普及，如果不精通它，真的不能说是称职的软件工程师。


# 如何选择？

其实我是怕折腾的，所以选择标准主要有3点：

* 安装简单（包括文档容易阅读）
* 主题（theme）多
* 容易定制（从页面样式到功能，也就是程序本身）

而说到容易定制，就跟程序所使用的编程语言有很大关系了。而这其中，大部分都是用Ruby或者Node.js编写的，关于这两种编程语言，我都不排斥。


以 “GitHub blog” 为关键字，在Google搜一下能出现很多结果，主流应该就是以下几种：

* [Octopress](http://octopress.org/)
* [Jekyll](http://jekyllrb.com/)
* [Hexo](http://hexo.io/)
* [Pelican](https://github.com/getpelican/pelican)
* [Middleman](https://middlemanapp.com/)
* [Hugo](https://gohugo.io/)

这些工具，工作流程也很像，主要是：

- 创建项目（博客）
- 编写文章
- 生成静态文件
- 部署

在编写的过程中，这些程序也都会提供预览功能，即内置一个Web服务器，用户可以在浏览器上预览结果。同时，它们也都支持watch功能，即文件修改后自动重新生成预览文件，用户在浏览器上刷新页面即可看到最新的结果。


## Hugo

采用Go语言编写，特点是安装简单（Go程序的通性，一个应用一个文件就能打包），速度快。

缺点可能就是改起来麻烦，说实话我没写过太多的Go代码，而且我对构建的速度也不是太关注，因此割爱。

## Pelican

Pelicat除了支持Markdown，还支持[reStructuredText](https://zh.wikipedia.org/wiki/ReStructuredText)格式的文档。

不过Pelican采用Python编写，只好割爱。

## Middleman

Middleman使用了很多常见的在 Ruby on Rails 中用到的技术：Sass、CoffeeScript、Sprockets、Erb & Haml 等。不过感觉过于复杂，是一个整站生成工具，Blog只是其中一部分而已，比如要想使用Blog功能，需要这么来安装：

```bash
# install
$ gem install middleman
$ middleman init MY_PROJECT --template=blog

# Configuration
$ activate :blog
```


## Hexo

Hexo采用了Node.js，相比Jekyll和Octopress也没有发现更强大的地方，又没有找到好主题，因此放弃了Hexo。

我虽然不排斥Node.js，但也不是Node.js控。

## Jekyll

Jekyll是使用Ruby语言编写的静态网站生成工具，它很强大，也支持博客，不过我没有打算要那么多东西，况且还有一个更合适的系统叫做Octopress。

## Octopress

Octopress是基于Jekyll的、专门为GitHub使用的博客系统，在我看来，它的优点主要有以下：

* Jekyll的强大 + Octopress的专注（博客）
* 采用 [Solarized](http://ethanschoonover.com/solarized) 展示代码，对开发者友好。
* 主题多、插件多（基于Jekyll）
* 网上资料多、用户多

最主要的是，我在Octopress中发现了一个非常不错的主题。

# Octopress安装过程

安装过程也算简单，对于有Ruby基础的人来说，应该1分钟就能搞定了。


## 下载octopress仓库

我本机已经有了Ruby开发环境，因此安装只需要：

```bash
$ git clone git://github.com/imathis/octopress.git
$ cd octopress

$ bundle install
```

## 安装主题

这里，我[选择主题](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)花费了些时间，最后我选择了[k-ui-octopress-theme](https://github.com/kui/k-ui-octopress-theme)，这是一个十分精简的主题，不过这个主题是日文版本的，我修改了一下，成为了现在使用的[中文版本的kui主题](https://github.com/liubin/k-ui-octopress-theme)。


```bash
$ git clone git://github.com/liubin/k-ui-octopress-theme.git .themes/kui
$ rake install["kui"]

```
## 配置网站

对Octopress的主要设置都在`_config.yml`文件中，基本必须修改的内容包括URL、博客标题，作者信息等：


```ruby
url: http://liubin.github.io
title: 自言自语
subtitle: 一个软件工程师的自言自语
author: bin liu
simple_search: https://www.google.com/search
description:
```

详细内容，可以参考[官方的配置说明](http://octopress.org/docs/configuring/)

## 创建文章

```bash
rake new_post["Start new blog"]
```

这将会创建一篇新文章，保存位置为`source/_posts/2015-12-30-start-new-blog.markdown`。由此可见，文件的默认名称就是一个日期加上`标题.lower.join('-')`。

这里需要注意的是，如果`title`里写了中文，那么可能你的文件名就会变成汉语拼音的格式，而文件名，也会成为URL的一部分，为了URL的美观，建议大家在创建新文章的时候，使用英语，然后在源文件中修改文章标题。

创建文章并编辑时，就可以预览了。这只需要执行如下命令即可：

```bash
$ rake preview
```

然后，就可以[打开浏览器](http://localhost:4000/)查看编辑结果了。

## 发布到GitHub

如果本地预览没有问题，就可以发布到GitHub上去了。

首先需要设置GitHub信息：

```bash
$ rake setup_github_pages
Enter the read/write url for your repository
(For example, 'git@github.com:your_username/your_username.github.io.git)
           or 'https://github.com/your_username/your_username.github.io')
Repository url: git@github.com:liubin/liubin.github.io.git
Added remote git@github.com:liubin/liubin.github.io.git as origin
Set origin as default remote
Master branch renamed to 'source' for committing your blog source files
rm -rf _deploy
mkdir _deploy
cd _deploy
Initialized empty Git repository in /Users/liubin/github/octopress/_deploy/.git/
[master (root-commit) f94cd35] Octopress init
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
cd -

---
## Now you can deploy to git@github.com:liubin/liubin.github.io.git with `rake deploy` ##
```

然后，生成文件并部署：

```bash
# 生成文件
$ rake generate
# 发布到GitHub
$ rake deploy
```

## 提交Markdown原文

以上的步骤只是生成静态网站，而原文我们需要自己另外保存。

```
$ git add .
$ git commit -am "add new post" 
$ git push
```

到这里，安装配置一个GitHub博客，并且发表文章的一个大致过程就简要介绍完毕了。


# 遗留问题

软件这东西，开头很简单，精通很难。至少，在花了两个小时安装并写了本文之后，我还有以下一些问题。

## 旧数据如何导入？

老的Blog数据如何倒过来？如果你有什么好的工具或者经验，欢迎推荐给我。

## 定制样式是否方便？

说实话，对css比较没感觉，这也是自己比较劣势的地方。

# 总结

不过，总的来说，安装还算简单，对这个主题也比较中意，算是不错的开头。

