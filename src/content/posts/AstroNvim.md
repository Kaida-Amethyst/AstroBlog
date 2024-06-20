---
title: "AstroNvim - 把Neovim打造成IDE"
published: 2024-02-21
description: AstroNvim是基于Neovim的一套预配置环境，通过集成多种插件，提供了一种快速、现代的开发环境配置方法。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/AI/blackhole-1.png
category: 开发工具
tags: [Linux, Ubuntu, Vim]
---


--------------

AstroNvim是基于Neovim的一套预配置环境，通过集成多种插件，提供了一种快速、现代的开发环境配置方法。AstroNvim已经经历了几个版本，本篇博客介绍的是最近版v4的安装。

## Neovim安装

在安装AstroNvim之前，需要确保你的系统中已经安装了Neovim。对于ubuntu系统而言，可以通过以下命令安装Neovim：

```bash
sudo apt-get install neovim
```

或者直接下载nvim-appimage

```bash
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim.appimage
chmod u+x nvim.appimage
./nvim.appimage
```

对于其它系统，可以到neovim上的[git install page](https://github.com/neovim/neovim/blob/master/INSTALL.md)进行查找，这里不再赘述。

## AstroNvim安装

安装AstroNvim前，确保Neovim已正确安装。AstroNvim的安装非常简单，访问其[官方网站](https://astronvim.com/)并按照“Get Started”部分的指引进行即可，这里依然做一个简单梳理。

### 首次安装

首次安装AstroNvim，确保你的`~/.config`目录存在，然后运行以下命令：

```bash
git clone --depth 1 https://github.com/AstroNvim/template ~/.config/nvim
```

重启Neovim后，AstroNvim将开始初始化并安装额外的插件。

### 升级或重新安装

如果已经安装过AstroNvim，建议先备份旧的配置：

```bash
mv ~/.config/nvim ~/.config/nvim.bak
mv ~/.local/share/nvim ~/.local/share/nvim.bak
mv ~/.local/state/nvim ~/.local/state/nvim.bak
mv ~/.cache/nvim ~/.cache/nvim.bak
```

之后，再运行上述`git clone`命令来安装最新版本。

### 个性化配置

如果要对AstroNvim进行个性化配置，可以先删除`~/.config/nvim/.git`，然后将该目录放入自己的版本控制中。这样可以更方便地管理个人配置，并保留对官方配置的更新。

## 快捷键配置

AstroNvim的快捷键配置可在[官方文档](https://docs.astronvim.com/mappings)中找到。对于AstroNvim来说，熟练使用快捷键是提升开发效率所必须的。

## 主题配置

AstroNvim支持多种主题切换，使用`<leader>ft`可以快速浏览和更换内置主题。如需更改默认主题，可在`~/.config/nvim/lua/plugins/astroui.lua`中修改`colorscheme`值。

### 添加更多主题

如果官方主题不满足需求，可以通过AstroCommunity下载更多主题。例如，你可能会喜欢cyberdream-nvim主题：图示如下：

![cyberdream](https://b3logfile.com/siyuan/1644568593533/assets/cyberdream-20240528145632-jwmwez5.png)


要安装`cyberdream-nvim`主题，需在`~/.config/nvim/lua/community.lua`文件中添加：

```lua
{ import "astrocommunity.colorscheme.cyberdream-nvim" }
```

重启Neovim后，AstroNvim将自动安装并应用新主题。

## 插件的查找

一般来说，日常开发的大多数插件，都可以在AstroCommunity里找到，访问其[仓库页面](https://github.com/AstroNvim/astrocommunity/tree/main)，可以在lua/astrocommunity目录下查找插件，前面一小节提到的cyberdream-nvim就在仓库目录`lua/astrocommunity/cyberdream-nvim`下。确认需要安装这个插件之后，直接在community.lua中添加代码，**不需要手动从github上下载插件。**

## Copilot配置

AstroNvim支持通过AstroCommunity集成Github Copilot。在`~/.config/nvim/lua/community.lua`中添加：

```lua
{ import "astrocommunity.completion.copilot-lua-cmp" }
```

启用Copilot需要运行`:copilot auth`进行认证。

## 语言服务器和调试支持

AstroNvim通过Mason插件管理多种语言服务器和调试工具。例如，安装C++的`clangd`服务器：

```vim
:LspInstall clangd
```

对于调试支持，例如安装C/C++的`codelldb`：

```vim
:DapInstall codelldb
```

可用的语言服务器和调试服务器可以通过`Mason`命令查看：

```vim
:Mason
```

### 调试快捷键

AstroNvim的调试快捷键配置详见官网Mappings部分。这些快捷键包括启动、暂停、重启调试器，设置断点等功能。这里也贴上：

| Action                  | Mappings |
| ------------------------- | ---------- |
| Start/Continue Debugger | `Leader + dc` or `<F5>`     |
| Pause Debugger          | `Leader + dp` or `<F6>`     |
| Restart Debugger        | `Leader + dr` or `<C-F5>`     |
| Run Debugger to Cursor  | `Leader + ds`         |
| Close Debugger Session  | `Leader + dq`         |
| Terminate Debugger      | `Leader + dQ` or `<S-F5>`     |
| Toggle Breakpoint       | `Leader + db` or `<F9>`     |
| Conditional Breakpoint  | `Leader + dC` or `<S-F9>`     |
| Clear Breakpoints       | `Leader + dB`         |
| Step Over               | `Leader + do` or `<F10>`     |
| Step Into               | `Leader + di` or `<F11>`     |
| Step Out                | `Leader + dO` or `<S-F11>`     |
| Evaluate Expression     | `Leader + dE`         |
| Toggle REPL             | `Leader + dR`         |
| Toggle Debugger UI      | `Leader + du`         |
| Debugger Hover          | `Leader + dh`         |

## 常见问题

### treesitter

如果频繁地报连接问题，尝试在`~/.config/nvim/init.lua`下加上：

```lua
require("nvim-treesitter.install").prefer_git = true
```

参考：https://github.com/nvim-treesitter/nvim-treesitter/issues/3232

## cland

如果出现不停的warning信息：`Multiple different client offset_encodings detected`。

尝试做以下处理：

修改`~/.local/share/nvim/lazy/nvim-lspconfig/lua/lspconfig/server_configurations/clangd.lua`。

```lua
local default_capabilities = {
  textDocument = {
    completion = {
      editsNearCursor = true,
    },
  },
  -- offsetEncoding = { 'utf-8', 'utf-16' },
  offsetEncoding = "utf-16",
}
```
