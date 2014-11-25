---
layout: post
title:  "动态语法的性能问题分析"
date:   2014-10-17 00:15:04
author: huzhiguang
tags:
  - HHVM
  - 动态语法
  - 性能问题
---


> [百度Lamp技术博客](/) 原创作品，转载时请务必以超链接形式标明文章 [原始出处](http://lamp.baidu.com/2014/10/16/dong-tai-yu-fa-de-xing-neng-wen-ti-fen-xi/) 、作者信息和本声明。否则将追究法律责任。

##背景
在某业务线使用HHVM的过程中，发现有一些机器的HHVM CPU使用率异常于其他机器，使用率高出了一倍多，上线和流量高时CPU高出更多，所以针对此问题定位和分析是哪里造成了此问题。
###线上问题分析定位
首先我们通过hhvm的监控接口(HHVM 的admin server访问，check-health和vm-tcspace)获取了unit、funcs、tcspace、load和queued等信息；
但是发现这类机器都有统一的问题，那就是在不上线的时候unit、func、tcspace涨幅过快，一般情况虽然有未触发文件，在请求时会编译+翻译小部分文件但是并未有过这么大幅度，几乎一天tcspace就超过了91%阀值，所以分析应该是动态函数的调用引起了此问题，由于线上使用了smarty模板，smarty中会使用eval和create_function，所以可能造成此问题，然后跟进线下的代码尝试是否可以复现此问题。

首先通过线上的datablock造成的core的栈来跟进问题栈内容如下：

````
#0  include() called at [xxxxxx/libs/Smarty/sysplugins/smarty_internal_template.php:432]
#1  Smarty_Internal_Template->renderTemplate() called at [xxxxxx/libs/Smarty/sysplugins/smarty_internal_template.php:xxx]
#2  Smarty_Internal_Template->getRenderedTemplate() called at [xxxxxx/xxxxx.file.page.tpl.php:xxx]
........
```

而且上面的core查看hhvm的栈一般都是jit cache溢出的栈，而且这种栈很多而且都暴露到了一个位置；

首先通过业务线RD定位是某个模板触发后才会出现此种场景，而非所有模板，就是我们上面看到的栈的内容：
xxxx.file.page.tpl.php

每次当落到这个栈的时候会出现这个问题，然后涨jit cache每次query都能命中肯定是和eval和create_function有关；
后来通过栈中发现其中一个位置中有调用eval:
xxx/libs/Smarty/sysplugins/smarty_internal_template.php:432 (renderTemplate函数)
{<1>}![](/content/images/2014/10/smarty.jpg)
然后将eval注释掉后发现jit cache不涨了，再打开继续增长，可以定位是此处造成了hhvm的jit cache增长；
然后打印了$this->compiled_template的值发现是如下内容：
{<2>}![](/content/images/2014/10/template.jpg)
红框位置是动态内容，所以每次调用请求是$this->compiled_template都会生成一个新的值，所以hhvm就判断了这个内容是有修改了，所以就重新编译+翻译了，这样jit cache就涨了上去,这段代码在每次query调用了7次，也会影响每次访问的性能，所以会造成请求变慢和cpu增加（由于有多次编译+翻译，这样比解释执行都要慢了），所以此种用法应该也和cpu使用率过高有关系；

**解决方式：**

$this->compiled_template里面的动态值，如时间，hash内容通过变量代替，变量在请求的调用入口处声明，可以让eval内的值识别到，这样就可以做到eval的值是固定的，并且可以做到jit也提高了执行效率；
但是php语法中其实不建议使用eval，因为此语法本来就会影响性能；

**线上验证：**

线上验证，去掉eval后，从之前的200ms-600ms的请求降低到了0.8ms，提升了约1000倍的性能（代码中多处调用了eval）

下面我们手动构造动态函数从源码的角度上分析此问题；

##动态函数分析
首先我们不建议使用eval和create_function这类动态函数，因为这些动态函数不仅会影响php的性能，而且在hhvm中会造成编译+翻译每次都进行，这样就跟解释语言一样或者更差；
###eval
Eval主要是构造一段php代码，然后直接执行，如：
```php
$time = gettimeofday(true);
//会造成jit cache上涨
eval ("?><?php echo ".$time.";?>");
//不会造成jit cache上涨
eval ("?><?php echo \$time;?>");
```

上面的2中构造的eval的内容其实实现的结果是一致的，但是第一种就会造成jit cache上涨，而第二种则不会这是为什么呢？

我们将上面的代码展开看一下

第一种展开

```php
?>< ?php echo 1413185434.4592?>
```

上面的1413185434.4592这个值并不是固定的，而是每次调用php时获取的$time的动态值，所以每次eval的值就一样了，而这个中会构造一个伪主函数，而代码出现了不一致，则判断有了修改，所以就重新编译+翻译了，所以就造成了jit cache的上涨a.code和astub.code上涨；（由于jit 翻译是通过判断func是否有修改进行判定的，而编译阶段则是通过文件是否修改进行判断的）

第二种展开：
```php
?>< ?php echo $time;?>
```

第二种执行的结果和第一种情况是一致的，而第二种则不会造成jit cache的上涨，这是未什么呢？因为第二种虽然首次进行eval也会翻译，但是其中的内容是固定的，所以每次判定时就不需要重新编译和翻译了；

**解决方式：**

所以我们在使用时需要注意，不要在eval中产出会<span style="color:red">动态出现的值</span>，这样很危险，如果想使用动态值，最好在eval使用的上面比如时间类的动态内容通过<span style="color:red">变量</span>的方式放在eval语句中，这样既可以使动态语法jit 又不上涨jit cache;

####eval实现

下面是eval的执行时的调用栈：
```
#0  HPHP::VMExecutionContext::compileEvalString 
#1  HPHP::VMExecutionContext::iopEval 
#2  HPHP::VMExecutionContext::opEval 
#3  HPHP::Transl::interpOneEval 
#4  0x00000000078000c8 in ?? ()
#5   enterTCHelper ()
#6   HPHP::Transl::TranslatorX64::enterTC
#7   HPHP::Transl::TranslatorX64::enterTCAfterPrologue
#8   HPHP::VMExecutionContext::enterVMWork
#9   HPHP::VMExecutionContext::enterVM 
#10  HPHP::VMExecutionContext::invokeFunc 
#11  HPHP::VMExecutionContext::invokeUnit
```

**iopEval实现：**

{<3>}![](/content/images/2014/10/eval1.jpg)

evalFilename是eval的包含文件+行号作为eval 的文件名，如：
<pre>
/mnt/huzhiguang/data/test/test_create_function.php(8) : eval()'d code
</pre>

然后通过compileEvalString函数进行eval的编译；

**compileEvalString实现：**
{<4>}![](/content/images/2014/10/eval2.jpg)

上面篮框处是执行eval编译的代码；
EvaledUnitsMap 是eval unit map类型（key是eval的代码，value是unit）
s_evaledUnits 是保存已经eval 的unit 的静态变量；

然后每次编译前会去s_evaledUnits中进行insert如果insert成功则表明之前没有这段code,所以进入分支进行编译环节；

<span style="color:red">
**注：**</span>
<span style="color:red">
如果是动态代码就每次都进入此环节进行编译，这样是相当影响性能的；
</span>

###create_function
create_function 和 eval是类似的都是动态语法，但是create_function其实比eval使用起来更加可怕，如果不加使用限制不仅仅影响性能，而且还会造成内存泄露这样的问题，除了内存泄露还会造成jit cache的猛烈增长；
下面我们看下create_function的实现：
{<5>}![](/content/images/2014/11/create_function.jpg)

首先我们看下create_function,这里只要进入函数，那么每次都会进行编译，这块还不如eval函数，至少eval函数每次还会在s_evaledUnits中判断是否有此code,但是其实就算把create_function 的内容也放入到一个map中去保存起来，只要是动态的构造都会出现问题的；
其实如果只是create_function的话只是构造函数，但是只要调用话就会增加jit cache了，因为这里的调用每次的函数都是新的functionid,所以在jit判断时就找不到这个函数了，认为就更改了，所以就每次都翻译了（因为判断函数是否变更是和funcId和offset有关，而这里每次都会构造新的func，所以 jit也会每次增加），<span style="color:red">这样的调用是很危险的(每次都编译+翻译)</span>，性能会很低，比eval还要危险，<span style="color:red">所以不建议用create_function</span>;
其实create_function也可以按照eval那种模式，将code和unit放到一个静态全局变量中，但是那样也防不住动态内容的，<span style="color:red">所以还是不建议用的</span>；

create_function测试代码：
<pre>
//每次调用编译compile_string
$newfunc = create_function('$a,$b,$c', 'return $a.$b.$c;');
//每次由于function是lambda的，所以functionid都不同，所以这里每次都做翻译
echo $newfunc(2, 3,"aaa") . "\n";
</pre>

当我们每次调用上面的函数时，我们通过监控的几个指标：
check-health 监控下funcs每次执行都会上涨
vm-tcspace 监控中会发现a.code和astubs.code 每次都会上涨
从上面观察看，其实每次function都编译，都在涨，而且jit也每次都在翻译，所以从原理和监控上来讲这个都是十分危险的；

###综上结论

虽然eval和create_function方便了我们的使用，但是动态函数的动态性却极大的影响了性能；
但是从hhvm的角度上来讲eval 如果不是动态值，其实是可以用的(只要内容相同就不重新编译+翻译)，但是create_function建议大家放弃使用，因为create_function每次都会进行编译+翻译（不管内容是否相同）；

后面我们会对eval和create_function加上监控和warning方便大家进行trace问题和监控此问题的使用；

> [百度Lamp技术博客](/) 原创作品，转载时请务必以超链接形式标明文章 [原始出处](http://lamp.baidu.com/2014/10/16/dong-tai-yu-fa-de-xing-neng-wen-ti-fen-xi/) 、作者信息和本声明。否则将追究法律责任。
