layout: '[layout]'
title: 搭建基于 Hexo + GitHub Page 的博客
date: 2016-04-09 22:12:24
desc: 搭建博客教程
---

## 前言
本文记录了博客搭建的过程，以及一些需要注意的事项。

如果要顺利的搭建完成，最好先了解一下 Hexo 和 Github Pages 是什么，它们的功能分别是什么，理解其中的基本原理之后操作起来会简单很多。

<!-- more -->

[Hexo](https://hexo.io) 它是一个快速、简洁且高效的博客框架。可以理解为静态网页生成器，只要你提供 [Markdown](http://wowubuntu.com/markdown/) 格式的博文，然后执行几行简单的配置和生成命令，它就会用把博文生成网页，然后通过本地预览就可以看到自己的博客了。

[GitHub Pages](https://pages.github.com) 免费保存博客的站点，其实就是个 Git 仓库。你当然不希望自己的博客只能在本地自己预览，所以你需要一个服务器平台。GitHub Pages 可以理解为博客的服务器，用于托管博客的静态网页。

现在基本的知识已经了解了，但还是想多说两句。虽然基于 Hexo 搭建博客算是简单的一种方法了，但是在搭建过程肯定不可能一帆风顺，每个人会遇到的问题肯定也不会一样。我只能尽可能记录一些我遇到的一些情况和需要注意的细节。只要遇到问题坚持多查阅，胜利肯定是在望的！

> 注意：该教程安装环境为 Mac OS X。

## 安装前提

安装 Hexo 前，需要先安装两个工具：

- [Git](https://git-scm.com)
- [Node.js](https://nodejs.org/en/)

#### 安装 Git
Git 有多种安装方法，最便捷的一种是使用 [homebrew](http://brew.sh) 进行安装。网上教程有很多，所以这里不再累赘。

#### 安装 Node.js
安装 Node.js 最普遍的方式是使用 [nvm](https://github.com/creationix/nvm). 所以得先安装 nvm。

安装 nvm 有两种方式，选其一即可(以下命令建议在用户的根目录下执行):
cURL:

```bash
$ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
```

或者Wget：

```bash
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
```

nvm 安装完成后，执行安装 Node.js 命令：

```bash
$ nvm install 5.0
```

**注意：** 如果执行`nvm`的时候出现`nvm: command not found`提示。请先使用`ls -a`命令查看根目录下是否存在[.bash_profile file]文件。如果不存在使用`touch ~/.bash_profile`创建，并重新执行如上安装命令。

如若还是不行，请尝试执行:

```bash
$ source ~/.nvm/nvm.sh
```

** nvm 安装完成之后，请务必记得重启终端（右键->退出）。**

#### 安装Hexo
Node.js 和 Git 都安装完成之后，执行如下命令安装 Hexo

```bash
$ npm install -g hexo-cli
```

接着需要在本地准备一个存放静态网页的文件夹了，我将文件夹命名为 Hexo，然后`cd` 到这个目录下。执行 Hexo 初始化命令：

```bash
$ hexo init
$ npm install -g
```

初始化完成之后，该文件下的目录如下所示：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   └── _posts
└── themes
```

这里的 `source/_posts` 目录就是我们存放 Markdown 格式的博文的地方，默认已经存在一篇 `hello-world.md` 文件。

我们尝试执行网页生成命令并开启本地服务：

```bash
$ hexo generate 
$ hexo server    
```

执行如上命令有可能会报错，请自行将错误粘贴到 Google.
如果顺利启动的话，现在可以成功访问 [http://localhost:4000](http://localhost:4000)了。

如果在本地成功访问到了博客页面，不过现在上面没有一点你的信息，还需要做写简单的配置。这时位于 Hexo 目录下的配置主角 `_config.yml` 该登场了。打开该文件先修改一些主要的信息。

```yml
title: iOSTalk              #网站标题
subtitle: 但行好事 莫问前程    #网站副标题
description: 记录学习的点滴    #网站描述
author: xfjaing             #网站作者
email: iostalkboy@gmail.com #联系邮箱
language: zh-CN             #语言```

注意标题与内容之间要有一个空格。

现在重复执行以下以上的两条命令，博客里就有你的信息啦。


## 部署到 GitHub Pages

在本地可以成功访问后，还需要将其部署到 GitHub 上别人才能访问到。

在自己的 GitHub 中新建一仓库，名为: [iostalks.github.io](http://iostalks.github.io)

> 仓库名格式最好为 `username.github.io`，`username`为你的 GitHub 用户名。

具体操作请戳[这里](https://pages.github.com).

仓库创建好之后，需要将 Hexo 链接过去，这样才能在发布博文的时候，将内容同步到 Github 上。这里需要通过配置 `_config.yml` 文件来完成，将该文件的最后几行修改如下（注意中间的空格）：

```yml
deploy:
  type: git
  repository: https://github.com/iostalks/iostalks.github.io.git
  branch: master```

保存后，执行如下部署命令：

```bash
$ npm install hexo-deployer-git --save
$ hexo deploy # 执行该命令会自动的将 Hexo 目录下的所有博客资源 push 到刚刚创建的 git 仓库中 
```

现在在浏览器访问 http://iostalks.github.io 就可以看到刚刚在本地预览的页面了。

> 注意将`iostalks`替换成你自己的 GitHub 用户名。

 
## 主题美化
这里有官方推荐的几个[主题](https://hexo.io/themes/)：

每个主题在 github 上都有一个仓库，这里以 [Apollo](https://github.com/pinggod/hexo-theme-apollo) 主题为例，介绍一下配置流程：

在 Hexo 目录中将主题源文件下载到本地 themes 文件夹下

```bash	
$ git clone https://github.com/pinggod/hexo-theme-apollo.git
```

配置 Hexo 目录下的 `_config.yml` 文件里的 `theme` 属性

```yml
theme: apollo
```

重新执行一次一次页面页面生成命令：

```bash
$ hexo generate
$ hexo server    #启动本地服务
```

此时访问 [http://localhost:4000](http://localhost:4000) 可能并不会得到理想的主题效果，因为有些主题需要插件的支持，所以这里还需要执行一下该主题插件安装命令：

```bash
$ npm install --save hexo-renderer-jade hexo-generator-feed hexo-generator-sitemap hexo-browsersync hexo-generator-archive
```

现在执行 `hexo server`再去本地预览看看有没有啦。

每个[主题](https://hexo.io/themes/)都会有对应的安装介绍的，喜欢哪个个点蓝色主题名称，进入该主题的 github 仓库，会有对应的安装介绍。

当看到自己的喜欢的主题在本地徐徐升起，兴奋之余点了一下`Weibo`，发现跳到别人家的微博去了。是的，这是该主题作者的微博，是不是该跟人家道个谢再走？之后我想设置成自己的微博怎么办？很简单，进入该主题的本地仓库，里面也有一个`_config.yml`，打开就全明白了。


## 写博客
```bash
$ hexo new "ArticleName" #新建文章，会在 source/_posts 目录下生成 .md 文件
	.....
$ hexo generate #生成静态页面至public目录
$ hexo server   #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
$ hexo deploy   #将.deploy目录部署到GitHub
```

以上命令都可以使用首字符简写，比如：

```bash
$ hexo d #将.deploy目录部署到GitHub
```

## 自定义域名

如果你还想让自己的博客更加的个性化，可以考虑购买一个自己的域名。

推荐使用国内的 [NDSPod](https://www.dnspod.cn)，你也可以选择国外的 [Godaddy](https://www.godaddy.com/?isc=cjcdplink&iphoneview=1&cvosrc=affiliate.cj.6162165)，功能上没有什么区别，我选择的是 NDSPOd，购买过程几分钟就搞定了，接着需要做一些域名的配置。在域名解析栏里面添加一条 CNAME 记录，将 GitHub 地址映射到这个域名上。添加完成之后像这样(第一条记录)：

![160409_01](http://7xsto7.com2.z0.glb.clouddn.com/160409_01.png)

我申请的域名为 iostalk.com，并为博客设置了一个二级域名。此时访问 blog.iostalks.com 会得到一个`404`。这里还需要做一些配置。在终端`cd`到 `Hexo/source` 目录下，执行如下命令创建一个 `CNAME` 文件。

```bash
$ vi CNAME
```
在里面写入刚刚添加的域名

```
blog.iostalks.com
```

`:wq`保存文件后执行：

```bash
$ hexo generate
$ hexo deploy 
```

现在打开浏览器输入你的域名就可以成功的访问啦！

## 最后

这是我博客的完整代码，可以参考[这里](https://github.com/iostalks/hexo-archive.git)。

感谢您的阅读，有问题欢迎在评论区与我交流。

推荐阅读：
[hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)
[基于 Hexo 和 GitHub Pages 搭建博客](http://col.dog/2015/11/12/hello-world/)