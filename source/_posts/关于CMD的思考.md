title: "关于CMD的思考"
date: 2015-05-17 16:22:00
tags: CMD seajs
---

最近在学习[`sea.js`][1]的用法，sea.js是一个基于[`CMD`][2]规范的js[`模块化`][3]加载框架。然后发现他和php的差别还是蛮大哈，有一些问题，就开始在脑中转啊转，记录下来，以免下次脑洞又大开了。
#CMD
## 什么是CMD
CMD（Common Module Definition）说白了就是一个前端规范，他规定了一个长的好看的`模块`（Module）应该长成什么样。那为什么要用CMD呢？要解释为什么用CMD我们要先知道什么是模块(module)。。。为了表示我还是有严谨性思维的，我们以下的讨论都只局限在`javascript`的大前提下。
### 什么是模块
[wiki上对于模块的定义如下][4]：
> 软件模块（Module）是一套一致而互相有紧密关连的软件组织。它分别包含了程序和数据结构两部分。
> 现代软件开发往往利用模块作合成的单位。
> 模块的接口表达了由该模块提供的功能和调用它时所需的元素。
> 模块是可能分开地被编写的单位。这使他们可再用和允许广泛人员同时协作编写及研究不同的模块。

解释的其实都比较清楚了，简单的说，就是一个可以复用，替换的数据和执行体的集合。
<!--more-->
### 为什么要用模块
这个啊，要从最开始说起，最开始，我们写javascript代码是这样的。
```
    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="utf-8">
            <title>this is a title</title>
        </head>
        <body>
            <div id="contianer"></div>
            <script type="application/javascript">
                var a = document.getElementById("contianer");
                a.addEventListener('click',function(){
                    //do some thing
                },false);
            </script>
        </body>
    </html>
```
这样写代码很简单，写的很爽，加上一点jquery啊，zepto啊，就更爽了，但是呢，如果代码一多，我们就会很蛋疼了。首先，我们会发现全局变量满天飞，根本不知道哪些变量能用，哪些变量不能用，哪些变量该GC哪些并不能够GC。然后，当我们想把一些代码通过
```
<script type="application/javascript" src="path/to/script"></script>

```
的方式外联的时候，就会发现我根本就不知道别人都用了什么啊，a的东西上线，b就不能用了。囧。。。

### 如何使用模块
所以，后来聪明一点的人们就开始这样写代码。
```
    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="utf-8">
            <title>this is a title</title>
        </head>
        <body>
            <div id="contianer"></div>
            <script type="application/javascript">
                ;(function(){
                    var a = document.getElementById("contianer");
                    a.addEventListener('click',function(){
                        //do some thing
                    },false);})();
            </script>
        </body>
    </html>
```
通过js的闭包机制，和js的变量作用域的机制，来分隔不同的代码。通过这样的分隔，我们权且将用匿名函数包起来的代码叫做一个模块吧。
```
 ;(function(jquery){
    var a = jquery();//通过变量传递的方式来使用函数外的变量，当然，如果是全局变量，可以不用。= =
    //some javascript code
 })(jquery)
```
然后呢，这样我们就可以在自己的模块中做一些羞羞的事了。但是，如果，我们要让其他人和我们一起做一些羞羞的事要怎么办？这个时候涉及到一个接口暴露的问题，在其他的面向对象的语言当中，我们可以使用实例化和命名空间来实现，但是js天生就没有类的概念啊，虽然，我们可以在上面模拟实现。既然我们可以模拟实现类，我们自然就可以模拟实例化和命名空间咯~！于是就有了以下的代码。
```
//定义
var moduleName = (function(){
    var a = {},b=1;//b作为这个module的私有变量。
    function privateFunction(){
        //do some thing;
    }
    
    a.b=1;//a的公有变量
    a.publicFunction(){
        //do some thing;
    }
    
    return a;
})();
//调用
moduleName.publicFunction();

```
这样的话，我们就可以将我们想允许对外调用的函数暴露出来给大家使用了。那么大家就可以一起做一些羞羞的事情了。如果要再复杂一点，我们可以看以下一个例子。
```
var util = (function(parent,$){
    var my = parent.parent = parent.parent||{};
    my.mymethod =  function(){//mymethod属于parent.parent对象的子方法
        // do some thing
    }
    return parent;
})(parent||{},jquery);
```
以上的话，是将子模块的属性方法动态加载到了父模块上。

## 为什么用CMD
讨论完如何用模块，我们回来讲为什么用CMD，为什么呢，这个啊，因为实现js的模块化的方法还是有很多的啊。可以参考[这篇文章][5]。虽然不知道jb从哪里抠来的文章，我没有找到原作者。。。除了实现细节的实现方法之外，加载的机制不同也出现了其他不同的规范,比如说`AMD`（Asynchronous Module Definition）。简单的说，二者区别在如下：
>-  AMD是依赖关系前置,
CMD是按需加载。
>- AMD:API根据使用范围有区别，但使用同一个api接口
CMD:每个API的职责单一
>- AMD的优点是：异步并行加载，在AMD的规范下，同时异步加载是不会产生错误的。
CMD的机制则不同，这种加载方式会产生错误，如果能规范化模块内容形式，也可以

