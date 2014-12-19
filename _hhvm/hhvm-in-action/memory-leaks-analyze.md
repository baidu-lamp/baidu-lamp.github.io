---
layout: book_page
title:  "内存泄露分析"
date:   2014-10-16 00:15:04
---

##	引言

<p>内存泄露对于c和c++来说比较普遍，而旧的php引擎来说由于是多进程的模式，而且会定期清理一部分进程，所以即使有内存泄露也不是很明显或者关注不多;</p>

Hhvm是单进程多线程模型，对于稳定性要求就极高，如果其中出现了内存泄露，那么会很明显，而且目前hhvm使用人群还不多，对于分析hhvm内存的方法还了解不多，我下面就介绍一些hhvm会出现的内存泄露情况和一些分析的方法。

##	内存泄露分析方法
对于hhvm有时候的内存增长是正常的，有些确实泄露，那么我们如何很好的判断呢？
一般可分为正常的内存增长和异常增长（泄露）：
### 正常内存增长：
增长非线性的，到达一定条件后会稳定或者释放

1. 线程局部变量(持久化对象,随线程增加而增加)
1. APC
1. 更新文件（jit cache、opcode cache）

### 异常内存增长：

每次请求都会线性增长内存，并且不会释放；

1.	某个语法或者函数引起的内存泄露
	1. define动态值
	1. create_function
	1. eval动态值

1.  引擎内部泄露
  1. libevent长连接未释放 (1.0时泄露，之后fix)
  1. ini 泄露( hhvm 2.2,2.3后fix)
1.  扩展内存泄露

通过上面的方式可以初步分析出一些情况，是否是内存泄露，如果怀疑是内存泄露后，那么我们就可以通过如下3种方式进行定位：

###	排除验证法
排除验证法一般是不借助分析工具，简单的进行分析和判断内存泄露的位置，一般通过压力工具（ab,http_load）对hhvm释放压力，通过top 或者查看进程的物理内存（cat  /proc/11460/status|grep VmRSS）来查看内存，然后判断是否存在泄露和哪里泄露；

（1）	阻塞worker 造成内存堆积

有时对于释放压力压测的过程中，会出现内存积压，但是这种情况是非内存泄露，我们通过hhvm的监控接口(见下详细)查看work是否被打满，是否存在队列，如果worker满了，并且队列有积压，那么这时候停止压力后，worker和队列都恢复后，内存下降，或者将压力降低不对hhvm worker进行积压时继续观察是否内存还在上涨如果稳定则不泄露，如果还在继续涨我们再进行下一步分析。

监控接口：

`http://ipaddress:adminport/check-health`

![Check-health](/hhvm/hhvm-in-action/imgs/check-health.png)
 
其中load是worker的活动数量，如果load满了（ThreadCount），将会进入到队列中(queued)，处理完毕后会都恢复；

（2）	排除正常内存增长

（3）	注释法判断函数泄露

如果排除掉不是由于挤压队列造成的挤压，又非正常内存增长，而是真正的内存泄露，那么可以根据二分法和注释法判断具体是哪个函数泄露，如果定位后，就可以根据泄露进行下一步的修复或者绕过；

但是此方法适合于业务简单的验证，或者已经定位了泄露点触发范围，一般通过此方法进行验证（一般结合如下2种分析结果后，定位问题通过此法进行验证）。

###	Jemalloc

Hhvm 采用2种内存分配库(tcmalloc和jemalloc)，目前采用的是jemalloc，而我们也是借用此项工具的prof功能和hhvm提供的adminServer的接口相配合一起进行分析。

hhvm可以用jemalloc的profile进行diff的差异比较生成图表来看内存在哪块区域里面进行了泄露；

首先安装jemalloc的时候需要添加编译选项：

`./configure --enable-prof`

添加prof选项后，然后安装jemalloc，然后我们就可以利用jemalloc的prof来对于内存进行分析了（编译好jemalloc后，我们可以将支持prof功能的jemalloc动态库copy到hhvm的cmake_prefix_path的位置，替换原来的库）；

首先在运行hhvm之前需要配置下面的环境变量：
export MALLOC_CONF=prof:true

然后我们运行hhvm，配置好管理端口（8084管理端口）

	AdminServer {
	    Port = 8084
	}


然后我们运行下面的命令：

`http://127.0.0.1:8084/jemalloc-prof-dump?file=prof1.txt`

这时会有一个文件（prof1.txt）生成到我们的hhvm运行的目录下面;

生成prof1.txt文件时我们只启动hhvm其他任何工作不要做，保证数据是干净的；

然后我们针对可能出现内存泄露的url进行压力测试，压测一段时间之后，我们再次运行dump命令（注：这里需要一个新的名字prof2.txt）

`http://127.0.0.1:8084/jemalloc-prof-dump?file=prof2.txt`

然后比较2个文件的差异性，生成图表：

pprof是按照jemalloc的时候生成的二进制文件；

（1）生成diff文件

	jemalloc/bin/pprof --text --base prof1.txt /home/work/data/test/hhvm/output/prefix/bin/hhvm prof2.txt >diffprof.txt

（2）生成diff的dot

	Jemalloc/bin/pprof --dot --base prof1.txt /home/work/data/test/hhvm/output/prefix/bin/hhvm prof2.txt >diffprof.dot

（3）通过graphviz的dot对于diff文件生成png文件(graphviz 制图工具需要安装)

	graphviz/bin/dot -Tpng diffprof.dot > diffprof.png

这样生成好了内存的diff 文件我们可以进行比对;

然后会生成如下图片：
 
![Alt text](/hhvm/hhvm-in-action/imgs/memory-leak.png)
 
这样我们可以根据图中的diff泄露点，然后找到我们需要解决的问题，这样对于庞大的hhvm来说就比valgrind好用多了；

###	Valgrand
Valgrand 是一款很好的内存分析工具，但是由于打印的信息过多，也许会造成你不容易发现真正泄露点的位置，而由于hhvm在启动的过程中其实默认就会有一些不释放的内存（默认线程局部变量），会让很多初期接触者迷茫，所以建议Valgrand用于定位扩展和怀疑位置时通过查看valgrand dump的日志来查看，由于网上对于valgrand资料很多，所以下面简单介绍下使用方式：

（1）	安装valgrand

（2）	扫描内存

	valgrind --leak-check=full  --tool=memcheck --log-file=leak.log hhvm -m server -c config.hdf

通过valgrind启动hhvm ，然后访问我们可能会泄露的case，然后ctrl+c终端进程信号，会dump一些未释放的内存会存在leak.log中，如果我们是对于扩展的内存扫描，就可以直接grep我们扩展的一些特定内存即可；

（3）	定位内容进行修复，重复扫描

## 总结

通过如上总结，我们一般是通过如上3种方法结合使用，但可分为如下几种场景:

（1）	复杂场景

Jemalloc+排除验证法

由于业务场景复杂，通过jemalloc可以很快的分析出泄露点的大概位置，根据经验分析哪里是否是泄露点，然后根据怀疑点进行排除验证，最终定位问题进行修复。

（2）	扩展开发场景

Jemalloc+Valgriand+排除验证法

扩展开发时其实3种方法结合使用也可以，或者valgrind+排除验证法也可以，因为扩展代码中代码范围可控，所以通过valgrind扫描我们可以准确定位扫出来的内容是否是泄露

内存泄露对于我们来说分析比crash要麻烦些，不仅仅需要借助工具、case复现，而且还需要一些经验来进行定位，增加了我们的成本，但是有了如上几种方法希望可以对大家分析有用。

