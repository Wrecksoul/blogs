---
title: 基于github-pages使用hexo搭建个人博客
categories:
- 杂谈
tags:
- hexo
- github
---

如果你在写博客的时候有以下困扰，可以试着自己搭建一个个人博客：

- 各个网站的博客系统需要审核才能发布，有延迟，而且时间不一定；
- 博客的样子不够个性；
- 博客的栏目不是你想要的；
- 博客中有各种无关广告和弹窗。

由于我是一个后端开发人员，对前端技术并不了解，在搭建博客的过程中难免有许多不足之处，但是如果你也是后台开发并且对前端技术并不熟悉的情况下，我倒是觉得看这篇博文可能会更有感觉一些，我会从后台开发的角度解释一些步骤的原因，从而方便你具体问题具体分析。
<!--more-->

# 知识普及
## 个人博客

如果你在csdn之类的网站上写过博客，那你在构建自己的个人博客过程中可能会产生一大堆的疑问——不知道我在干什么，所以在这之前我打算花一点文字来描述一下搭建个人博客的大体流程和大致原理。

一般来说，一个网页，不管它提供的功能多还是少，只要放在互联网上人人都能访问，那么它就需要寄托在一个服务器上，就像淘宝、百度、新浪都有自己的服务器一样。当你在浏览器里面敲下他们网站的地址的时候，其实你访问的是互联网另一端的一台电脑，并且它给了你一个网页作为反馈。

对于像csdn这种博客管理系统来说，当我们注册了账号之后就拥有了自己的一个博客网址，目录、标签等等都是csdn定制好的。我们可以在专门的编辑页面提交文章，然后等csdn审核通过后就可以在互联网上看到我们自己的博客了。整个流程简单易行，总体来说还是比较方便的，但是当我们需要一些更加个性的样式的时候可能这些就满足不了我们的需求了。

对于基于github的pages功能来搭建博客，总体流程比使用一个现成的博客管理系统要麻烦一些。

一篇博文从我们编辑的.md文件到最后上传到我们的博客页面上，大体上分几步来完成：

1. 编辑`.md`文件，在里面写你的博文
2. `.md`文件被转换成`html`（这是我们使用的是hexo而不是github原生支持的jekyll，hexo根据我们的`.md`文件生成对应样式的`html`片段，并把它嵌入到hexo自定义的主题模板中去）。
3. 生成的完整的静态页面（可能是一堆）通过git上传到github你的repository中去。
4. 通过浏览器访问。

*注意，在上面的几个步骤中，第一步和最后一步肯定是人工完成的（废话），第二、三步由我们发出一行命令行指令后hexo替我们完成。*

## 编辑环境说明
根据上面的说明，可能你已经看出来了，我们的博文开始是`.md`文件最后上传的`html`文件，这里是有一个转换过程的，同时，这个跟java一样是有编译前的`.java`文件和最后用于执行的`.class`文件之分的，而最后有用的是html文件，但是源文件（`.md`)文件以及它所依赖的环境我们也是不想丢弃的，除非你只发一次博文并且再也不修改了。

所以，在使用github托管我们的文件的时候我们希望保持两部分内容，而这两部分内容，我们恰好可以使用git的分支来分别保持。
### 说明
由于我个人的一些原因，我要把源码也上传到github上，所以才会使用两个分支分别保持，这是因为我可能会使用不同的计算机来写博文，而我又不想把`源码环境`拿来拿去，所以我选择托管到github上，当然你也可以选择托管到码云之类的其他代码托管（因为貌似他们提供了免费的私人代码托管服务）或者使用百度云、U盘来保持你写博客环境的可移动性，或者你根本不在乎这些，那你就可以具体问题具体分析一下了。


# 搭建
## 环境说明
我是在window环境搭建，当然搭建和写博客本身跟平台是无关的，但是搭建或者使用过程中软件版本会有影响，比如Git、nodejs都对不同平台提供了不同版本并且在设置会有写不同。

## 环境准备
1. 一台电脑安装了git的pc
2. 一个github账号

## 前期准备
有了github账号并且pc上正确安装了git（我安装的Git for windows）之后，由于git需要往github上提交代码，如果每次都需要我们输入密码认证的话是很麻烦的，所以我们需要给本地的git设置ssh登录方式。
### 配置git以ssh方式登录github
这里我只是给出简单流程，如果有疑问可以自行搜索。

一、生成ssh私钥和公钥

随便打开一个文件夹右键-》Git Bash here，*其中`xxx@yy.com`替换成你github注册的邮箱地址*：


