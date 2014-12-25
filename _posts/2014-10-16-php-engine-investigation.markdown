---
layout: post
title:  "PHP 引擎调研"
author: wangweibing
date:   2014-10-16 17:18:04
tags: PHP 性能分析 HHVM Zend引擎
---


### 简介

该调研是2013年10月份做的，目标是寻找更好的PHP引擎，来代替百度各产品线正在使用的PHP 5.2。

### 环境说明

机器环境：

* cpu: Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz， 12核。
* 内存：64G

引擎：

1.	php 5.2.17 (当时百度所用版本)
2.	php 5.5.4 (当时最新版本)
3.	hhvm 2.3-dev (当时最新版本)

为了公平起见，三个引擎都采用-O2的编译优化选项。

加速器：

1.	php5.2.17：采用eaccelerator
2.	php5.5.4：eacc不可用，采用自带的zend opcache
3.	hhvm：内置opcode缓存，开启jit

扩展：

由于目前的百度扩展只有很少移植到了hhvm，所以为了公平起见，其它百度扩展也在php5.2和php5.5中禁用，而采用纯php的方式实现，实践证明采用纯php实现性能并未有太大损失。

### 性能

下面用了不同的场景，来比较PHP引擎的性能。

#### 场景一：纯cpu计算

用php官方提供的测试脚本：

* bench.php : <https://github.com/php/php-src/blob/master/Zend/bench.php>
* micro\_bench.php : <https://github.com/php/php-src/blob/master/Zend/micro_bench.php>
* 第三方提供的测试脚本bench_third.php : <http://www.php-benchmark-script.com/>

测试结果如下：

<table>
<thead>
<tr>
  <th>引擎</th>
  <th>bench.php 耗时</th>
  <th>micro_bench.php 耗时</th>
  <th>bench_third.php 耗时</th>
</tr>
</thead>
<tbody>
<tr>
  <td>php5.2</td>
  <td>6.692s</td>
  <td>41.890s</td>
  <td>9.226s</td>
</tr>
<tr>
  <td>php5.5</td>
  <td>3.609s</td>
  <td>14.972s</td>
  <td>5.893s</td>
</tr>
<tr>
  <td>hhvm</td>
  <td>0.579s</td>
  <td>5.832s</td>
  <td>2.869s</td>
</tr>
</tbody>
</table>

分析：php历次新版本对性能都有优化，因此php5.5在纯cpu计算上比php5.2有明显优势。而hhvm采用jit机制，将php转化为本地代码执行，所以性能上比执行opcode的php5.5还要快上很多。

**结论：对于纯cpu型计算，hhvm>>php5.5>>php5.2。**

#### 场景二：简单http服务

简单的http服务即由一个简单的php逻辑接口层加一个后端C模块（或通用存储）构成的服务，这里用了百度内部一个实际的简单服务为例。

简单做了一下性能测试，client并发10，client与service同机部署，service与后端C模块不同机部署。结果如下：

<table>
<thead>
<tr>
  <th>引擎</th>
  <th>平均响应时间</th>
  <th>Cpu idle</th>
  <th>Qps</th>
</tr>
</thead>
<tbody>
<tr>
  <td>php5.2</td>
  <td>3.6ms</td>
  <td>50%</td>
  <td>1900</td>
</tr>
<tr>
  <td>php5.5</td>
  <td>2.9ms</td>
  <td>60%</td>
  <td>2500</td>
</tr>
<tr>
  <td>hhvm</td>
  <td>2.0ms</td>
  <td>60%</td>
  <td>3900</td>
</tr>
</tbody>
</table>

分析：对于简单http服务，自身也有一部分的逻辑计算，除此之外，hhvm是webserver与php引擎一体化，不需要webserver->php这一层的开销，因此在响应时间和qps上有更明显的优势。由于C模块后端本身响应时间较短（1ms），所以不同引擎的整体响应时间的区别就比较明显。

**结论：简单http服务，在qps上hhvm>>php5.5>php5.2，在响应时间上，除掉后端服务耗时，也是hhvm>>php5.5>php5.2。**

#### 场景三：产品前端UI

这里用了贴吧的最新版ui，串行访问后端。

采用20个并发，测试结果如下：

<table>
<thead>
<tr>
  <th>引擎</th>
  <th>整体耗时</th>
  <th>除访问后端之外的耗时</th>
  <th>idle</th>
  <th>qps</th>
