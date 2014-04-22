title: ZF2的autoload机制分析
date: 2014-04-22 15:30:55
tags:
- ZF2
- 设计模式
category: 编码
---

今天开始研究一下zf2框架（含应用skeleton），很喜欢这种面向对象的风格，也是至今看过最喜欢的php代码风格。

这里说一下对框架默认autoload机制的理解，基本原理：

框架默认采用`AutoloaderFactory`来初始化和管理loaders，AutoloaderFactory通过静态工厂方法注入实现了SplAutoloader接口的待注册的loader类的路径和配置，静态工厂将配置中指定路径的loader类实例化并调用对应的register方法将loader的加载机制注册到SPL __autoload函数栈。

<!-- more -->

autoload机制的类图关系如下（只做示意）：

<img src="/images/zf2_study_autoload_1.jpg" alt="class relations by staruml" width="800" />

注意到，AutoloaderFactory的特点：

1）抽象类，猜测原因是避免实例化再调用的方式；

2）StandardAutoloader的loader机制一定会被注册；

3）维护loader的单例，支持loader的注销；

4）支持多autoload机制，允许用户自定义autoload机制；

5）Loader的异常类都是直接require，而不会递归autoload；

`StandardAutoloader`是基于文件查找的支持命名空间（namespaces）和前缀（prefixes）的类加载机制，zf2不推荐include_path的查找机制，不过还是提供了不被推荐的`fallback autoloader`的选项。

1）`namespaces`的机制是根据`命名空间+类名`建立和文件路径的对应关系，`Zend`命名空间的核心类就是以这种方式被autoload的；

2）`prefixes`的机制相当于建立5.3以前不支持namespace的类命名和路径的部署关系；

3）根据类名所包括的分隔符种类来进行模式选择的，对于指定了fallback标识的会查找include_path；

4）标准的autoload实际是利用类名和文件路径和命名的规范对应，并`没有缓存机制`。

`ClassMapAutoloader`设计的则是从性能出发，它建立类名和文件的映射关系，加载过一次的类直接从map中就可以找到。为此ZF提供了`classmap_generator.php`工具用于自动建立map对应关系。

1）支持运行时手动配置和map文件静态配置；

2）适用于应用比较频繁的类，最好是数量较稳定的类；

3）在实际上线部署中增加类以后map文件同步更新会有问题；

`ModuleAutoloader`是为`Module`准备的，在接下来的学习中再来补充。













