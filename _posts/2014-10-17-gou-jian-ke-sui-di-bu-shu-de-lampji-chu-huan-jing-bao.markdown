---
layout: post
title:  "构建可随地部署的LAMP基础环境包"
date:   2014-10-17 00:15:04
author: wangweibing
---


## 背景

LAMP是Web开发中最流行的模式，即Linux + [Apache](http://httpd.apache.org/) + [Mysql](http://www.mysql.com/) + [PHP](http://php.net/)。近年已经有一些变化，比如webserver除了Apache外，还有[Nginx](http://nginx.org/)和[Lighttpd](http://www.lighttpd.net/)；数据库除了Mysql外，还有各种NoSQL引擎；PHP引擎除了Zend之外，还有[HHVM](https://github.com/facebook/hhvm)。所以LAMP不再指的是具体软件的集合，而是Linux + webserver + php语言 + db这种开发模式。

在百度内部，LAMP也得到了广泛的应用。但是，对于一个新的开发者而言，自己搭建一套完整的LAMP环境，成本还是很高的：

1. 自己编译PHP/HHVM，编译各种PHP/HHVM扩展（开源 or 百度内部）
1. 自己编译Nginx/Lighttpd，包括各种Nginx/Lighttpd扩展（开源 or 百度内部）
1. 自己下载各种PHP基础库（开源 or 百度内部）
1. 自己搭建各种数据库（开源 or 百度内部）

就算有最完善的文档，对于一个新手，也得起码一天才能把这整个环境搞定吧。

因此，我们需要一个很方便部署的LAMP基础环境包，来减少这种重复搭建的成本。

由于数据库是有状态的，通常和web服务不同机部署，所以我们的LAMP环境包暂不包括数据库。

## 目标

我们对LAMP基础环境包有几个要求：

1. **免编译**，编译一方面很耗时，另一方面对目标机器的编译环境有依赖，容易因为编译环境的问题导致安装失败
1. **免root**，百度内部的测试机，大部分是多个账号共享一个机器，普通账号拿不到root权限
1. **支持多种Linux发行版**，比如RedHat,Ubuntu,CentOS，这些都有人用
1. **可以安装到任意目录**，不同用户的工作目录不一样，无法事先预知
1. **同一机器可以安装多套环境**，一方面多账号，另一方面同一个账号也有搭建多套环境的需求
1. **支持任意移动**，安装后，仍可以把整个环境打包移动到其它机器其它目录运行，方便各种机器迁移、线上线下环境同步、定制自己业务的环境包等。
1. **性能不能有明显损失**，相比原来自己编译的方案，性能损失应该在`1%`之内。

## 方案

### 路径替换方案

上述目标中，最关键的问题是如何实现**免编译任意路径安装**，因为编译的时候指定的prefix是固定的，所以要实现免编译安装，必须有一个路径替换的方案。有没有一种通用的路径替换方案呢，是有的：

1. **虚拟机**，虚拟机可以满足免编译、任意系统、任意目录的要求，但是安装虚拟机需要root权限。也有一些不需要root权限的虚拟机，如[qemu](http://wiki.qemu.org/)，但严重影响性能。
1. **cgroup相关技术**，如[docker](https://www.docker.com/)，但这仍然需要root权限。
1. **用户态虚拟化技术**，如[fakechroot](https://github.com/dex4er/fakechroot)，通过LD_PRELOAD去hook libc.so相关函数实现来虚拟化路径，但是有两个缺点，一是对性能有一定影响，二是稳定性较差，比如[fakechroot和jemalloc有冲突](https://github.com/dex4er/fakechroot/issues/21)会导致HHVM死锁。

如此看来，通用的路径替换方案是行不通的，但是我们可以根据具体软件的特点，来实现各自的路径替换方案：

#### 配置文件中的路径替换
这是最简单的，因为配置文件是纯文本，只要通过`sed`命令，把配置文件中以前的路径替换成当前路径即可。

但是，也有些配置文件格式不能简单的做sed，比如`pear.conf`，是php serialize格式，必须先反序列化，再替换，再序列化。这类文件不多，单独处理一下即可。

#### PHP路径替换

PHP在编译的时候，会根据编译时指定的prefix路径生成头文件，定义`PHP_PREFIX`、`PHP_CONFIG_FILE_PATH`等宏，并在启动时注册为PHP常量。

PHP默认会在`PHP_CONFIG_FILE_PATH` 和`PHP_CONFIG_FILE_SCAN_DIR`指定的路径去查找ini配置文件，想让PHP加载指定的配置文件，其实也是有办法的：

```bash
export PHP_INI_SCAN_DIR=$php_home/etc/ext
$php_home/bin/php -c $php_home/etc/php.ini "$@"
```

但是这种方式有两个缺点，一是执行php的命令变复杂了，需要用一个shell脚本来包装，二是代码里定义的`PHP_PREFIX`常量并没有替换，这会导致如pear这样的工具不可用。

最终，采用了修改PHP源码的方案，启动时，通过`readlink('/proc/self/exe')`获得可执行程序路径，优先到当前路径的相对路径查找配置文件，并且根据这个路径注册`PHP_PREFIX`等常量。

能加载指定的ini后，在ini里面指定各种配置项如`error_log`, `include_path`,`extension_dir`的路径，就可以把php所用的所有路径都覆盖了。

#### HHVM路径替换

HHVM编译并没有提供configure脚本，也没有提供指定prefix的方法，也不会定义`PHP_PREFIX`常量，所以倒是比php省事，只需要启动时指定配置文件即可：

```bash
$hhvm_home/bin/hhvm -c $hhvm_home/conf/hhvm.hdf "$@"
```

需要包个脚本，但因为HHVM本身是个新东西，不需要兼容过去的用法，所以直接让用户都通过脚本启动HHVM就好了，脚本可以命名为`hhvm`，真正的hhvm可执行程序改名为`hhvm_bin`。

同样的，在HHVM的配置里，指定各种路径如`Log.File`, `Server.DocumentRoot`, `Server.FileSocket`, `PidFile`, `Repo.Central.Path`, `Debug.CoreDumpReportDirectory`等，就可以覆盖HHVM所用的所有路径了。

#### Nginx路径替换
Nginx可以通过运行时参数指定prefix，如

```bash
$nginx_home/sbin/nginx -p $nginx_home
```

但这样有个问题，就是nginx在读指定配置文件前，就会尝试按编译时路径创建日志文件，先打印出一条日志文件不存在的warning。虽然不影响最终运行，但还是会让用户困惑。

最终采用编译时指定prefix为相对路径的方式：

```bash
./configure --prefix=../
```

这样，被硬编码到nginx可执行文件中的prefix就是`../`，启动nginx时，反正都是通过脚本启动的，先cd到`$nginx_home/bin`，再启动，这样prefix就是`$nginx_home`了。

然后在配置文件里面设置好各种`error_log`,`pid`,`access_log`,`xxx_tmp_path`,`root`的路径就可以了。

#### Lighttpd路径替换

Lighttpd同样可以通过参数指定配置文件路径，如

```bash
$lighttpd_home/bin/lighttpd -f $lighttpd_home/conf/lighttpd.conf -m $lighttpd_home/lib
```

这种方式不会有什么副作用。

### 解决动态库依赖问题

解决了路径替换问题之后，还有一个大问题是动态库依赖问题，因为我们需要支持多种Linux发行版，各种版本系统自带的动态库有的缺失，有的版本不匹配。因此，我们的LAMP环境包需要能自带依赖库。

自带依赖库之前也有人做过，比如在[这篇文章](http://chenyufei.info/blog/2012-09-14/packaging-linux-applications/)中就介绍了一些方法，我们这里采用也是类似方法。

#### 找出依赖库

首先要知道我们程序依赖于哪些动态库，有几种方法：

1. ldd命令，如`ldd some_bin`
1. ldd不能找出运行时使用`dlopen`方式打开的动态库，可以使用strace做动态跟踪，如`strace -f some_bin 2>&1 | grep 'open.*so' | grep -v ENOENT`
1. 考虑到未来的扩展需求，我们把系统自带lib里面的一些常用lib都带上，用`/sbin/ldconfig -p`找出全部，再去掉一些不常用的，最终总共打包后也就20M，不算大。

#### 运行时加载指定动态库

把依赖库打包自带后，我们需要在运行时，指定加载自带的动态库，有几种方法：

1. 使用`LD_LIBRARY_PATH`环境变量，如`LD_LIBRARY_PATH=$my_lib $my_exe $args`，但这种方式，一来需要增加shell脚本来包装命令，二来会影响该程序启动的所有子进程（比如通过`system`函数调用系统的shell，系统的shell并不应该加载`LD_LIBRARY_PATH`指定的动态库）
1. 编译链接时，设置rpath，如`gcc -o $my_exe -Wl,--rpath='$ORIGIN/../lib'`，这里的`$ORIGIN`，会在加载动态库时被自动替换成可执行程序的所在目录，只要我们保证可执行程序与依赖库的相对路径不变，不管绝对路径在哪都能正常加载。
1. 编译链接后，通过[patchelf](http://nixos.org/patchelf.html)工具修改rpath，如`patchelf --set-rpath $my_lib --force-rpath $my_exe`

最终采用的是方案2，其实方案3也可以，只不过安装时需要多一步操作。设置rpath后，可以通过`readelf -d $my_exe | grep RPATH`来查看rpath。

如果你做到这一步，就以为ok了，这时你打包到一个不同的Linux发行版去执行程序，很有可能会遇到core dump，因为有一个so，并不能通过rpath或`LD_LIBRARY_PATH`来指定，那就是`/lib64/ld-linux-x86-64.so.2`（在x64架构下是这个名字，在32位架构下是`/lib/ld-linux.so.2`)。

这个so是干什么的？这个so的功能是实现动态库的加载，不妨称之为linker。也就是说，加载其它所有so的逻辑，都是在这个linker里实现的。

那么linker又是谁加载的？linker是执行exec系统调用时，由内核先加载的，加载完它之后，再加载可执行文件本身，再加载其它的动态库，它比可执行文件还先加载！

如何指定这个linker呢？有几种方法：

1. 通过`/lib64/ld-linux-x86-64.so.2 $my_exe $args`的方式运行可执行程序。缺点一是需要加shell脚本包装命令，二是进程名变成了`ld-linux-x86-64.so.2`，这对于ps命令或者其它依赖于可执行文件的名字、路径的逻辑都会影响。
1. 编译链接时，通过参数指定linker，如`gcc -o $my_exe -Wl,--dynamic-linker=$my_linker`,但是linker不支持像`$ORIGIN`这样的语法，只能设置实际路径，不能满足任意路径安装的需求。
1. 编译后，可以使用patchelf工具来修改linker，如`patchelf --set-interpreter $my_linker $my_exe`。

最终采用了方案3，安装时设置linker。设置后，可以通过` readelf -l $my_exe | grep interpreter`查看linker。

这里还需要考虑一种情况，就是服务上线之后，可能会有单独升级可执行程序的情况，所以执行patchelf的时机，不仅是初次安装时，而且在每次通过脚本启动/重启服务时，都需要检查当前的linker是不是正确的，如果不是，可能是升级过程序，需要重新设置一下。

### 支持任意移动

做完上面的事情，我们的目标大部分都能实现，还有一个目标就是支持任意移动。

其实在上面基础上，再支持任意移动很简单：

1. 安装时，记录当前安装路径到文件
1. 每次执行服务启停脚本时，判断当前路径是否与记录的路径相同，如果不同，说明目录可能被移动，或者被整体打包到其它机器，这时需要执行上面说的路径替换操作
1. 替换完路径后，更新记录的安装路径

## 效果

基于上述方案实现的免编译安装包，和我们以前基于现场编译的安装包，效果对比如下：

1. 平均安装耗时： 10分钟 -> 30秒
1. 支持的系统种类： 1种 -> 3种以上
1. 安装成功率： 98% -> 99.9%以上
1. 任意移动：不支持 -> 支持