</tr>
</thead>
<tbody>
<tr>
  <td>php5.2</td>
  <td>520ms</td>
  <td>127ms</td>
  <td>50%</td>
  <td>40</td>
</tr>
<tr>
  <td>php5.5</td>
  <td>500ms</td>
  <td>107ms</td>
  <td>45%</td>
  <td>45</td>
</tr>
<tr>
  <td>hhvm</td>
  <td>470ms</td>
  <td>72ms</td>
  <td>65%</td>
  <td>50</td>
</tr>
</tbody>
</table>

分析：ui大部分时间消耗在后端服务访问，这方面引擎不起作用。而另一方面，除后端服务外的耗时也很高，这部分对cpu消耗较大，导致qps较低。

显然，这个效果并不是很符合我们的预期，为了达到理想的qps，我们需要想方法进一步降低除后端之外的耗时。

首先我们在PHP5.2下运行xhprof，找出了一些较耗时的函数，并进行了优化：

1.	去掉不必要的编码转换，减少`15ms`
1.  去掉不必要的warning日志，耗时减小`4.3ms`
1.	改进xss转码函数实现，耗时减少`1.8ms`

通过上面几个优化，HHVM下ui的耗时降到了52ms，其中模板耗时34ms。但是这样仍不令人满意，如何进一步优化呢？

既然上面的xhprof是在PHP5.2下执行的，那我们尝试直接在HHVM下运行xhprof会如何？结果发现，HHVM下xhprof的结果与PHP5.2下有很大不同，发现了一些新的优化点，进行了优化：

1.	有好几个函数会在每个请求中重复计算同样的信息，加上apc cache之后，耗时减小`16ms`
1.	用户自定义的errorHandler，很耗时去掉后耗时减小`7ms`
1.	去掉一些debug日志，耗时减小`1ms`

通过上面的优化，我们将HHVM下ui的耗时降到了26ms。那么，同样的优化，作用在PHP5.2或PHP5.5下又如何呢？

<table>
<thead>
<tr>
  <th>引擎</th>
  <th>优化前</th>
  <th>业务优化后</th>
</tr>
</thead>
<tbody>
<tr>
  <td>php5.2</td>
  <td>127ms</td>
  <td>103ms</td>
</tr>
<tr>
  <td>php5.5</td>
  <td>107ms</td>
  <td>87ms</td>
</tr>
<tr>
  <td>hhvm</td>
  <td>72ms</td>
  <td>26ms</td>
</tr>
</tbody>
</table>

令人失望。。。究其原因，是因为在php下耗时在各函数的分布相当分散，即使优化个别函数，也难以有大辐提升。

而在hhvm下，由于引擎本身性能较高，因此一些原来实现得不合理而比较耗时的函数就突显出来了，因此针对这些函数进行优化，可以取得更加明显的效果。

**结论：对于产品前端UI，引擎可以在一定程度上优化响应时间和qps，hhvm>php5.5>php5.2，如再结合一些针对hhvm的优化，可以大幅提升hhvm下的qps。**

### 兼容性
#### 语言兼容性
语言方面的兼容性来自两个方面：

##### php不同版本的语法差异

hhvm是基于php5.4的语法标准，因此，hhvm和php5.5与我们目前用的php5.2都存在语法差异，php5.5差异更大一些。

目前所遇到的差异个数如下：

<table>
<thead>
<tr>
  <th>引擎</th>
  <th>影响处理结果</th>
  <th>不影响结果，打warning</th>
</tr>
</thead>
<tbody>
<tr>
  <td>php5.5</td>
  <td>1个</td>
  <td>1个</td>
</tr>
<tr>
  <td>hhvm</td>
  <td>1个</td>
  <td>1个</td>
</tr>
</tbody>
</table>



所有差异可见这里：

* <http://www.php.net/manual/en/migration53.php>
* <http://www.php.net/manual/en/migration54.php>
* <http://www.php.net/manual/en/migration55.php>

##### hhvm与zend引擎之间的实现差异
由于是两套不同实现，因此hhvm与zend引擎也有差异。

目前遇到的差异数如下：

<table>
<thead>
<tr>
  <th>影响处理结果</th>
  <th>不影响结果，打warning</th>
</tr>
</thead>
<tbody>
<tr>
  <td>2个</td>
  <td>0个</td>
</tr>
</tbody>
</table>

hhvm文档也提到了一些差异：<https://github.com/facebook/hhvm/blob/master/hphp/doc/inconsistencies>