```
$ cd ~
$ ssh-keygen -t rsa -C "xxx@yy.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/xxxx_000/.ssh/id_rsa):#这里使用默认路径，留空，直接回城
Enter passphrase (empty for no passphrase):   #输入密码（可以为空）
Enter same passphrase again:   #再次确认密码（可以为空）
Your identification has been saved in /c/Users/xxxx_000/.ssh/id_rsa.   #生成的密钥
Your public key has been saved in /c/Users/xxxx_000/.ssh/id_rsa.pub.  #生成的公钥
The key fingerprint is:
e3:51:33:xx:xx:xx:xx:xxx:61:28:83:e2:81 xxxxxx@yy.com
```
以上完成后就生成了私钥和公钥，他们存储在默认路径下：`C:\Users\你的用户名\.ssh\`下，其中id_rsa.pub是公钥。

二、把公钥复制到github的`SSH keys`中
将公钥用文本编辑器打开复制里面的内容到github中去

![](http://ozhp30d7a.bkt.clouddn.com/image/%E5%9F%BA%E4%BA%8Egithub-pages%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/github1.png)

![](http://ozhp30d7a.bkt.clouddn.com/image/%E5%9F%BA%E4%BA%8Egithub-pages%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/github2.png)

三、配置git连接github时候的name和email
```
$ git config --global user.name "xxxxxx"

