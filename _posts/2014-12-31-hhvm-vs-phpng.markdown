## 1.	结论

引擎+模式	吞吐

Hhvm非repo模式	1049.42

Hhvm repo 模式	1533.08

Php7 opcache.revalidate_freq 设置0	1157

Php7 opcache.revalidate_freq设置60	1185

Cpu  idle 全部压倒了0，但是由于测试机器有限（无法实现同交换机同机房），所以压力工具和数据库都在一台机器上，可以参见下面详细cpu图中hhvm的mysql cpu消耗更多一些，所以如果不是同机器的话，测试应该会更准确；
Php7 的opcache.revalidate_freq 0和60差距不大，我们就取最优的来对比吧(60)：

Php7 vs hhvm norepo ,php7高于 hhvm *12%*

Php7 vs hhvm repo ,hhvm 高于php7 *30%*

如果repo模式在非同机hhvm应该会更高，这是个相对的测试，测试的wordpress，但是repo模式我们还未在线上使用，因为需要预先线下编译，和之前PHP的流程上有些冲突，需要一套预案，目前用途广的还是非repo模式（facebook默认是repo模式）。
Repo模式比非repo模式在wp下高接近50%的，这个目前没有prof后期我会进行prof分析出结果继续跟进。

HHVM 版本是3.0.1 release

Php7是2014.12.18 dev

## 2.	测试环境

操作系统：redhat 4.3

Cpu:12 cores  Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz
    
内存：16G

预热：并发30，fetch 3000次

测试：并发50, fetch 30000次

应用:wordpress 4.1

## 3.	Hhvm 

ThreadCount:128

预热并发30，3000次

### 3.1.	Hhvm 非repo 模式
![](/img/phpng_vs_hhvm/hhvm1.jpg)
![](/img/phpng_vs_hhvm/hhvm2.jpg)

1049.42

### 3.2.	Hhvm repo 模式
![](/img/phpng_vs_hhvm/hhvm3.jpg)
![](/img/phpng_vs_hhvm/hhvm4.jpg)

1533.08

## 4.	Phpng

进程数量：100

配置：

		max_execution_time=600
		memory_limit=128M
		error_reporting=0
		display_errors=0
		log_errors=0
		user_ini.filename=
		realpath_cache_size=2M
		cgi.check_shebang_line=0
		
		zend_extension=opcache.so
		opcache.enable_cli=1
		opcache.save_comments=0
		opcache.fast_shutdown=1
		opcache.validate_timestamps=1
		opcache.revalidate_freq=0
		opcache.use_cwd=1
		opcache.max_accelerated_files=100000
		opcache.max_wasted_percentage=5
		opcache.memory_consumption=128
		opcache.consistency_checks=0
		
		extension=mbstring.so
		extension=mysql.so

### 4.1.	Phpng opcache.revalidate_freq 0
![](/img/phpng_vs_hhvm/php1.jpg)
![](/img/phpng_vs_hhvm/php2.jpg)

1157

### 4.2.	Phpng opcache.revalidate_freq 60

![](/img/phpng_vs_hhvm/php3.jpg)
![](/img/phpng_vs_hhvm/php4.jpg)

1185
