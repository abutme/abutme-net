title: hexo静态博客建设
date: 2014-02-18 15:58:19
tags: 
- hexo
- markdown
category: blog
---

在看了一些程序员的博客后，觉得这个事情很有意义。博客建设是个长期的事情，就从此开始吧。在[Jim Liu's Blog][1]初步了解了hexo博客的搭建方法后又搜索了其他相关的分享，最后认定就是[hexo][2]了。

`hexo`的优点是自带轻量级的web server，跟`github`整合的不错。整个环境搭建用了些时间，最后用起来还是很顺畅的。
环境搭建的时间主要是花在github部署打通和linux安装node的尝试。

下面我记录一下我在windows下的搭建过程，linux搭建主要解决node的安装问题就好办了。`node`对gcc的版本以及依赖要求很高，还好最后用我厂的包安装神器顺利解决。

<!-- more -->

## windows用户

顺便说一下，本人机器是win7 x86

*1. 安装git客户端和node.js*

git是windows 8风格的，很简约，已经够用了。
安装过程表明已经自动注册环境变量了，可以在命令行用npm命令。


*2. 安装hexo*

	$ npm install -g hexo
	
在命令行可以运行hexo命令就表示OK了。


*3. hexo试用*

新建一个文件夹，在命令行下cd到该目录，执行一下：

	$ hexo init

执行结果是会在当前目录下自动生成项目的基础框架和文件。再执行一下：

	$ hexo server

启动web server，提示`4000`端口打开。

在浏览器敲入[http://localhost:4000](http://localhost:4000)
就可以看到hexo默认提供的主题和Hello World了。

本人不喜欢默认主题，于是我从[Jim Liu's Git Repo][3]中把light主题搞下来了，这个主题改的不错。加到项目需要修改theme配置，同时需要修改一个css的问题，否则发文章后标题会没有icon font图标。

*4. 配置修改*

配置修改的关键是`_config.yaml`文件。
在项目根目录和具体主题目录下都用这个配置文件，具体配置方法可以试出来。
需要注意的是，配置修改需要停启server，而文章的修改是刷新生效的。

*5. 组件添加*

本人只添加了评论组件[多说][4]。进入duoshuo的创建站点功能，填入站点信息，拿到一段js通用代码。duoshuo提供评论的后台管理功能。

把通用代码命名为`duoshuo.ejs`放到对应主题的`layout/_partial/`目录下，同时在需要引用duoshuo的主题布局模板（如：layout目录下的post.ejs）中加入引用的代码。具体参考本人后面附上的Git Repo源文件。


*6. 域名绑定*

域名绑定先仔细阅读[github的官方说明][6]，其提供两种页面发布方式：用户/组织页和项目页。

前者，一个特殊的项目，需要将发布的页面部署在`username/username.github.io`的主干。后者，则需将发布页面部署在该项目`username/projectname`的`gh-pages`分支。

有人是将博客源代码部署在项目的主干，发布后的代码部署到gh-pages分支，这样维护起来只需要一个项目。
本人是新建了一个git项目用来维护源码，发布页面部署到`abutme/abutme.github.io`的主干。

github的发布方式已经提供了一个外网访问域名：`username.github.io`；项目页面的域名：`username.github.io/projectname`。这两种访问都路由到对应的index.html。

本人提前在[万网][5]申请了个人域名，在域名管理后台，在域名解析中增加A记录指向git的ip，数分钟后生效。

*7. github账号和信任关系*

没有账号就需要注册github账号。

信任关系的建立参考github官方说明。
流程就是在本机生成`SSH Key`，将它添加到git账号的`SSH Keys`设置里。
这样每次连接github的时候，不需要填用户名和密码，为后续的自动deploy做准备。

确认信任关系的方法是：

	$ ssh -T git@github.com
	
如果出现欢迎信息（Hi *username* ...）就表示OK了。

*8. hexo自动部署*

以上工作完成后，需要修改hexo项目的配置文件。将deploy部分配置为：
```
deploy: 
    type: github
    repository: https://github.com/abutme/abutme.github.io.git
```
个人理解，hexo的自动发布也是会按照github的发布规则来自动识别是发布到主干还是gh-pages分支的。

*9. 文章编辑和维护*

hexo的操作可以参考hexo的官方说明。

本人是将github源码clone到本地，在该项目里进行hexo的操作，这样修改源码和本地部署就一体了。
确认无误后，再`hexo deploy`自动部署到git发布，最后再把源码ci到源码的git项目。

*10. 推荐及参考*

[MarkDown标记语法参考][7]

[小小小牛的Blog源码][8]

###以上~

[1]: http://jimliu.net/2013/09/08/%E4%BD%BF%E7%94%A8hexo%E6%90%AD%E5%BB%BA%E9%9D%99%E6%80%81%E5%8D%9A%E5%AE%A2/
[2]: http://zespia.tw/hexo/
[3]: https://github.com/LiuJi-Jim/hexo-theme-light
[4]: http://duoshuo.com/
[5]: http://www.net.cn/
[6]: https://help.github.com/categories/20/articles
[7]: https://www.zybuluo.com/mdeditor
[8]: https://github.com/abutme/abutme-net