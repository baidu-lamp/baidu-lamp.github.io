---
layout: book_page
title:  "运维篇-监控报警 "
date:   2015-03-17 08:34:04
---
## 3. 监控报警

监控报警是我们对于HHVM运行稳定的一个基本保障，那么我们都做哪些监控和报警呢？

###	error 和warning报警

error是每抛出一个就会进行一次报警，如异常信息、超时等；

warning 的话是积累到一定条数后，然后进行报警；

这些我们都是通过定期抓取的hhvm-error的log信息。

### cpu和内存报警

cpu和内存设定一定的阀值，超过预定阀值进行报警告知运维，然后分析是否是cpu异常和内存异常，然后进行进一步处理；

### hhvm 内置监控

在hhvm中有个adminserver，我们可以通过配置的端口进行访问，然后通过一些命令获取我们的信息，当我们访问端口后会获取到如下内容界面：

![admin server](/hhvm/hhvm-in-action/imgs/monitor1.png)

但是其实我们一般常用的会有如下几个，其他的使用者可以去自己尝试（以3.0.1为例了）：

####	check-health
 
 ![Check-health](/hhvm/hhvm-in-action/imgs/monitor2.png)
 
标红位置的是我们后来添加的监控项，默认监控项就如上这些：

**load:**就是我们work的负载，也就是正在处理的线程数量，如配置了threadCount为16个，那么如果到16个的时候说明我们的目前已经是满负荷了，再来请求将会进入到队列进行等待（queue）

**queued:**队列，也就是当调度不够用时，会将请求暂时放置在队列中，等work线程释放后我们再进行处理

**hhbc-roarena-capac:**这个是在RepoAuthoritative的一种堆内存监控，具体未详细调研

**tc-xxx：**是我们对jit cache的这些监控，但是一般我们不在这里进行监控，一般是在vm-tcspace命令中进行监控

**units:**是我们已经编译过的文件数量

**funcs：**我们已经编译过的函数数量

####	vm-tcspace

 ![vm-tcspace](/hhvm/hhvm-in-action/imgs/monitor3.png)
 
上面的监控项中：code.hot、code.main、code.prof、code.stubs、code.trampolines都是hhvm jit
cache项，一般如果你是用noRepoAuthoritative模式运行，会发现code.main和code.stubs会高，一般监控这些值的比例，我们线上一般是到达80%时进行重启；

data是code的总和；

RDS是Request Data Segment