$ git config --global user.email "xxxxxx@yy.com"
```
四、测试
```
$ ssh -T git@github.com
#如果显示如下信息，说明配置正确了：
Hi xxxxxx! You've successfully authenticated, but GitHub does not provide shell access.
```

### 安装node.js
[nodejs中文网](http://nodejs.cn/)

## 开始搭建

### 第一步 建立github仓库
新建一个github仓库（repository），这里可以分两种情况，这是我自己在搭建时候遇到的困惑，

1. 用username.github.io命名仓库。例如我的就是`wrecksoul.github.io`
2. 用一个任意的名字。例如我的就是blogs

这两种都是可以的，但是各有各的坑，具体情况下后面的“具体说明”。

新建的时候，只添加一个readme就可以了，.gitignore是不需要的，因为待会安装hexo，hexo自带。



#### 具体说明
1、
github在博客这上面的设置是这样的，如果你的仓库使用`username.github.io`这种名称的话，那么就只能发布master分支作为博客静态页面
那么：
![](http://ozhp30d7a.bkt.clouddn.com/image/%E5%9F%BA%E4%BA%8Egithub-pages%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/github3.png)
这个地方的设置就会变灰，无法更改，只能是master分支。
不过这个也不算什么缺点了，但是貌似如果你使用jekyll的话会有坑。用hexo应该是没有问题的。
不过这个算是一个新手不知道的话会踩到的坑，比如我。
2、
最大的区别在于，如果你的博客使用`username.github.io`这个根url，这时候，就好像你设置servlet的拦截路径为 `/` 一样，所有请求都被它拦截了，某天你开发了一个开源项目，发布到github上面，你想给这个开源项目一个首页来介绍它，这就尴尬了。

而如果你使用`username.github.io/blog`这个url（也就是你的github仓库叫做blog），那你还可以建立`username.github.io/blog1`、`username.github.io/blog2`等等。

### 第二步 搭建hexo环境

node.js有个官方支持的npm工具，可以方便的管理各种依赖module，类似于java的`maven/gradle`这种工具，不过它只用于管理依赖，说起来可能更像是centos的包管理工具`rpm/yum`

不同的在于nodejs在包管理时，会为每个工程都下载一个依赖包在当前项目路径下，也就是说，如果你想在本机重新搭建一个hexo环境的话，需要重新从网上下载一个下来，而不是利用本机已有的。
#### 克隆github仓库到本地
克隆下来之后，进入到下载后的文件夹中，比如我的项目名叫做blogs，就cd blogs
```
$ git clone [your url]
$ cd [your repository's name]
```

#### 安装hexo


剪切走之后：
```
$ npm install hexo --save
```
等待一会就装好了。

由于我们希望使用git方式提交到github，所以还需要下载hexo往git上提交的一个模块
```
$ npm install hexo-deployer-git
```

在初始化hexo之前，首先进入到刚刚Git克隆下来的目录里面，把里面的隐藏文件`.git`文件剪切到其他地方去备用，（原因：hexo在初始化的时候需要一个没有`.git`文件的文件夹，他本身在init过程中会生成一个`.git`文件，最后再删除掉。)

执行：
```
$ hexo init
```

安装完成后：
![hexo目录结构](http://ozhp30d7a.bkt.clouddn.com/image/%E5%9F%BA%E4%BA%8Egithub-pages%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/github4.png)

可能\*.log \*.json的文件会没有，这些是hexo自动生成的，在你没启动服务之前没有这些文件很正常，而且这些文件会被`.gitignore`文件忽略并不会纳入git版本管理中。

到此为止，环境就搭建好了。可以进行本地测试了。

### 第三步 测试hexo环境

在博客的文件夹内部打开命令行工具，可以是cmd/powershell/git bash，执行：
```
$ hexo s -p [your port] #比如我的是hexo s -p 5000
```
*其实这里可以不加-p 和后面的5000来指定端口，但是我本地出现了端口冲突，特征是发布成功，命令行显示正常，但是浏览器访问时候一直转圈圈无法获得反馈。搞了半天才弄好，因为java平台下，服务器如果使用的端口被占用根本无法正常启动，并且会抛出addr blind之类的异常信息。*

命令行反馈：
![hexo测试](http://ozhp30d7a.bkt.clouddn.com/image/%E5%9F%BA%E4%BA%8Egithub-pages%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/github5.png)

此时使用浏览器访问：`http://localhost:5000/blogs/` 就可以了。

此时如果现实正常的博客页面就说明可以了。

### 第四步 更换主题

搭建好环境之后要做的第一件事不是写博客，而是更换hexo的theme，我选择的是next这个主题:

[Hexo-Theme-Next](http://theme-next.iissnan.com/)

使用方式非常简单，并且next提供了官方的中文版本的文档，这里我就不详细说了，具体可以看文档。

# 平时的使用方式

这里只是给出比较简要的流程，方便你快速入门，具体详情请看官方文档：[Hexo 中文文档](https://hexo.io/zh-cn/docs/)

### 写博客
写博客是可以使用命令行指令
```
$ hexo new "my post title"
```
那么hexo会自动建立一个my-post-title.md文件，它位于source/_post目录下，然后你就可以修改它进行写作了。

编写.md文件有很多种工具，我使用的是`MarkdownPad2`，不过经试验这个工具在win10系统会出现崩溃的bug。目前我也在重新寻找更好用的工具。

### 查看效果

当你完成初次编写，处理在`.md`编辑工具中可以查看文档的样子，你还可以让hexo生成静态页面，然后通过浏览器进行本地预览，**注意，只是本地预览，跟前面说过的`hexo s -p 5000`指令相似，你可以使用`hexo s -p 5000 --debug`指令，这样你可以随时查看修改的情况而无需重启服务器，当然记得清楚你浏览器的缓存哦。**

### 提交到github

当你觉得文章写好了，可以发布了,你需要做两件事
1、保证git提交相关配置正确（基本上每台电脑只需要配置一次）
2、执行提交指令
```
$ hexo clean #注意这个指令并不是必须的，但是它可以很好的保证你提交的内容一定是最新的，而非缓存内容
$ hexo d
```
#### 配置git提交
hexo 支持多种提交方式，而我只使用git这一种。

编辑你项目根目录下的_config.yml,找到并修改如下内容
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/[your-path]/[your-repo-name].git
  branch: gh-pages
  message:
```

如你所见，配置文件的注释部分给出了[配置文档](https://hexo.io/docs/deployment.html)的地址。详情还是看文档吧。

#### 提交源码*

*就像我在前面说过的，如果你不需要保持写博客的环境的移动性这一步请忽略。*

这里需要一点Git的知识，如果没有接触过，可以看这里：[Git 圣经——Pro Git 中文版](https://git-scm.com/book/zh/v2)

1. 进入项目根目录，我的是blogs
2. 右键鼠标，Git Bash Here
3. `$ git add .` 添加工作目录修改到暂存区，请具体情况具体分析
4. `$ git commit -m "你提交的内容说明"` 提交源码到本地仓库
5. `$ git push` 发送到远程仓库

# 更换电脑时候的初始化步骤

*就像我在前面说过的，如果你不需要保持写博客的环境的移动性这一步请忽略。*

1. 安装好nodejs 和 Git
2. 配置Git 登录Github的 SSH 认证方式
3. `$ git clone [url]`把工程克隆到本地
4. `cd blogs` 进入目录后
5. `$ npm install hexo --save`安装hexo
6. `$ npm install hexo-deployer-git`
7. 开始编写吧

{% note danger %}一定不要执行`hexo init指令`，上面执行的指令目的是安装好hexo需要的依赖，并不是要重复搭建环境时的步骤{% endnote %}