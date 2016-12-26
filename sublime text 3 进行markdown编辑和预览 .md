---
title: sublime text 3 进行markdown编辑和预览
date: 2016-11-30 20:33:34
categories: 工具
tags: [sublime,markdown]
---

### 安装package controll
安装好sublime3后，首先需安装package controll，这是进行其他插件安装的前提

#### 自动安装
使用Ctrl+`快捷键或者通过View->Show Console菜单打开命令行，粘贴如下代码：

    import urllib.request,os; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); open(os.path.join(ipp, pf), 'wb').write(urllib.request.urlopen( 'http://sublime.wbond.net/' + pf.replace(' ','%20')).read())

#### 手动安装
- 点击Preferences > Browse Packages菜单
- 进入打开的目录的上层目录，然后再进入Installed Packages/目录
- 下载Package Control.sublime-package并复制到Installed Packages/目录
  下载地址：https://sublime.wbond.net/Package%20Control.sublime-package
- 重启Sublime Text。

### 安装markdown editing和markdown preview
按Ctrl + Shift + P，分别输入markdown editing和markdown preview
点击对应插件项进行下载安装
重启后生效

### 设置markdown语法高亮
按Ctrl + Shift + P使用Markdown语法编辑文档语法高亮，输入ssm 后回车(Set Syntax: Markdown)

### 预览快捷键设置
直接在浏览器中预览效果的话，可以自定义快捷键：点击 Preferences --> 选择 Key Bindings User，输入：
- 普通预览

    { "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} },

- 在github中预览

    { "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"github"} },

保存后，直接输入快捷键：Alt + M 就可以直接在浏览器中预览生成的HTML文件了。

以上为sublime支持markdown编辑的初始化过程，后续学习心得将继续整理
