---
title: hexo 博客目录介绍
date: 2019-04-26 16:24:21
tags:
---

摘抄自：http://qiyujian.com/2017/09/08/20170908/

### Hexo工作流程

有2种方式可以了解hexo的工作流程. 一是去官网直接读document或者API, 二是自己一边动手配置一边搜索或查看文档. 我推荐的过程是, 先到官网大致浏览一下tutorial, 然后找个demo自己动手配置, 遇到问题再搜索或者查阅官网文档. 这样既能在起步之初对于hexo整体有个大概了解, 又可以边动手边查阅.

[这篇文章](http://coderunthings.com/2017/08/20/howhexoworks/)不纠缠细节, 从总体给出了hexo的工作流程, 感觉比较清晰. 如果借助MVC的概念来理解hexo的话:

- hexo项目路径下的source路径相当于M, 即数据. 对于博客系统来说, 以.md文件形式存储的博文就是hexo的数据.
- themes路径下的模版相当于V, 即视图. hexo会将source中的数据(博文或其他)填到theme路径下的模版中.
- themes路径下的js或其他代码相当于C, 即控制. 会在静态网页中进行一些用户交互响应.

这样的类比理解肯定有不妥的地方, 后续如果有新的理解会再来修改更正.

还有一些其他的路径:

- public
  是hexo生成的静态网页的全部, 而且包括js和css, 图片资源等. 这些会随着 hexo deploy 命令推送到github项目的master分支上. Github Pages就是解析这些文件, 呈现出静态页面的博客来.
- node_modules
  路径下是hexo安装的一些插件. 安装的插件可以在package.json中配置. 如果是用npm进行安装, 建议加上*–save*参数, 这样该插件的信息就会被保存到package.json中, 下次只需要运行*npm install*即可. 这对多地更新博客, 或者重新搭建本地博客编写环境会非常方便.

另外, hexo项目根目录下会有一个_config.yml, 在themes/some_theme/ 路径下也会有一个_config.yml, 前者是整个hexo项目的配置, 后者是某个theme的配置.

所以, 总体描述hexo的过程就是:

1. *hexo generate* 命令会将source路径下的数据根据配置信息填充到themes路径下的模版中, 生成的静态网页需要的文件存放在public路径下;
2. *hexo deploy* 会将public路径下的文件上传到Github该项目的master分支;
3. Github Pages解析master分支里的静态网页文件.
   这样当我们访问时, 就会看到博客内容了. 当然, hexo工作过程中会调用一些列依赖的插件. 在网上都很容易找到某些功能需要的插件和安装方法.

### 如何实现多地更新博客

创建的博客这个repository本身就是托管在github上的, 在运行 *hexo deploy* 命令提交成功后, 也会看到已经提交到master分支的输出结果. 那么, 就可以利用github来进行博客备份了. 大概思路就是, 给博客repository再拉个名为hexo的分支(当然名字是随便的), 平时的改动, 都用 *git clone / pull/ add / commit / push* 等命令来进行管理, 就当是代码版本那样管理. 在本地编写好博客文章, 想要布置到github时, 使用命令 *hexo deploy* , 这样还是会推送到master分支. 也就是说, hexo相关的操作与之前完全一样, 没有变化. 知乎的这个[问题](https://www.zhihu.com/question/21193762)下的几个答案挺有帮助的, 可以参考. 几个小的注意的地方是:

1. 在浏览器页面建立新的分支hexo后, 在repository的settings中, 将hexo设置为默认分支, 这样clone到本地后, 每次git管理不用在切换分支了, 主要是因为*hexo deploy* 命令是默认推送到master的.
2. 根据自己情况配置.gitignore文件, 其中public路径和node_modules路径都不需要同步到github. node_modules路径下是存放下载的hexo插件, 而且体积很大. 但是注意, package.json一定要同步, 这样在搭建新环境时, 安装完NodeJS和Git后, 只需要*npm install* 即可, 还是很方便的. 上面提到的知乎的[问题](https://www.zhihu.com/question/21193762)下面有一个答案详细介绍了哪些需要同步, 哪些不必同步.