**结论：总体上，hhvm与php5.5在语言方面与php5.2都存在一些差异，并且对处理结果造成影响。不过，大部分的差异都会导致执行时报fatal或warning，我们只需根据相应提示修改代码即可，改动成本不大。**

#### 扩展兼容性
#####	php5.5
将原百度用到的扩展在php5.5下编译，大部分能编译通过，少数编译失败：

1. 升级到最新版本可解决，5个
1. 修改代码解决，2个

#####	hhvm
hhvm内置了很多常用的第三方扩展，因此对于第三方扩展，基本无须迁移。主要需要考虑的是百度扩展的迁移。

hhvm编写扩展有两种方式：

一、静态编译

1.	编写扩展的idl文件（json格式）
2.	根据idl自动生成代码框架
3.	填充代码框架，实现自己的逻辑
4.	重新编译hhvm

缺点：

1.	现有百度扩展需要重新改造，与获取输入参数、返回结果相关及其它调用到php.h中的api的地方都需要重写。这是主要的成本。
2.	hhvm的api目前还不太稳定，2.1.0下编写的扩展，在2.2.0下就编译不通过了。。

优点：

1.	新扩展编写，比在zend下编写更加方便，hhvm提供类似于boost variant的api，处理输入输出很方便

二、动态加载（HNI：hhvm native interface）

1.	编写函数声明.php（语法与php很像）
2.	编写函数实现.cpp
3.	hphpize && cmake . && make，只须单独编译扩展

动态加载更为方便，因此推荐使用这种方式编写扩展。

三、ZendCompatLayer

hhvm正在实现一个ZendCompatLayer，即兼容zend扩展的一个层，实现zend引擎相关api，以方便现有扩展迁移到hhvm。具体见：
<https://github.com/facebook/hhvm/blob/master/hphp/doc/php.extension.compat.layer>

按照其描述，zend扩展可以不做太多修改，即可迁移到hhvm。

1.	尝试开启ZendCompatLayer功能，并编译包中自带的yaml扩展，确认ZendCompatLayer已经可用
2.	尝试使用ZendCompatLayer编译两个百度扩展，发现很多问题，比如链接时缺少各种函数、加载不了ini、调用不了minit/rinit等函数、gcc版本不兼容、原扩展不支持多线程环境等等，经过各种修改hhvm和扩展代码，终于跑通，但成本很高，而且稳定性很难保障。
3.	测试了一下ZendCompatLayer的性能，用一个包含string, int和嵌套数组的数组，反复调用一个百度扩展做打包解包，结果如下：

<table>
<thead>
<tr>
  <th>耗时</th>
  <th>实现方式</th>
</tr>
</thead>
<tbody>
<tr>
  <td>199 ms</td>
  <td>HNI</td>
</tr>
<tr>
  <td>769 ms</td>
  <td>HNI+ZendCompat</td>
</tr>
<tr>
  <td>348 ms</td>
  <td>php5.2</td>
</tr>
</tbody>
</table>

结论：ZendCompatLayer目前还不完善，很多api未实现，另外性能还较差，对性能要求高的扩展还是建议用hhvm原生方式实现，而长尾扩展可以用ZendCompatLayer，减少迁移与维护成本。

**结论：php5.5对扩展保持较好的兼容性，迁移成本不高。hhvm目前扩展迁移成本较高，不过如果有ZendCompatLayer，也能大大减小迁移成本，可以等ZendCompatLayer完善之后，再做进一步调研。**

#### 系统兼容性
php5.5对操作系统基本没有什么要求。

hhvm对系统要求较高，需要在ubuntu/centos下编译、运行，百度大部分机器不符合条件。。

不过，经过研究后发现，可以通过打包so方式，让hhvm能在百度机器运行，上面的测试都是在百度机器上进行的，没有发现什么问题。

根据同样的原理，通过打包依赖so和头文件，也能让hhvm在百度机器上编译，已经成功编译。

内核方面，hhvm要求至少2.6.32系统，百度大部分系统都符合要求。

**结论：系统兼容性上，php5.5和hhvm都没有什么问题。**

#### 配置兼容性

php5.5对php.ini保持兼容，但php-fpm.conf则变成ini格式，原来的php-fpm.conf需要修改，并且php-fpm也被编译成一个单独的可执行文件，而不再和php-cgi共用。

hhvm的配置是一种新的hdf格式，而且其配置项都不相同，不一定每个php.ini配置都能找到对应的配置项，不过常见的都有。新版hhvm也支持指定php.ini文件了，不过仍然不能支持所有配置项。

