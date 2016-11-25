# openstack开发基础知识

在学习openstack源码时发现了这个系列的博客，讲的很好，分享给大家。


##为什么写这个系列

整个OpenStack项目包含了数十个主要的子项目，每个子项目所用到的库也不尽相同。因此，对于Python初学者和未接触过OpenStack项目的人来说，入门的难度相当大。

幸运的是，OpenStack中的项目有很多共同点。比如，它们有相同的代码库结构，都尽可能是用同样的库，配置文件和单元测试的规范也都几乎一样。因此，通过学习这些共通的部分，我们就可以快速掌握多个OpenStack项目。但是，万事开头难，学习这些基础知识总是痛苦的。不过，学习的难点并不在于这些知识点本身有多难理解，而是这些基础知识的应用场景和应用效果对初学者来说都是模糊的。这个系列文章的目的就是帮助有需要的人了解OpenStack中一些常见的知识点。理解过程就是通过动手做一个web application demo来实现的。

这个系列文章会涉及到以下的知识点：

- 包管理和pbr
- WSGI, RESTful Service和Pecan框架
- eventlet
- SQLAlchymy
- 单元测试

##1. 包管理和pbr

OpenStack使用setuptools工具来进行打包，不过为了满足OpenStack项目的需求，引入了一个辅助工具pbr来配合setuptools完成打包工作。

pbr 是openstack社区是为了更方便的使用setuptools而开发出来的，有以下特性：

- 使用setup.cfg文件来提供包的元数据。这个是从disutils2学来的。
- 基于requirements.txt文件来实现自动依赖安装。requirements.txt文件中包含了一个项目所要依赖的库，这个文件的格式是和pip兼容的。
- 利用Sphinx实现文档自动化。
- 基于git history自动生成AUTHORS和ChangeLog文件。
- 针对git自动创建文件列表。
- 基于git tags的版本号管理。


###推荐文章：

https://segmentfault.com/a/1190000002940724


##2. API服务

API是openstack提供服务的基础，openstack采用基于HTTP协议的RESTful API，使用python开发RESTful API要解决两个问题：

1. 服务如何部署？
2. 用什么框架？

###服务如何部署

说到Python的Web服务部署这个问题，就不得不提到WSGI。目前Python有两种方式来开发和部署一个Web应用：用WSGI和不用WSGI。

OpenStack的API服务都是使用WSGI的方式来部署的。在生产环境中部署WSGI，一般会考虑使用Web服务器 + 应用服务器 + 应用(框架)的方案。OpenStack官方推荐的是使用Apache + mod_wsgi的方案，不过这个要换成其他方案也很容易，你也可以选nginx + uWSGI。对于开发调试的目的，有些项目也会提供使用eventlet的单进程部署方案，比如Keystone项目的keystone-all命令。采用eventlet这种异步架构来进行应用开发也是一个比较大的话题，本文不覆盖这方面的内容。

当然，也可以不用WSGI。在Python中，如果不使用WSGI的化，一般开发者会选择一些专门的服务器和框架，比如Tornado，或者最新最潮的aiohttp。不过在OpenStack的项目中我还没见过不使用WSGI的。

###用什么框架

Python的Web开发框架很多，最出名自然是Django了。基本上，还活跃的框架都支持RESTful API的开发，有些框架还专门为RESTful API的开发提供了便利的功能（比如Pecan），有些框架则通过第三方模块来提供这种便利，比如Django和Flask都有不少和REST相关的第三方库。




**推荐文章：**

1. 使用OpenStack服务的方式
  - https://segmentfault.com/a/1190000003718598
2. 理解WSGI
  - http://python.jobbole.com/84372/
  - https://segmentfault.com/a/1190000003069785
3. Paste + PasteDeploy + Routes + WebOb
  - https://segmentfault.com/a/1190000003718606
4. 理解Pecan、WSME框架
  - https://segmentfault.com/a/1190000003810294
5. 用demo理解API服务
  - https://segmentfault.com/a/1190000004004179



##3. 数据库

OpenStack官方推荐的保存生产数据的是MySQL数据库，在devstack项目中也是安装了MySQL数据库。不过，因为OpenStack的项目中没有使用特定的只有在MySQL上才能用的功能，而且所采用的ORM库SQLAlchemy也支持多种数据库，所以理论上选择PostgreSQL之类的数据库来替代MySQL也是可行的。

1. 对象关系映射（ORM）：

  - 概念：简单的说，ORM就是把数据库的一张表和编程语言中的一个对象对应起来，这样我们在编程语言中操作一个对象的时候，实际上就是在操作这张表，ORM（一般是一个库）负责把我们对一个对象的操作转换成对数据库的操作。
  - SQLAlchemy项目是Python中最著名的ORM实现

2. 数据库版本管理
 - 概念
 - 实现（Alembic）

###推荐文章：
- https://segmentfault.com/a/1190000004261891#articleHeader3 本文讲述了SQLAlchemy
- https://segmentfault.com/a/1190000004466246   本文讲述了数据库版本管理模块 Alembic



##4.单元测试

Python的单元测试工具很多，为单元测试提供不同方面的功能。OpenStack的项目也基本把现在流行的单元测试工具都用全了。单元测试可以说是入门OpenStack开发的最难的部分.

###单元测试工具小结

1 测试环境管理: tox

  	使用tox来管理测试运行的虚拟环境，并且调用testrepository来执行测试用例。
 
2 测试用例的运行和管理: testrepository, subunit, coverage
 
 	testrepository调用subunit来执行测试用例，对测试结果进行聚合和管理；调用coverage来执行代码覆盖率的计算。
  
3.测试用例的编写: unittest, mock, testtools, fixtures, testscenarios
 	
	使用testtools作为所有测试用例的基类，同时应用mock, fixtures, testscenarios来更好的编写测试用例。

###推荐文章：
  - https://segmentfault.com/a/1190000004595130 本文讲了各种测试工具的用途，并以keystone为例说明如何进行测试