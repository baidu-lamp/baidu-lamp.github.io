---
layout: post
title:  "HHVM at Baidu"
date:   2014-11-04 00:15:04
author: huzhiguang
tags: HHVM Baidu
---

在这之前我们介绍了[我们为什么要迁移PHP到HHVM](http://lamp.baidu.com/2014/10/17/php-engine-investigation/)，
本文将介绍HHVM在百度的应用情况以及我们遇到的问题及经验。


##背景

HHVM前身是HipHop PHP，HipHop通过将php代码->cpp代码->二进制的转换来提升性能，
Facebook应用了4年（2007-2011），但是由于开发、编译、调试、维护不方便，
2011年12月Facebook开始了HHVM的开发和调研。

以下为其发展的路线：

1. HipHop PHP: 将PHP代码翻译成C\++代码，然后编译成二进制运行，有很大的性能提升，但开发成本比较大。
1. hphpi：为了解决编译太慢的问题，其实现的PHP解释器用于提升开发效率，不过代码和hphpc很多都不一样，有很多问题，可能导致线下bug不能发现。
1. HHVM：真正的虚拟机，目标是和Zend VM保持兼容。

HHVM是HipHop Interpreter的改进版，方便调试使用，性能只有hphpc的一半，但是到了2012年11月，
HHVM到了一个里程碑，它的性能接近了hphpc，随后HHVM2.2超过了hphpc，性能比HipHop多40%，HHVM2.3又比HHVM2.2提升20%的性能。

具体HHVM的历史过程可以参见[他们的博客](https://www.facebook.com/notes/facebook-engineering/speeding-up-php-based-development-with-hhvm/10151170460698920)，还有[官方网站](http://hhvm.com/blog)

百度从2013年11月开始正式启动PHP业务到HHVM的迁移，通过多部门合作首先在某业务线进行了HHVM的迁移，上线后性能提升了**60%**以上，之后其他业务线也跟进迁移HHVM。经过各大产品的迁移积累的经验，百度内部目前的HHVM技术支持及扩展库等都比较完善了，下面将具体介绍HHVM在百度的使用情况。

##使用规模及收益(Deployment Scale & Results)

* 部署机器规模超过**4000台+**
* 日均访问HHVM的PV超过了**500亿以上**（包括了接口调用和用户访问）
* CPU使用率节约**40%~60%**
* 响应时间减少**50%~80%**

对于CPU计算密集型的业务来说收益最为明显，CPU的节约和响应时间相关比较大，而对于IO密集型的应用来说可能没有线性关系。（但是可以利用并行处理和异步化来优化IO密集型的应用，比如使用HHVM提供的异步操作进行，以后会单独介绍这一块）。

##迁移方式(Migration)

对于一个新的技术来说，通常是会存在一些问题的，为了保证迁移的顺利，我们的迁移并不是一口气全部迁移过去的，迁移方式也采用的渐进式，一来通过在线运行的方式积累经验，二来减少因为不稳定对已有服务的影响。

我们的迁移经历了两个过程：旁路阶段及全量阶段。下面介绍我们迁移时的做法。

###旁路阶段

所谓旁路阶段是我们的业务任然使用Zend PHP，但是一小部分功能使用HHVM来做，主要的业务通过RPC的方式来进行通信。这样运行环境就隔离开了。

![](/content/images/2014/11/hhvm-side-1.png)

比如我们为了确认迁移的收益，我们使用HHVM去处理消耗CPU较大，而且比较容易迁移的代码，比如Smarty渲染占了我们业务性能的60%，所以我们把这个逻辑单独抽取出来放到HHVM中运行，其他的逻辑依旧在Zend运行，也就是把应用拆成了2个部分分别运行，那么这时我们就需要考虑如下几个问题：

 1. ZEND和HHVM通信方式
 1. 容错容灾：出现问题是怎么避免和降低影响
 
**ZEND 和 HHVM 通信:**

我们通过curl+共享内存的方式将zend和hhvm连接在一起，curl用于触发hhvm执行逻辑但是不传输数据，共享内存主要是传数据在2个进程之间，这样curl不用post传数据，数据在内存中传输相应也会提高很多性能，共享内存之前HHVM有个内置的，但是我们业务这边又开发了一个shmop比较通用的扩展来支持共享内存的功能，这样的旁路迁移后，比如在HHVM处理的部分占之前消耗60%，那么我们这一般可以提升性能在30%以上；

**容错容灾**

1. HHVM 进程状态监控，挂起及时拉起（supervise)
1. HHVM 失败后流量切换
   我们一般前面有nginx/lighttpd这些web server，如果失败了，一般会通过error_page或者lighttpd的负载容错的措施将失败流量转到ZEND上，保证连接正常（但是延迟会高，也有风险，我将会在全量时说明）
   
1. HHVM状态监控
   通过HHVM admin server进行HHVM状态监控，如：
   check-health (load,queue,funcs,units)
   vm-tcspace(监控tc状态，一般astub和acode会超出阀值，动态翻译时此值会根据你调用函数更新后进行上涨）
   当超过一定阀值后会进行报警告知OP，然后进行相应处理

###全量阶段

![](/content/images/2014/11/hhvm-full-1.png)

全量阶段主要是在功能完善后（扩展），去掉了共享内存这个中间层，直接上线整个应用，此时的通信就直接是web server连接hhvm，在hhvm失败时进行容错容灾处理（同旁路阶段处理）

**<span style="color:red">注:</span>**

但是全量后的HHVM如果错误切到ZEND上风险也很大，一般在高峰流量时，HHVM使用一半CPU或者30%左右，但是切换到ZEND 后，一般CPU会被打满，这样就会造成服务拒绝（这样影响也是很大的),后期我们将考虑双HHVM的预案；

##扩展开发和框架使用(extension develop and use framwork)

###扩展开发
HHVM 虽然性能较好，但是初期兼容性上要做不少工作的，扩展开发就是其中重要的一部分，如下是我们所做的工作：

* 开发、迁移扩展25+（其中包括公司内部扩展和未实现的通用扩展）
* 使用的通用扩展 mysqli,mysql,memcached,redis,apc and so on
* 可进行开源的扩展 ap(yaf),protobuf,shmop  其中ap 为纯php(hack) merge而成；

对于扩展我们一般都会进行php单元测试、cpp单元测试、压力测试、稳定测试、功能测试来确保迁移或者开发扩展的质量。

### 使用的PHP框架
* smarty
  模板渲染，使用HHVM一般来说可以提高50%以上性能（但避免使用eval等动态语法）
* phpunit
  扩展php单测框架

## 上线
### 测试方式

为了确保我们的迁移是成功的，我们使用了一下的方式来保证质量：

* 功能测试：功能输出diff比较、tcpcopy对比、网络包对比
* 性能测试：确保性能符合预期
* 稳定性测试：确保没有：
	- 内存泄露
	- crash
	- Hang住
	- 错误日志

## 遇到的问题及经验

一下总结我们在迁移和维护HHVM的过程终于到的问题及经验。

我们遇到的问题主要有：

* crash
* 内存泄露
* 死锁
* 兼容性问题
* 性能问题

下面只是简单的汇总，后续会考虑把问题追查的过程和方法分享出来。

### Crash
我们在线下测试的时候也会遇到一些Crash，我们争取在线下修复这些crash，如果是HHVM本身的Bug，我们会将修复提交给官方。

HHVM使用的时单进程多线程模型，不像PHP-FPM的多进程，如果程序crash了会导致服务挂掉，这里建议使用`supervior`这种进程监护程序来保证即使是HHVM崩溃了也能即使的帮你重启程序，减少因为crash而导致影响服务。

### 内存泄露
为了确保服务的稳定性我们会在线下进行压测，来提前发现内存泄露，内存泄露HHVM本身可能有，我们自己开发的扩展力可能也会有。 如果我们发现了可能的泄露，我们使用`jemalloc`及`valgrind`来辅助定位问题。比如我们发现了一下的泄露：
  
  1. libevent keeaplive (fixed)
  1. Init::setting (2.3 fixed)
  1. define 函数动态变量内存泄露（百度内部fix一部分未提供给官方）
  1. create_function 内存泄露（百度内部fix一部分未提供给官方）
  1. uploadFile 功能内存泄露（fixed）
  
### 兼容性问题
兼容性问题主要有两种:

  1. HHVM本身的兼容性不够完善：这个可能我们通过一个Hack的方式来避免
  1. 我们使用是PHP5.2，迁移到HHVM之后对应于PHP的版本是5.5，这个时候也是有兼容性差异的。

这类的问题都比较好解决，通过测试一般都是可以发现的。

### 死锁 hanging
我们在迁移时也发现了死锁的情况，表现是HHVM没办法处理请求，日志不打印，如果此时。

  1. 代码递归死循环（如pcre + 魔术函数递归）
  1. 死锁：比如我们遇到过因为扩展问题导致出core，但是由于HHVM捕获到core后处理时死锁的问题。

### 性能问题 Performance issue
  1. global scope (未将代码放在函数内执行,不执行jit)
  1. Exception
  1. 使用eval和create_function动态函数
  1. exit 函数
  1.  Jit translate code (datablock) 
  	* stub和maincode超过阀值crash（可通过vm-tcspace监控，后期可通过内部双tc buffer替换，或者双hhvm 进行替换）
  1. nonAuthoritativeRepo
  	* Facebook使用repo 模式，就是线下编译为hhbc，官方说法可以在预热时提高30%-40%性能（其实就是节约了编译时间），但是hhvm却不支持热加载（后续可添加此功能）
  1.我们使用的是非线下编译，直接去check php 文件，有更新则执行编译+翻译，但是此种方式的问题是，当更新文件过多和更新完触发文件多样性时，所有的work线程就会都进行处理编译+翻译，这样cpu将会一下子被打满，后续我们计划将编译+翻译放到独立线程池处理，通过双buffer替换方式，这样不会由request work处理编译和翻译会减少阻塞

## 我们的工作(our job)
1. 内部修复超过60个补丁（包括官方拉取的patch到我们内部版本，和bug修复后PR给官方的）
1. 维护2个HHVM版本(2.2和3.0.1),Facebook不维护低版本，但是我们内部需要使用，我们进行了向下兼容
1. 操作系统支持：支持redhat 和centos,无依赖系统独立gcc-4.8.2,可任意部署
1. 修复Bug、更新hhvm版本和定期patch,提升性能，兼容zend和支持新feature

## 下一步计划(next plan)
1. 提交更多内部补丁到开源社区：我们任然有一部分Patch没有提交到官方，后续会把这些Patch进行提交。
1. 复用jit translate cache：解决JIT超限问题
1. 编译、emithhbc和翻译jit阶段使用非work模式的多线程模式，采用双buffer替换减少阻塞和cpu消耗
1. hhbc 支持热加载
1. 实现异步function和异步io模型等
1. 更新hhvm 3.6版本预计在明年（Facebook工程师预计在明年会将hhvm的性能提高50%-300%）


后续完成后会在本站发表相关的技术细节。
