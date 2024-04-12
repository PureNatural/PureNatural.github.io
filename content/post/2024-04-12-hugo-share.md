---
layout:     post
title:      "GitHub+Hugo搭建个人博客经验分享"
subtitle:   ""
date:       2024-04-12
author:     "YuRonan"
image: "/img/hugo-header.png"
published: true 
tags:
    - Hugo
URL: "/2024/04/12/hugo-share"
categories: [ "Tech" ]    
---
## 主题选取

在选择主题时，我尽量选择排版较为充实，内容能够铺满屏幕的设计，这样网站显得大气简洁一些，同时也希望能够在移动端做展示，方便网友随时随地可以访问博客阅读文章，于是我选择了hugo-theme-cleanwhite主题来搭建博客网站，该主题主要页面设计如下：

![theme](/img/theme/main-page.png)
<center>theme</center>

个人很喜欢头部的图片，能够让博客网站显得不那么单调，同时首页可以显示个人的主要信息，还可以为朋友的博客引流，是一个不错的小功能，同时页面头部也支持多种类型的博客展示，个人希望不仅仅把它当作是一个技术博客，关于兴趣爱好方面的文章可以放到网站中，包含但不限于电影、音乐、旅行方面，这样可以更好的向大家展示个人的一些爱好，吸引更多的小伙伴。


## 搭建过程

网上较多的教程将编译前的代码和编译后的代码分成了两个仓库，我这里为了方便放到了一个仓库，但会导致两部分代码混到一起，我发现在GitHub Pages的页面可以设置具体哪个文件夹用于部署，如下图所示

![pages-doc](/img/theme/pages-doc.png)
<center>pages-doc</center>

如果当前仓库即为编译后的代码，则不需要选择文件夹。

在使用hugo命令编译后，代码默认在public文件夹下，但另我纳闷的是在GitHub上我只能选择/docs文件夹，不知道什么原因无法选择其他文件夹，于是在每次编译后我必须把public改为docs这样即可以部署成功，这是一个相对麻烦的点。

## 技术选择

助教提供了三个框架可以选择hexo,jekyll,hugo,其实3者都不算特别难，我在选择的时候，先选择了自己喜欢的主题，随后该主题使用了hugo，故个人选择了hugo，随后又对其他两种技术进行了了解，发现以下几点区别

* Hugo star数最高72.3k，Jekyll star数48.2k，Hexo star数38.4k。
* Hugo 编译速度最快
* Hugo 不需要root权限即可部署成功

从上面几点来看，Hugo确实是一个不错的选择。