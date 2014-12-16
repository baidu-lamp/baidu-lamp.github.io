---
layout: post
title:  "HHVM简介"
date:   2014-10-16 00:15:04
author: wangweibing
en_url: /2014/10/16/hhvm-introduction-en/
---




HHVM (HipHop Virtual Machine) 是 Facebook 开源的 PHP 执行引擎。 HHVM 采用一种JIT（just-in-time）的编译机制实现了高性能，同时又保持对 PHP 语法的充分支持。 

### HHVM功能

* PHP5.4语法的完全支持
* [hack语言](http://hacklang.org/)支持
* 可支持cli, fastcgi server（相当于phpfpm)，http server（相当于nginx+phpfpm）三种运行方式

### HHVM优势

* 高性能：从目前百度已经上线的应用看，与php5.2相比，普遍节省cpu `40%` ~ `60%`。
* 易用的扩展API：HHVM扩展编写非常方便，同一个扩展，平均代码量不到zend扩展的一半。
* 单进程架构：更方便运维，方便共享全局数据结构（如配置、字典甚至网络连接）。

下面是一篇介绍HHVM的PPT：

<object height="500" align="middle" width="100%" id="reader" codebase="http://fpdownload.macromedia.com/pub/shockwave/cabs/flash/swflash.cab#version=6,0,0,0" classid="clsid:d27cdb6e-ae6d-11cf-96b8-444553540000"><param value="window" name="wmode"><param value="true" name="allowfullscreen"><param name="allowscriptaccess" value="always"><embed height="500" align="middle" width="630" pluginspage="http://www.macromedia.com/go/getflashplayer" type="application/x-shockwave-flash" name="reader" src="http://wenku.baidu.com/static/flash/apireader.swf?docurl=http://wenku.baidu.com/play&amp;docid=a94dd5dbd5bbfd0a795673a6&amp;readertype=external" allowfullscreen="true" wmode="window" allowscriptaccess="always" bgcolor="#FFFFFF" ver="9.0.0"></embed></object>
