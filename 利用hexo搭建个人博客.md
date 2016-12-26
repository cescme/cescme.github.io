---
title: 利用hexo搭建个人博客
date: 2016-12-10 15:34:21
categories: 工具
tags: [hexo,博客]
---

昨天利用hexo、github成功搭建了个人博客，将之前写的md格式的文章发布，感觉效果不错。这里简单记录下整个搭建过程，如果以后换电脑的话做个参考，如果大家看到了也可以顺着步骤一起实践，体验一下现下流行的基于github构建静态博客的酷炫过程。

1. 安装
- 首先安装node.js，这个比较简单
- 打开git bash命令行
- 由于自带npm不稳定，建议安装淘宝npm镜像
```bash
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
```
- 安装hexo
```bash
$ cnpm install hexo-cli -g
```
这时候hexo会默认安装在c盘，大家可以自行调整安装路径

2. hexo配置
- 初始化，新建一个blog目录
```bash
$ mkdir blog
$ cd blog
$ hexo init
```
- 新建文章test
```bash
$ hexo new 'test'
```
- 预览
```bash
$ hexo server
```
- 打开浏览器，进入地址localhost:4000，即可预览整体博客和test文章

3. github配置
- 新建一个仓库，如我们的用户名是abc，则仓库名称必须为abc.github.io
- 到仓库中的settings--github pages--sources，将master分支作为博客的提供分支，并保存
- 打开blog目录的_config.yml，配置github地址
```
deploy:
  type: git
  repository: git@github.com:abc/abc.github.io.git # abc为github用户名
  branch: master
```

4. 内容发布
- 发布前首先清理
```bash
$ hexo clean
```
- 生成文章。public目录能看到生成的前端代码
```bash
$ hexo generate
```
- 发布，将代码推送到github远程库的master分支
```bash
$ hexo deploy
```
此时，查看github，确认代码已提交，打开
https://abc.github.io
即可看到hexo博客。

*附*：使用maupassant主题

hexo有很多开源的主题，大家可以自行安转并切换配置

maupassant是一款极简主义的主题，并不酷炫，但简约朴实，一下就抓住我的心。现将其安装过程记录下来

- 安装python2.7
- 切换到blog目录
```bash
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
$ cnpm install hexo-renderer-jade --save
$ cnpm install hexo-renderer-sass --save
```
- 主题配置，修改blog/_config.yml文件
```
theme: maupassant
```
- blog/themes/maupassant/_config.yml配置文件中提供主题的相关配置
