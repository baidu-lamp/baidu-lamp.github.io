---
layout: post
title:  "HHVM vs PHP7"
date:   2014-12-31 09:00:00
author: huzhiguang
tags: HHVM Baidu
---

## 1.	结论

<table>
<thead>
<tr>
  <th>引擎+模式	</th>
  <th>吞吐</th>
</tr>
</thead>
<tbody>
<tr>
  <td>HHVM非repo模式</td>
  <td>834</td>
</tr>
<tr>
  <td>HHVMrepo模式</td>
  <td>1093</td>
</tr>
<tr>
  <td>Php7 opcache.revalidate_freq 设置0</td>
  <td>800</td>
</tr>
<tr>
  <td>Php7 opcache.revalidate_freq设置60</td>
  <td>844</td>
</tr>
</tbody>
</table>

之前测试过一次混合部署的数据，但是那样不是很准确，会受到mysql等干扰（穷啊，没有同机房、同交换机下的其他测试机），所以采用了cpu隔离的方式将hhvm、mysql和httpload分别进行隔离（绑定cpu用taskset）：

Php和hhvm绑定cpu 在0-6核（7个core,php绑定100个进程，hhvm绑定一个进程）

Mysql绑定cpu在8-10(3个core)

http绑定cpu在11（1个core）

测试这次都采用100个 work(进程或线程)，并且把7个core都给打满，这样最终比较吞吐还是比较准确的，其他无干扰，数据如下:

<table>
<thead>
<tr>
  <th>引擎+模式	</th>
  <th>比例</th>
</tr>
</thead>
<tbody>
<tr>
  <td>Php7（60） vs hhvm norepo </td>
  <td>php7高于hhvm 是 **1%**</td>
</tr>
<tr>
  <td>Php7（0） vs hhvm norepo </td>
  <td>hhvm 高于php 7 是 **4%**</td>
</tr>
<tr>
  <td>Php7(60) vs hhvm repo</td>
  <td>hhvm 高于php7 是 **30%**</td>
</tr>
<tr>
  <td>Php7(0) vs hhvm repo</td>
  <td>hhvm 高于php 7 是 **37%**</td>
</tr>
<tr>
  <td>Hhvm norepo vs hhvm repo </td>
  <td>hhvm repo高于hhvm norepo 是 **31%**</td>
</tr>
</tbody>
</table>
work除了100外也设置为24测试了，吞吐hhvm和php7都会适当提升10%左右，但是相对比例都跟如上接近。

综上，php7 跟hhvm norepo模式性能差不多，Php7 的opcache.revalidate_freq 0和60差距不大，我们就取最优的来对比吧(60)，hhvm repo模式比php7高**30%**，这是个相对的测试，测试的wordpress，但是repo模式我们还未在线上使用，因为需要预先线下编译，和之前PHP的流程上有些冲突，需要一套预案，目前用途广的还是非repo模式（facebook默认是repo模式）。

Repo模式比非repo模式在wp下高接近31%的，这个目前没有prof后期我会进行prof分析出结果继续跟进。

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

ThreadCount:100

预热并发30，3000次

### 3.1.	Hhvm 非repo 模式

![](/img/phpng_vs_hhvm/hhvm1.jpg)

![](/img/phpng_vs_hhvm/hhvm2.jpg)

834

### 3.2.	Hhvm repo 模式

![](/img/phpng_vs_hhvm/hhvm3.jpg)

![](/img/phpng_vs_hhvm/hhvm4.jpg)

1093

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

800

### 4.2.	Phpng opcache.revalidate_freq 60

![](/img/phpng_vs_hhvm/php3.jpg)

![](/img/phpng_vs_hhvm/php4.jpg)

844
