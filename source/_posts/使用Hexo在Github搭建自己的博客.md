---
title: 使用Hexo在Github搭建自己的博客
date: 2018-08-13 14:00:39
tags:
---

# 使用Hexo在Github搭建自己的博客

## hexo本地部署

hexo是一款基于Node.js的静态博客框架

主页: https://hexo.io/zh-cn/

部署过程：

由于是基于Node.js 故需要先安装node

node安装完毕后 打开命令行窗口 运行下列命令（在windows下，推荐使用git bash）

```bash
- npm install hexo-cli -g
- hexo init blog
- cd blog
- npm install
- hexo server #启动本地后台 此时可通过浏览器访问 
#这个命令可以指定端口号 加入电脑的4000端口被占用会报错
#浏览器输入http://localhost:4000
```

  ​

## hexo常用命令
```bash
- hexo new"postName" #新建文章
- hexo new page"pageName" #新建页面
- hexo generate #生成静态页面至public目录
- hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
- hexo deploy #将.deploy目录部署到GitHub
- hexo help # 查看帮助
- hexo version #查看Hexo的版本
```


## hexo目录内容

- node_modules：是依赖包
- public：存放的是生成的页面
- scaffolds：命令生成文章等的模板
- source：用命令创建的各种文章
- themes：主题
- _config.yml：整个博客的配置
- db.json：source解析所得到的
- package.json：项目所需模块项目的配置信息



## 什么是Github Page

​    在Github上创建个人主页非常方便，只要创建一个名为(user-id).github.io的版本库， 并将自己编写的网页文件推送到master分支即可。最重要的优点是支持Markdown自动渲染。Github提供个人page和项目page，我们这个说的是个人page，也可以为每个项目创建一个page。

## 部署到github

在github新建版本库 hebinglong.github.io（名字必须是这个 不能变）

github会自动将该项目作为个人主页进行渲染，可以通过该项目名称直接从外网访问（即可在浏览器上直接通过hebinglong.github.io 进行访问）。注意只支持静态网页（可以理解为纯文本吧，但是可以支持外链图片等）。

## 工作流

使用hexo deploy 进行部署的时候 只会将生成的静态网页（即目录下的public目录）部署到github上，也就是说，源码并没有进行版本管理，假如不小心删除了只有渲染后的html文件，而原本的markdown文件就不见了。所以建议在该项目下，新建一个分支（如src），用来对所有源码的管理。同时将该项目的默认分支设置为src，使用git commit 和git push时自动将源码部署到src分支中去，而在hexo deploy中，配置默认分支为master。

这样每次写完新的博客后，使用hexo d部署master 再用 git commit 和git push备份源码。