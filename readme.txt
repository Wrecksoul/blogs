需要在一台新机器上写博客，并且发布时，需要进行的操作。

0、前提说明，博客已经按照2分支的方式管理：master主干 博客源代码文件。gh-pages分支 博客静态页
1、准备环境，nodejs安装好。git安装好。
2、git clone [url] 把项目克隆到本地。

2.9、对于下面两个步骤，一定不要只想hexo init指令。只有初次创建博客的时候才需要这样。
3、默认clone完成应该是在master主干分支下，安装hexo：npm install hexo --save
4、安装hexo提交git的module ：npm install hexo-deployer-git（3,4两个步骤其实就是按照hexo相关的东西，可以hexo s -p 5000 测试一下是否可以启动本地的博客系统，可以就ok了，不可以它会提示缺少什么，那你就装或者改就行了。）
5、完成本地搭建工作。

http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more

这个地址有更加详细的讲解