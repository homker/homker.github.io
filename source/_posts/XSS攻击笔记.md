title: "XSS攻击笔记"
date: 2015-06-09 15:42:26
tags: XSS
---
#XSS（Cross-site scripting）

中文名叫跨站攻击脚本,这个攻击方式很是厉害。他经常会出现于一些，你可能永远都想不清楚的地方，然后，突然给你一下，让你不知所云，然后莫名其妙的中枪。

# 攻击原理

基于js的编译即执行的机制，和其不完善的脚本权限控制，我们很容易构造一个基于js的攻击payload。嗯，google给了一份关于web app 的XSS的介绍，可以点击[这里][1]获取。嗯，说道xss攻击，其本质在于其的脚本能力，精悍短小而不易发现，同时，利用脚本的映射性，将攻击性变得特别的诡异。嗯，讲攻击分类之前，先说几个概念性比较强的东西。

## 反射

> 在计算机科学中，反射是指计算机程序在运行时（Run
> time）可以访问、检测和修改它本身状态或行为的一种能力。用比喻来说，那种程序能够“观察”并且修改自己的行为。要注意反射和内省（type
> introspection）的区别。

以上内容来自[wiki][2]。说白了，就是程序自己修改自己。

## 映射
不知道从哪里看到的理解，感觉有点道理，放进来。不是学数学出身，所以并不是很清楚在纯数学里，这个东西是怎么定义。找了很多资料，但是并不能很明确的找到一个官方的定义。wiki中给出的数学概念，映射是函数的一种，看[这里][3]，和[这里][4]。个人的理解就比较简单啦，把一个函数（function）A的运行环境放到另一个函数B的运行环境下面运行，则我们定义为，A映射在B下。并不知道这个理解对不对啊。那么回到XSS里面来，简单的说，就是一些攻击代码被映射到了正常代码下面，在正常代码的运行环境中运行了攻击代码，导致了一些不可控的事情的发生。



## 编码
[编码][5]这个东西，说起来很奇怪哈，从我和php打交道开始，它就一直萦绕在我的身边根本甩不开，而且很麻烦啊。什么GBK UTF-8 GB2312 ，但是这些都是我们常用的编码集。这些编码的根基是ASCII码表，这个并不需要我多说什么，但是我们一般看到的ASCII码表示是十进制的，但是，计算机的编码不是只有这一种啊，还有八进制，十六进制啊。当然，上面说的都是正统的编码，也就是通用的编码，还有一部分叫做html实体的编码格式则更是让人抓狂。介绍内容看[这里][6]。这些实体的本来目的是为了防止html解析出现混乱，但是后来变成了各种payload的利用对象。还有一堆的js编码格式，哦，对了，还有url base64编码嗯，反正这个世界语言一多，就各种编码漫天飞。对应到xss当中，我们在过滤标签的和关键的时候，各种编码就让你目不暇接，分分钟想着，你妹，这TM也行。嗯，水平有限，就不多说了，有兴趣的同学，可以参考[这里][7]。
<!--more-->
上面对两个可能会产生理解障碍的词汇做了一个自己的理解，然后，我们回头头来看XSS的攻击。还是举个例子来的痛快。
首先，我们先来单纯的写一个将url里面的query信息输出在页面中。

- index.html

```
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>test</title>
</head>
<body>
<div class="content">

</div>
<script type="application/javascript" src="script/app.js"></script>
</body>
</html>
```

- app.js