hhvm是webserver与php一体化，webserver的常见功能，如rewrite、简单防攻击、proxy、日志、静态文件服务也都有。因此webserver的相关配置也要成hhvm的配置格式。不过现在HHVM也支持fastcgi模式了，所以这个已经不成问题。

**结论：配置兼容性上php5.5更好一些。不过配置文件也并不多，人工改造成本也不是很高，因此这方面问题并不大。**

### 新功能

php从5.3到5.5增加了很多新功能，具体可以见：

* <http://www.php.net/manual/en/migration53.php>
* <http://www.php.net/manual/en/migration54.php>
* <http://www.php.net/manual/en/migration55.php>

总体上可概括为语法更灵活，功能更丰富。

hhvm的语法基于5.4，库的支持也不如php5.5丰富。但hhvm提供了几个功能，可能对性能优化会有很大帮助：

1.	StartupDocument：可以在启动时执行该文件，在此文件中做一些全局初始化工作，并将结果变量通过apc传递给worker线程。这样我们将一些重复的初始化工作放到该文件，避免每个请求都执行同样的工作。
2.	ThreadDocuments：可以用后台线程执行一个文件，做一些比如信息收集、定时更新之类的操作，避免在每个请求中执行相应过程，影响在线请求的响应时间。
3.	Pagelet Server/Xbox Server：可以启动一个子Server，子Server维护一个线程池，主Server可以将一些计算任务分发给子Server异步执行，从而实现并行化。我尝试了一下这个功能，利用xbox server很方便地就实现了并行调用后端服务的功能，确实很方便，而且这个机制是通用的，不仅后端调用可以并行，本地计算也可以并行，可以考虑将模板并行渲染？呵呵。
4.	多线程架构：这其实不算一个功能，而是hhvm多线程架构给我们带来的天然便利。因为在同一个进程内，内存共享会很方便，访问速度更快，而像一些比如连接池的功能，也变得可以实现。

上述功能的介绍详见：

* <https://github.com/facebook/hhvm/blob/master/hphp/doc/server.documents>
* <https://github.com/facebook/hhvm/blob/master/hphp/doc/threading>

### 开发者支持

php5.5的开发者支持很好，文档全，api非常稳定。

相比之下，hhvm的开发者支持目前还比较弱，文档很少，很多地方要自己摸索，api还不太稳定，不过hhvm的社区很活跃，github有很多人在提issue和patch，hhvm官方团队也很给力，迭代速度很快，每8周发布一个稳定版本。

### 总体结论

总体上，使用php5.5，性能会有一定提升，迁移成本较小。

使用hhvm，性能会有很大提升，但迁移成本会较大，主要是扩展方面的迁移成本。

------

**UPDATE**

看这里的结论很明显，百度大部分产品线使用了HHVM，因为我们相信资源的节约对于
我们这么大规模的应用收益很大。

读到这里，你可能会有疑问思考你到底要不要升级到HHVM？你想知道PHP7对比怎么样？

这里是你**需要迁移**HHVM的原因：

1. 你现在服务器成本比较大，想要节约成本，或者流量持续增长资源不足。
1. 你需要比较透明的提升性能
1. 你没有什么自有扩展，或者有时间迁移扩展（扩展迁移到HHVM的成本很低，后面会做相关的介绍）
1. 你想使用Hack语言
1. 或者你们想尝试新的技术

同样，这也有**不建议迁移**HHVM的原因：

1. 你的服务器及流量很少
1. 你的业务很稳定，没有性能上的要求
1. 你们是土豪，机器随便堆，预算一大把

**这还有一些答疑：**

- Q: 有人提到运维成本的问题，HHVM真的不容易运维么？
- A: HHVM的多线程模型是相对PHP-FPM不稳定，但是有非常多的进程守护解决方案可以很好的解决，
   这个不是个问题，以百度现在的运维来看比较稳定。

- Q：PHP7的性能不是已经不错了么？为什么还要迁移到HHVM？
- A：首先PHP7还处于开发状态，首先迁移大版本的风险还是有的，至少现在考虑是不靠谱的，如果愿意等，
  可以等到PHP7出来以后来进行对比。看看哪个更适合你，目前这个阶段HHVM是比PHP更优的选择。

**做你们的测试(Do Your Own Benchmarking)**

如果你们决定迁移了，那么开始前也建议做一次你们业务相关的评测，以确保收益在预期范围内。
