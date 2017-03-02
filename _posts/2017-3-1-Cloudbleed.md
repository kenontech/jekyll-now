---
layout: post
title: 一个等号引起的血案
---
----
晓勇
版权所有 欢迎转发
----

2017年2月17号，Google零号项目的一个成员告知Cloudflare，他们发现了Cloudflare的一个安全漏洞。这个漏洞会导致大量敏感数据的泄漏，这些敏感数据包括用户名，密码，文字，照片，视频，等等。受影响的网站多达3400家，包括Uber，FitBit，OKCupid等公司。当你浏览这些公司网站时，有可能你看到的不是正常网页，而是含有敏感数据的页面。就好比你去参加考试，发给你试卷背面写满了考试答案。业界将这个事故称作Cloudbleed（云出血）

Cloudflare的安全漏洞为什么会引起这么多公司网站的信息泄漏？Cloudflare是一家网络加速和安全公司，它的客户包括上述公司，以及Nasdaq，BainCapital，Cisco等。当用户访问这些客户的网站时，流量被导入Cloudflare分布在全世界的数据中心，在这些数据中心里，流量被处理，相应的回答或者被发回给用户，或被发给客户的服务器进一步处理。如果Cloudflare的软件出错了，很有可能它的所有客户的网站都被影响。幸运的是，因为造成这次泄漏的软件只部署在部分特性中，所以只有部分客户受影响。

这么严重的一个安全事故，是由什么引起的呢？Cloudflare接到安全漏洞报告后， 立即组织人员进行调查，调查结果令人哭笑不得。原来有一行对终止条件进行检查代码，本来应该是判断当前指针是不是“大于等于（>=）”被处理文件的结束位置，结果写成了判断当前指针是不是“等于（==）”被处理文件的结束位置，一个符号的差别，造成缓冲区溢出（Buffer Overflow），结果就是云出血不止。

这行代码是这样的：
```
if ( ++p == pe )
    goto _test_eof;
```
本来应该是这样：
```
if ( ++p >= pe )
    goto _test_eof;
```
这个软件错误是不是似曾相识。1962年美国宇航局（NASA）水手一号项目失败了，其原因就是引导程序里有个地方少写了一个横道（“-”），导致推进器故障，最终水手一号飞船被迫自爆。

实事求是地讲，Cloudbleed这个软件虫（bug）情况没有上面说得这样简单和直接。我们上面看到的代码其实是软件自动生成的。为了达到高速处理网页的目的，Cloudflare用了Regal。工程师写的是Regal代码，这些Regal代码然后被编译成C代码，最后被编译成二进制代码执行。

相应的Regal代码是这样的：
```
script_consume_attr := ((unquoted_attr_char)* :>> (space|'/'|'>'))
>{ ddctx("script consume_attr"); }
@{ fhold; fgoto script_tag_parse; }
$lerr{ dd("script consume_attr failed");
       fgoto script_consume_attr; };
```

是不是有点晕？初一看，我也有点。咱们从头开始，现在的互联网的网页都是用HTML来描述。这段代码是Cloudflare的HTML解释器。它的输入就是HTML的网页。比如，以下是一个非常简单的HTML页面：
```
<!DOCTYPE html>
<html>
<head>
	<metadata charset=”utf-8”>
</head>
<body>

<h1>标题</h1>

<p>段落内容</p>

</body>
</html>
```
HTML页面由各种标签（在<>之间的文本，比如<h1>，<p>等）组成，每个标签可以有零或多个属性，比如这个页面中<metadata>有一个属性，叫charset。

在浏览器上，这个HTML页面会是这样的，所有的标签都被浏览器解释了。

![_config.yml]({{ site.baseurl }}/images/SimpleWebPage.png)

我们刚才看到的Regal代码是用来解释<script>标签的。我们再看一下：
```
script_consume_attr := ((unquoted_attr_char)* :>> (space|'/'|'>'))
>{ ddctx("script consume_attr"); }
@{ fhold; fgoto script_tag_parse; }
$lerr{ dd("script consume_attr failed");
       fgoto script_consume_attr; };
```

第一句的意思是说，这个<script>标签里应该有零或多个unquoted_attr_char，后面应该跟空格，‘／’，或者‘>’。
如果HTML页面写对了，一个属性格式正确，那么就应该进入以下处理：
```
@{ fhold; fgoto script_tag_parse; }
```
如果有一个属性格式错误，那么就应该进入以下处理：
```
$lerr{ dd("script consume_attr failed");
       fgoto script_consume_attr; };
```

看起来都没什么问题，唯一的差别是，格式正确处理里有调用fhold，而格式错误处理里没有调用fhold。

在Regal程序内部，有一个指针p，它被用来指向当前正在处理的字符。fhold干什么用的？它就是将p这个指针往回移一个位置（相当于C语言里的p--）。上面代码第一句执行后，如果属性格式错误，p应该指向引起错误的第一个字符。比如，以下的<script>标签中type属性的格式是错误的，p就会指到=之后的那个位置。
```
<script type=
```
问题就来了，如果这个标签正好是整个HTML页面的最后部分，p就会指到页面之后，那么，刚开始我们看到的 if ( ++p == pe ) 因为p已经比pe大了，也就是说p已经指到文档结束字符之后了，所以检查就不起作用。如果后面的程序继续往缓冲区里写，就会造成缓冲区溢出。

改正这个错误很简单，就是在属性格式错误的处理也调用fhold。正确的程序应该是这样的：
```
script_consume_attr := ((unquoted_attr_char)* :>> (space|'/'|'>'))
>{ ddctx("script consume_attr"); }
@{ fhold; fgoto script_tag_parse; }
$lerr{ dd("script consume_attr failed");
       fhold; fgoto script_consume_attr; };
```
故事到这里没有结束，上述Regal代码，已经存在好几年了，一直没有问题，为什么最近突然出了问题？原来，在一年多以前，Cloudflare觉得Regal代码太复杂，所以它开始开发一种新的HTML解释器。这个叫cf-html的解释器被开发出来后，就开始使用到一些特性中，包括HTTP自动重写，邮件伪装等3个特性。其中邮件伪装这个特性在2月13日被升级过，正是这个特性引起了大部分的数据泄漏。

其实cf-html这个解释器本身没有问题，相反，它修正了原来Regal写的解释器的一个软件错误。但问题就在这里，原来的Regal解释器虽然错了，与它配套的特性代码也错了（就是我们前面看到的在属性格式错误时没有调用fhold的代码），但因为Regal解释器的错误，上述代码的$lerr分支根本不会被调用，所以这个分支里有没有fhold都没关系。可是，cf-html修正那个软件错误后，再碰到HTML属性格式错误后，$lerr分支就会被调用了，这时，有没有fhold就至关重要了。你看，有时候做比不做还糟糕。
整个事故暴露了几个问题：一个是原有代码的问题，缺乏足够的负面测试实例，没有好的代码检查工具，也没有代码覆盖率的测试。另一个是新的代码，改变了模块的行为，也就是改变了与其它模块的接口，但因为这个接口是隐含在一个复杂的数据结构中，根本没有人注意到。开发人员可能还为自己修了一个软件虫而高兴，但没有想到接口的变化会产生如此大的后果。

【参考文献】
[Cloudbleed事故报告](https://blog.cloudflare.com/incident-report-on-memory-leak-caused-by-cloudflare-parser-bug/)