```
(function(){
    function getquery(){
       var query = {},
           url = document.documentURI||location.href,
           url = url.split("?")[1];
        if(!url) return;
        for(var i = 0; i< url.length; i++){
            var get = url[i].split("=");
            if(!get) return;
            query[get[0]] = get[1];
        }
        return query;
    }
    var content = document.querySelector(".content");
    for(value in getquery()){
        content.innerHTML += value +":"+getquery()[value]+"<br/>";
    }

})();
```
嗯，代码基本就是这样，然后我们看看怎么攻击。
最开始啊，我是想着，我们直接构造一个`<script>`就可以了嘛，应该不是很难，所以，就有了如下的url
```
http://localhost:63342/test/index.html?test=<script>alert('a');</script>
```
我以为会直接构造出，简单的XSS攻击，但是too young，too simple，chrome分分钟交你做人系列。
实际上，这条url被提交的时候，会是如下的状态：
```
http://localhost:63342/test/index.html?test=%3Cscript%3Ealert(%27a%27)%3C/script%3E
```
被url编译了一次，ie的话，直接整个就给你过滤掉了。。。心好塞。。。但是，你会发现，这个好像是编码漏洞唉，如果输入的是中文或者特殊符号的话，就不能用了，所以代码段应该会有decodeURI才对，我们加上去，再来一发，我去，too young too~！这里还有浏览器的第二个机制[CSP][8]，所以，你期待的XSS还是不会发生。。。但是你会发现，yeah，这个script的脚本好像已经被渲染了，但是没有执行而已，但是如果是业务需要的话，它应该正常显示才对啊，嗯，我们对他进行一次html编码。就有了如下的代码。
```
(function(){
    function getquery(){
        var query = {},
            url = location.search,
            url = (url.indexOf("?") !== -1)?url.split("?")[1]:null,
            url = url&&(url.indexOf("&") !== -1)?url:url.split("&");
        console.log(url);
        if(!url) return;
        for(var i = 0; i< url.length; i++){
            var get = url[i].split("=");
            if(!get) return;
            query[get.shift()] = get.join("=");
        }
        return query;
    }
    var content = document.querySelector(".content"),
        query = getquery();
    for(value in query){
        content.innerHTML += value +":"+htmlencode(decodeURI(query[value]))+"<br/>";
    }
    function htmlencode(s){
        var div = document.createElement('div');
        div.appendChild(document.createTextNode(s));
        return div.innerHTML;
    }
    function htmldecode(s){
        var div = document.createElement('div');
        div.innerHTML = s;
        return div.innerText || div.textContent;
    }
})();



```
这下业务代码应该没有漏洞了，不管你输入什么，我们都可以在页面上显示出来，但是我们接着想想如何XSS，这个的编码是安全的吗？其实并不是，来，我们来个新的payload。
```
http://localhost:63342/test/index.html?test=%3Cimg%20src=%22%22%20onerror=%22alert(%27a%27)%22/%3E
```
这里使用img标签，通过他的onerror来触发XSS，简直防不胜防。。。那我们怎么办呢，嘻嘻嘻，这个东西，我们把代码改一下。
```
(function(){
    function getquery(){
       var query = {},
           url = location.search,
           url = (url.indexOf("?") !== -1)?url.split("?")[1]:null,
           url = url&&(url.indexOf("&") !== -1)?url:url.split("&");
        if(!url) return;
        for(var i = 0; i< url.length; i++){
            var get = url[i].split("=");
            for(var j = 0; j<get.length;j++){
                get[j] = htmlencode(decodeURI(get[j]));
            }
            console.log(get);
            if(!get) return;
            query[get.shift()] = get.join("=");
        }
        return query;
    }
    var content = document.querySelector(".content"),
        query = getquery();
    for(value in query)){
        console.log(query[value]+"A");
       content.innerHTML += value +":"+query[value]+"<br/>";
    }

    function htmlencode(s){
        var div = document.createElement('div');
        div.appendChild(document.createTextNode(s));
        return div.innerHTML;
    }
    function htmldecode(s){
        var div = document.createElement('div');
        div.innerHTML = s;
        return div.innerText || div.textContent;
    }
})();
```
这里的话，本身就有一个很奇葩的编码习惯，我们通过切割字符串来获取query的值，好像很不优雅，嗯，我们先看看优雅的写法。
```
function getQueryString(){
        var result = location.search.match(new RegExp("[\?\&][^\?\&]+=[^\?\&]+","g"));
        if(result == null){
            return "";
        }
        for(var i = 0; i < result.length; i++){
            result[i] = result[i].substring(1);
        }
        return result;
    }
```
~~优雅的写法下，我们很容易就发现他对于字符串的处理是大块大块的处理，这样的，就算整体编码还是会触发XSS，除非，你吃饱了撑得，不嫌麻烦的话，把字符做一个替换的话，也可以解决这个问题。但是，我们回过头来看下我们这种不优雅的写法，先把字符串按"="切割，这个地方我就已经不把它当可执行代码看了，然后先行uri解码和转码，这样的小段解码和转码之后，得到的是一堆html实体字符，他是不会运行的，就算你把他们拼起来。嗯，这里举了个小栗子啊，其实要破解的话，还是有办法的啊，能力有限，我就不深究了。~~
好吧，为以上的low bee 言论做出深刻的检讨，我们来看看更优雅的解决方法。

```
(function(){
    function getquery(){
        var __a={'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'},
            __b=/[&<>"']/g,
            query = {},
            result = decodeURI(location.search).match(new RegExp("[\?\&][^\?\&]+=[^\?\&]+","g"));
        console.log(result);
        if(result == null){
            return "";
        }
        for(var i = 0; i < result.length; i++){
            result[i] = result[i].substring(1).split('=');
            query[result[i].shift()] = result[i].join('=').replace(__b, function (m) {return __a[m]});
        }
        console.log(query);
        return query;
    }
    var content = document.querySelector(".content"),
        query = getquery();
    for(value in query){
        content.innerHTML += value +":"+query[value]+"<br/>";
    }
})();
```


#XSS分类
上面一大段用一个例子和一堆的说明性的文字介绍了XSS是什么，下面我们简单的讲一下分类。

- 反射型XSS
- DOM XSS
- 持久型XSS

## 反射型XSS
将你的代码通过某种手段映射到运行环境，修改了原先的代码，这样的话，构成了XSS。举个栗子。
```
//index.php
<?
    if($_GET['s']){
        echo $_GET['s'];
    }
?>
```
我们很容易构造如下的payload进行攻击。
```
http://localhost/XSS/index.php?s=%3Cscript%3Ealert(%271%27)%3C/script%3E
```
前提是，你不会遇到CSP。

