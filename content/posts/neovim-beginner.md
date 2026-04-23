---
draft: true

title: "Neovim Beginner"
date: 2023-10-15T16:03:14+08:00
lastmod: 2023-10-15T16:03:14+08:00
# weight: 1 . # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
# aliases: ["/first"]
tags: ["Neovim"]
author: "fanyuxin"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
comments: true
description: "Neovim Design Tradeoff" # 文章描述，与搜索优化相关
summary: "" # 文章简单描述，会展示在主页
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
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
    URL: "https://github.com/fanyxok/fanyxok.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Neovim Beginner
===
init.lua
---

### Install.sh
`plugins.lua` is where we start to configure Neovim.

All Lua configuration files are under the lua folder (:h lua-require).

### Startup Screen
Use alpha.nvim as the startup screen. 

The configuration file is `lua/config/alpha.lua`.

`alpha.lua`:

- pcall is used to prevent errors requiring a non-existent module or a module with syntax errors.
- We specify an ASCII art header, some convenient shortcuts, and a footer to indicate the number of plugins, date and time, and a quote.

### Git
use `Neogit` for version control. 

The configuration file is `lua/config/neogit.lua`.

```lua
local M = {}function M.setup()
  local status_ok, neogit = pcall(require, "neogit")
  if not status_ok then
    return
  end  neogit.setup {}
endreturn M
```

- pcall is used to prevent errors requiring a non-existent module or a module with syntax errors.
- For now, we use the default configuration for Neogit.

### options

Besides Vimscript files, Lua files can be loaded automatically from special folders in your runtimepath (:h load-plugins). All *.vim files are sourced before *.lua files.
We configure a few default settings by using after/plugin/defaults.lua.

defaults.lua

We configure <Space> as the Leader key (:h <Leader>).

For each setting, you can check out the help documentation for more details. We shall see how to further customize Neovim with sensible defaults later.
We configure an autocmd to highlight the text when yanking or copying.
We also configure a file type plugin (:h ftplugin) for Lua files with buffer-specific settings.

A filetype plugin is like a global plugin, except it only sets options and
defines mappings for the current buffer.

vim.bo.shiftwidth = 2
vim.bo.tabstop = 2
vim.bo.softtabstop = 2
vim.bo.textwidth = 120
To remove the configuration, simply delete the ~/.config/nvim-beginner folder.

If we want to start nvb in a new terminal without installing all the plugins again, put the following lines in the RC files, e.g. .zshrc, or .bashrc.

Key Mappings ans WhichKey
---
Plugin Management
---
Status Line
---
Fuzzy File Search
---
File Explorer
---
Buffer
---
Motion
---
Built-in Completion
---
Completion Plugin
---
Auto Pairs
---
LSP
---
LSP using null-ls.nvim
---
LSP Plugin
---
Debuging using DAP
---
Testing
---
Debugging using vimspector
---
Performance
---
Snippets
---
Lua Autocmd and Keymap Functions
---
Color Scheme
---
Remote Plugins
---
Session
---
Snippets using Lua
---
Refactoring
---
Code Annotation and Documentation
---
Note Taking, Writing, Diagramming, and Presentation
---
Source Code Control
---
Window Bar
---
User Interface
---
Python Remote Debugging
---
Python Code Refactoring
---
Plugins for Key Mapping
---
Code Folding
---
Terminal Debugger
---
GUI
---
Cheatsheet and Coding Assistant
---
Conventional Commits
---
Database Explorer
---
Code Context
---

