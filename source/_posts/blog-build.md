---
title: 个人博客搭建总结
date: 2020-06-10 14:00:00
tags: 
 - blog 
 - hexo
categories:
 - 博客搭建
index_img: /img/blog-build.jpg
banner_img: https://wallroom.io/img/1920x1080/bg-02fdff7.jpg
---

<i class="fas fa-coffee"></i>

折腾了一番，博客顺利搭建完毕。不得不说到了2020年的今天Hexo确实很强大了，以往个人博客比较难配置的一些站长功能都已经有了现成的集成，特别是主流的几个Theme模版都囊括了常用的几乎所有博客功能，只要自己愿意玩玩技术，不需要多少编程能力就能独立搭好不弱于任何主流社区的个人博客了，巴适得很。

总结一下几个比较容易踩坑的地方：

###  git repo 嵌套问题

这个问题应该所有想用git管理博客的blogger都会遇到，而且如果不擅长查资料的话会非常坑，因为官方基本没怎么提这个问题。

当我们搭建博客项目的时候，一般博客本身就是一个repo，而Hexo会把你下载的主题放在themes这个子文件中，如果你是用 git clone 来下载的话，就会发现当你试图build博客的时候，本地运行一切正常，但一旦你deploy到Github Pages就会收到下面的邮件：

> You are attempting to use a Jekyll theme, which is not supported by GitHub Pages. Please visit https://pages.github.com/themes/ for a list of supported themes. If you are using the "theme" configuration variable for something other than a Jekyll theme, we recommend you rename this variable throughout your site. For more information, see https://help.github.com/en/articles/adding-a-jekyll-theme-to-your-github-pages-site.

这个问题乍一看好像是Github不识别Jekyll主题，但很明显我们用的是Hexo而不是Jekyll模版，如果你直接搜索这些关键词会找到很多几年前很类似的问题。当时Github会默认使用Jekyll去解析网页项目，所以需要在项目里放一个空的 .nojekyll 文件来让Github知道不需要进行解析，但这个问题已经被解决了。

真正导致这一问题的关键在于repo的嵌套导致的deploy失败，由于你clone的主题本身就是一个repo，而你的博客项目是另外一个repo，两个repo有了嵌套关系就会导致Github不知道如何处理deploy上来的文件。

所以要用 git submodule 来导入主题，而不是 git clone

```bash
git submodule add https://github.com/geektutu/hexo-theme-geektutu.git themes/geektutu
```

这样theme模版项目作为一个子模块加载到博客项目里面来，层次逻辑就不会混乱了。

当你要更新模版时也很简单：

```bash
git submodule upate
```

不过还是推荐直接下载主题到本地，解压放到theme目录下，不要使用submodule，就不用折腾了。

### css等资源加载失败

deploy成功之后可能还会发现下面的情况：

![css error](https://raw.githubusercontent.com/Yikhan/ImageHost/master/blog/1564116640441.png)

css和js等资源文件统统加载失败，博客只有内容而没有样式。

404问题一般都是由于访问路径错误导致的，可以看一下这些文件是在访问什么路径的时候失败的，比如:

```bash
正确的:
yourname.github.io/blog/css/1.css

实际访问:
yourname.github.io/css/1.css
```

其实Hexo文档里面已经提示过你了：

>  URL
If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'

当你的博客网址不是第一路径时（就是网址不是../..这种形式，有/就表示你的博客在一个子路径里面），你要密切注意子路径的配置方式。很多人的博客项目实际网址是 yourname.github.io/blog/ 这时候你就必须要在 \_config.yml 文件里面严格按照下面的方式来配置:

``` yaml
url: https://yourname.github.io/blog/
root: /blog/
```

这是我在搭建博客时候碰到的两个坑，总结一下方便其他朋友绕坑。

### 如何删除submodule

如果想停止使用submodule的方式引入主题，可以使用下面的命令：

```bash
# Remove the submodule entry from .git/config
git submodule deinit -f path/to/submodule

# Remove the submodule directory from the superproject's .git/modules directory
rm -rf .git/modules/path/to/submodule

# Remove the entry in .gitmodules and remove the submodule directory located at path/to/submodule
git rm -f path/to/submodule
```
