---
title: "我的Neovim心得"
date: 2023-09-10T23:18:43+08:00
# weight: 1
# aliases: ["/first"]
tags: ["vim"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
NeoVim的基本用法及我的配置教程
---
Neovim是Vim的现代版本, 早于Vim提供异步任务特性, 更新激进, 社区活跃.

[toc]
---

VIM的作者去世了,我刚接触编程的时候使用的第一个编辑器就是VIM,但是还不知道VIM插件的存在,在没有任何提示辅助的条件下编写完成了很多个Python程序. 
后来知道了VIM插件, 在一通极其复杂的copy-paste配置之后运行ctags检索符号.
vscode很快发布了, 我也因为vscode大而全又方便使用了很多年的vscode.
再后来我感到编写程序的过程在完成构思之后,最大的限制就是编辑效率, 经常觉得编辑效率跟不上我预期的开发速度. 
在vscode上又启用了VIM键位, 但是VIM插件终究是一个模拟器, 模拟了VIM的键位但是缺少VIM的更多指令.
另一个感受是vscode上插件的日新月异, 与我一次学习终身受用的出发点背道而驰.
我决定重新回归VIM的怀抱.
NeoVim是Vim的现代版本, 社区远比Vim更活跃, 并且可以看到持续的活跃. 以NeoVim作为回归Vim的终点, 应该是最好的选择.
## 安装引导
Nvim的配置文件默认位于`~/.config/nvim/`目录, 默认配置文件`init.lua`.
插件管理器选择`LazyNvim`, 这个插件管理器界面和配置现代一些, 配置文件里面不需要很多分隔符, 用起来还可以.

## 配置C/CPP开发环境
任何编程语言的开发环境, 在能力上应包含三个功能:
1. 符号查找: 查找变量和函数的声明、定义和引用
2. 重构: 变量和函数的重命名
3. 代码补全

在显示上应提供的功能:
1. (侧边)(常驻)文件浏览器
2. (侧边)(常驻)code layout/outline
3. git 集成: make diff、checkout operations fast

搜索功能:
1. 按文件名搜索文件
2. 按字符搜索出现位置
3. 按正则规则进行以上搜索

### 插件的选择
# IDE三大件
## 符号查找
符号查找有两种方案, 第一代的语法树方案, 第二代LSP方案
### 查找声明
局部变量或者全局变量的声明由于局限于一定的作用域, 并不难找. 
### 查找定义
函数的定义可能会出现在不同的位置, 由编译宏开关控制真实采用的定义.
所以查找符号要有确定当前环境宏定义的能力, 才能找到正确的定义.
### 查找引用
与函数有多个定义类似, 相同签名的不同函数定义也分别有其应用. 它们不一定是互斥的, 也可能是相交或者包含等等关系. 
准确的查找引用和查找定义, 背后需要的原理是一样的.
## 重构: 重命名

## 代码补全