## DOM XSS
通过构造DOM节点来实现XSS攻击。举个栗子啊。
```
<form action="./" method="post">
	<input class="xss" name="name" type="text" value="<?php echo @$_POST['name'] ?>" />
	<input type="submit" value="submit"/>
</form>
```
比如这样，构造的话，在input框里输入
```
" onclick="alert(1)"
```
额，前提也是没有CSP的啊，不然这些简单的payload都是会被拦截的。
通过如上的构造，会有如下的代码效果。
```
<input class="xss" name="name" type="text" value="" onclick="alet(1)" />
```
嗯，是不是感受到了这个世界满满的恶意，人和人最基本的信任在哪里，居然还敢说我是个好人。。。

## 存储型XSS

当攻击代码被存储到了数据库上，当从数据库中读取的时候，在渲染的时候触发了DOM XSS构成的XSS攻击，叫做存储型XSS。不单独举例子了，各种例子满大街都是啦。

# 防御方法

首先啊，我觉得的啊，这个世界就没有什么一劳永逸的事情，所有的对抗，都是不断升级的，针对不同的业务场景进行不同的具体的分析之后的XSS的防御才是靠谱的啊，如果非要拿出一个万用的XSS防御，我觉得这个比较天真啊。但是，是不是所有的XSS都要单独处理啊？肯定不是啊，世界有个性也有共性的嘛，先把共性打碎，再来研究个性嘛。嗯，所以，以下讨论的都是针对XSS共性的处理方法，并不是万用的处理方法，针对不同的XSS攻击还是要有针对性的进行分析后得出具体的个性化的解决法案。一下的解决方案，都是基于js的啊，php和其他后台语言不在讨论之列，但是理论是想通的啦，使用的库不一样而已啦。

## 过滤输入

http本来就是无状态的协议，~~你不知道我在干嘛，我也不知道你在干嘛，我为什么要天真的认为，你会爱我？~~所以，不要相信所有的输入，就算她是你暗恋了很久的妹子，你永远不知道她会对你做什么。。。

过滤的话，分白名单过滤和黑名单过滤，意思的话，都很好理解，不多说了，嗯，这里给一个github上开源的白名单过滤方案，[看这][9]。将一些不要的东西进行过滤，使之能够干净的存在于你需要的地方。

这里还有一个问题，如果，你在前端过滤了一遍，后端是不是还要再过滤一遍。这个还是要的啊，要保持职业严谨性和职业的怀疑态度。~~审计学得真好~~

## htmlDecode

在最开始的方法里，我们使用的方法就是html编码，将js代码变成html实体编码，以此来避免运行。这里给出两种编码方式。
一种是黑名单变换
```
private static String htmlEncode(char c) {
    switch(c) {
       case '&':
           return"&amp;";
       case '<':
           return"&lt;";
       case '>':
           return"&gt;";
       case '"':
           return"&quot;";
       case ' ':
           return"&nbsp;";
       default:
           return c +"";
    }
}
 
/** 对传入的字符串str进行Html encode转换 */
public static String htmlEncode(String str) {
    if(str ==null || str.trim().equals(""))   return str;
    StringBuilder encodeStrBuilder = new StringBuilder();
    for (int i = 0, len = str.length(); i < len; i++) {
       encodeStrBuilder.append(htmlEncode(str.charAt(i)));
    }
    return encodeStrBuilder.toString();
}
```
嗯，上面那个其实是个low bee 我们看这个才是王道.
```
var __a={'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'},
      __b=/[&<>"']/g;

return str.replace(__b, function (m) {return __a[m]});};

```
嗯，这个针对常见的不干净的字符进行转换，然后替换。还有就是利用一点小技巧啦。
```
function htmlencode(s){
        var div = document.createElement('div');
        div.appendChild(document.createTextNode(s));
        return div.innerHTML;
    }
    function htmldecode(s){
        var div = document.createElement('div');
        div.innerHTML = s;
        return div.innerText || div.textContent;
    }
```
参考这个[博客][10]。
嗯，基本上就是这些了，明天有空的话，记一下常用的攻击平台。


  [1]: https://www.google.com/about/appsecurity/learning/xss/
  [2]: http://zh.wikipedia.org/wiki/%E5%8F%8D%E5%B0%84_%28%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6%29
  [3]: http://zh.wikipedia.org/wiki/%E6%98%A0%E5%B0%84
  [4]: http://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B0
  [5]: http://zh.wikipedia.org/wiki/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BC%96%E7%A0%81
  [6]: http://www.w3school.com.cn/html/html_entities.asp
  [7]: http://drops.wooyun.org/tips/689
  [8]: https://chrome-apps-docs-chs.appspot.com/extensions/contentSecurityPolicy.html
  [9]: https://github.com/leizongmin/js-xss
  [10]: http://www.cnblogs.com/ghd258/archive/2009/10/18/1274429.html