如果要深入了解他们二者的区别，请看[玉伯大大在知乎上的回答][6]；

## 如何使用CMD
好了，说了那么多，我们要怎么用呢。CMD的常用接口有五个。我也就看懂了这五个 T…T。分别是`define`，`require`，`require.async`，`exports`，`module`。
### define
define 从字面上理解，是定义的意思，那么他就是定义的意思。定义一个`函数`，`对象`，或者`字符串`。如果是一个参数的时候：
```
//define(factory)
define({'foo':'东东'});
//这个东西可以理解成为
define(function(){
    return {
        'foo':'东东'
    }
})；
define('字符串中的{{foo}}')//据说这个是用来定义模板的，其实我并不是很清楚这个要怎么理解。
//定义函数时，默认传入的参数有三个 require ，export，module
define(function(require,export,module){
    //do some thing;
});
```
define一般情况下会使用两个以上参数，`id`，`deps`，`function`。其中，id表示模块标识符，deps`数组`表示依赖关系。*id和deps可省，如省则有构建器自动生成。*

```
define('a',['zepto'],function(){
    //do some thing
});

//tips：一个小技巧，用来判断页面上是否有CMD加载器

if(typeof define === "function"&&define.md){
    //说明页面中存在cmd加载器
}

```
### require
require是一个`function` 用来将其他模块的接口导入。sea.js中对require有自己的一个规范。[详情点我][7]
```
define(function(require,exports,module){
    //加载
    var a = reuire('./a');
    //调用
    a.method();
    // 异步加载
    require.async('./b',function(b){
        b.method();//调用
    });//单个
    require.async(['./c','./d'],function(c,d){
        c.method();
        d.method();
    })；//多个
    //require.resolve()通过require的内部机制获取module的绝对路径
    console.log(require.resolve('./e'));//http://example.com/path/to/module
});
```
### exports 
exports 是一个`objetc`，用来提供对外的接口。

```
define(function(require,exports,module){
    exports.a = "aaa";//对外开放a
    exports.method = function(){
        //do some thing
    }；//对外开放的方法
    //可以使用return object 来代替exports
    return {
        a: "aaa",
        method: function(){
            //do some thing
        }
    }   
});
```
### module
module 是一个object，用来存储当前module的一些属性和方法，可以简单的理解为this。
```
define('id',['jquery'],function(require,exports,module){
    //module.id记录了当前的id
    console.log(module.id)//id
    //module.uri通过内部机制获取当前module的绝对路径
    console.log(module.uri);//http://example.com/path/to/this/file.js
    //如果不填id的话，module.id === module.uri
    console.log(module.dependencies)//当前module的引用['jquery']
    console.log(module.exports === exports);//true
    //exports是module.exports的一个引用
});
```

# 一些新的思考
CMD 其实是sea.js在推广过程中的规范化输出，但是，还是有一些东西在定义上会给日以困惑，比如说交叉引用的问题。具体的问题描述和解决可以看[这里][8]。在sea.js的3.0中对于这个有了一个修复，使用了node对于module的加载机制，可以看[这里][9]，简单的说，使用cache和flag的机制来实现重复引用时只加载一次。以下是sea.js的源码中和这个有关的部分。

```
Module.prototype.pass = function() {
  var mod = this

  var len = mod.dependencies.length

  for (var i = 0; i < mod._entry.length; i++) {
    var entry = mod._entry[i]
    var count = 0
    for (var j = 0; j < len; j++) {
      var m = mod.deps[mod.dependencies[j]]
      // If the module is unload and unused in the entry, pass entry to it
      if (m.status < STATUS.LOADED && !entry.history.hasOwnProperty(m.uri)) {
        entry.history[m.uri] = true
        count++
        m._entry.push(entry)
        if(m.status === STATUS.LOADING) {
          m.pass()
        }
      }
    }
    // If has passed the entry to it's dependencies, modify the entry's count and del it in the module
    if (count > 0) {
      entry.remain += count - 1
      mod._entry.shift()
      i--
    }
  }
}

function require(id) {
    var m = mod.deps[id] || Module.get(require.resolve(id))
    if (m.status == STATUS.ERROR) {
      throw new Error('module was broken: ' + m.uri)
    }
    return m.exec()
  }
  
  Module.get = function(uri, deps) {
  return cachedMods[uri] || (cachedMods[uri] = new Module(uri, deps))
}
```


  [1]: http://seajs.org/docs/#intro
  [2]: https://github.com/seajs/seajs/issues/242
  [3]: https://github.com/seajs/seajs/issues/547
  [4]: http://zh.wikipedia.org/wiki/%E6%A8%A1%E7%B5%84_%28%E7%A8%8B%E5%BC%8F%E8%A8%AD%E8%A8%88%29
  [5]: http://www.jb51.net/article/34958.htm
  [6]: http://www.zhihu.com/question/20351507
  [7]: https://github.com/seajs/seajs/issues/259
  [8]: http://www.jianshu.com/p/1245af09383e
  [9]: http://www.infoq.com/cn/articles/nodejs-module-mechanism/
