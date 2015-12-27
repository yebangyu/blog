---
layout: post
title: "Windows下OctoPress环境搭建"
date: 2015-10-17 11:28:19 +0800
comments: true
categories: Web
tags: [octopress, 博客, Windows]
keywords: octopress, Windows, 搭建, 博客
description: Windows下Octopress博客搭建
---

------

近期学习了如何搭建个人博客的方法，这里备忘一下，如果能帮助到别人，那就更好了。

需要安装的组件主要包括OctoPress、Github、Ruby、Python等，在配置的过程中，主要参考了本文末尾的参考链接，在此表示衷心的感谢。

### Github

首先，得有一个Github的账号，便于托管。在申请了Github账号之后，建立一个个人仓库，仓库的命名必须是`yourusername.Github.io`这样的形式。如果您有购买个人域名，还可以通过CNAME来完成绑定。这一步可以google一下

### Ruby

安装Ruby的时候需要特别注意勾选“Add Ruby executables to your PATH”

<!--more-->


### DevKit

下载解压到某个目录（例如DevKit），打开cmd，执行如下口令

    cd DevKit
    ruby dk.rb init 
    ruby dk.rb install

### Python

安装python2.7并添加系统变量。切记，一定不能用python3，否则很多功能没法正常使用，例如代码高亮。

安装easy_install，然后在cmd中执行如下命令安装pygments

    easy_install pygments
    
### OctoPress

首先，通过git命令将OctoPress下载到本地

    git clone git://Github.com/imathis/octopress.git octopress
    
切换到目录，然后执行

    gem sources -a https://ruby.taobao.org/
    gem sources -r http://rubygems.org/
    gem sources -l
    
请特别注意，第一行里，是https，而不是http。

修改Gemfile下的文件，把将第一行的http://rubygems.org/ 改为https://ruby.taobao.org/ 

然后，依次执行如下命令

    gem install bundler
    bundle install

并安装Octopress的默认主题

    rake install
    
环境基本上就配置好了，运行`rake preview`，然后打开http://127.0.0.1:4000/ 看看效果吧，恩，本地的效果。接下去就要写文章，继而发布到Github上了。

### 编写文章

运行`rake new_post['helloworld']`

这样就可以在octopress/source/_posts 下生成一个markdown文件，然后就可以开始通过编辑该文件来写文章了。我一般使用Cmd Markdown 编辑阅读器来写文章，很适合我这样的新手

写完文章，就可以生成了，命令是`rake generate` ，然后再通过前面的`rake preview`来预览下

### 发布

运行命令，`rake setup_Github_pages` 来设置您的Github账号。注意，这一步得在git bash下完成，而不是Windows命令提示符下。

运行命令，`rake deploy` ，将文章发布到Github上。
    
这样一篇文章就发布出去了。以后写新文章或者更改文章后，只需要`rake generate` 然后 `rake deploy` 就可以啦

### 参考资料：

Octopress在MAC下的安装、配置

http://www.chongh.wiki/blog/2015/12/16/octopress-tutorial/

Ocotpress在Windows下的安装、配置：

http://www.cnblogs.com/oec2003/archive/2013/05/27/3100896.html

Octopress的个性化配置：

http://www.tuicool.com/articles/miaUR3 

http://www.tuicool.com/articles/VbqYNjn

http://foggry.com/blog/2014/04/28/custom-your-octopress-blog/

Octopress添加多说评论功能：

http://droidyue.com/blog/2014/07/29/integrate-duoshuo-in-octopress/   

http://dev.duoshuo.com/threads/5364eb968df62dd658bc77af

其他：

http://blog.txx.im/blog/2013/09/04/liquid-exception-about-braces/

http://khaos.Github.io/blog/2012/12/06/using-chinese-category-tags-in-octopress/

https://Github.com/imathis/octopress/issues/1683

http://stackoverflow.com/questions/14932141/octopress-capitalization-in-new-posts

http://blog.depressedmarvin.com/2014/07/08/new-google-fonts-cdn/

