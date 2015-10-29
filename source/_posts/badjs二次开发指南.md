title: "badjs二次开发指南"
date: 2015-09-17 12:11:30
tags: badjs
---
#badjs是什么

    badjs是一个基于node.js的监控终端js错误的系统，包括js错误上报,错误信息统计和分析，数据可视化分析。

<i class="icon-exclamation-sign"></i>  假装你已经拥有所有的前置能力要求，如果并不是这样的话，在最后面有详细说明。

#  首先，看看badjs都有些什么

先看图：
![此处输入图片的描述][2]

在其开源的[github][3]上，我们能很清晰的看到，整个badjs的结构还是很清晰的，主要是以下几个组件，嗯，整套系统都是用node写的。整个都是js的狂欢。
    - badjs-web（管理端）
    - badjs-storage（存储端）
    - badjs-report（终端上报）
    - badjs-accepter（服务端接收）
    
    
# 然后，从宏观上的看一下BadJs都干了些什么。

##在浏览器端
    这一部分，主要是badjs-report，他的任务是捕捉js的报错，并把报错进行上报。这一部分，主要是要在页面中引入js，并配置，这一部分并不属于二次开发的范畴中，所以，不详述了。
    
    
##在服务器端
在服务端，整套badjs包括接收端，存储端和管理端共三个部分，这三个部分都是基于express的框架。

- badjs-web
- badjs-storage
- badjs-accepter
    ![流程图][4]
其中，`accepter`负责接收badsjs-report上报过来的数据，也就是页面上报过来的数据。然后将数据整理一下，校验一下是否是有效数据，然后通过[`zmq`组件][5]将数据传递给badjs-storage，badjs-storage则负责将传递过来的数据进行存储，这里使用了mongoDB作为主存储，`file`作为辅助cache，在badjs-storage中使用了`map-stream`作为其数据流的管理。而badjs-web则是将badjs-storage的数据用一种更人性化的形式呈现出来，这里用到了`mysql`作为存储。嗯，整个体系比较简单的看就是这样的。

<!--more-->
#具体的来分析一下

## badjs-storage
### 结构图
![badjs-storage结构图][8]

这个组件是基于node的，所以，他们使用了express作为其整体的骨架，嗯，文件的目录结构就是这样的。
![目录结构图][9]
嗯，从文件上来看，主要是`acceptor`，`service`，`storage`这几个部分。嗯，就像我们都知道的那样，express的入口文件当然使我们的app.js，看看它都干了什么。并不是很长，我就直接贴出来吧。
```
//...此处略去若干

var dispatcher = require(GLOBAL.pjconfig.acceptor.module)
  , save = require('./storage/MongodbStorage');
var cacheCount = require('./service/cacheErrorCount');


// use zmq to dispatch
dispatcher()
  .pipe(save());


logger.log('badjs-storage start ...');

setTimeout(function (){
    require('./service/query')();
    cacheCount.insertAuto();
    require('./service/autoClear')();
},1000);

```

其实前面有一大堆都是进行环境判断的，在启动的时候，附加不同的参数，将会使用不同的配置文件。
### 存储逻辑
在app.js的第3行，这里使用了调度器，而这个调度器指向的是`acceptor\zmq.js`,在这个zmp中，拉起了一个和badjs-accepter的tcp连接。这里通过map-stream来保证每个tcp数据流能以一个队列的形式来执行map定义的函数。。。不知道理解的对不对啊。可以直接看[这里][10],你也可以看[这里][11]来理解。而调度器指向的存储是storage里面的mongostorage，这个文件将得到的数据存入分布式的mongoDB中。注意哦，这里使用是分布式的mongoDB，所以在使用一些函数的时候，要注意是不是支持分布式。
### 查询逻辑
存储逻辑比较好理解，因为为了保证高并发的数据流能够得到很好的处理，所以，越是简单的设计，越是可靠~！然后，我们来看看复杂的查询逻辑。这里的查询逻辑是交给了service的query.js来处理。嗯，这个文件有点长，我们不直接贴了，看一下折叠后的代码，理解下逻辑。
[代码图][12]
嗯，很直白的express的用法，connect中间件将请求分流导向不同的处理函数，在处理函数里处理自己的逻辑即可。处理逻辑中，比较建议的写法是只在函数中处理请求检查，函数response填充处理。将具体的逻辑处理抽象成一个函数放在exports的外部，如果是比较重的逻辑，则可以当初写成一个service来执行。

### mongo
这里重新提一下分布式的概念。先看图
![此处输入图片的描述][13]
会有一台前置机，负责如何是分配存储和读取，在处理的时候，请注意mongo命令中对分布式的支持。
嗯，说两个比较复杂的，其他的就很好理解的。一个是在数据插入的时候。也就是storage里面的MongoStorage。

```
var insertDocuments = function(db , model) {
    var collectionName = 'badjslog_' + model.id;
    var collection = db.collection(collectionName); //获取数据集
    collection.insert([  //插入数据
        model.model
    ] , function (err , result){
        if(err){
            logger.debug('err,err is'+err);
            return;
        }
        if (hadCreatedCollection[collectionName]) {
            return ;
        }
        collection.indexExists('date_-1_level_1' , function (err , result ){ //判断数据集合索引
            if(!result){
                collection.createIndex( {date : -1 , level : 1 } , function (err , result){//建立索引，其中-1表示降序，1表示升序，前端的key表示对象。

                });
                if (global.MONGO_SHARD) {
                    shardCollection(db, collection, collectionName);
                }
            }
            hadCreatedCollection[collectionName] = true;
        })
    });

    logger.debug("save one log : " + JSON.stringify(model.model));
};

// shard new collection when created
var shardCollection = function (db, collection, collectionName) {
    collection.createIndex({_id: "hashed"}, function (err, result) {
        if (err) {
            logger.info("failed to create hashed index");
        } else {
            logger.info("hashed index created");

            var adminDb = db.admin();
            if (global.MONGO_ADMIN_USER && global.MONGO_ADMIN_PASSWORD) {
                adminDb.authenticate(global.MONGO_ADMIN_USER, global.MONGO_ADMIN_PASSWORD, function (err, result) { //权限认证，因为这里使用的是admin，要执行命令
                    if (err) {
                        logger.info("failed to access adminDB");
                    } else {
                        adminDb.command({ //执行mongo命令
                            shardcollection: "badjs." + collectionName,
                            key: {_id: "hashed"}
                        }, function (err, info) {
                            if (err) {
                                logger.info("failed to shardcollection " + collectionName);
                            } else {
                                logger.info(collectionName + " shard correctly");
                            }
                        });
                    }
                });
            }
        }
    });
};
```
这个函数的作用是，插入数据的时候看一下是不是有分布式，如果是，则使用分布式。
嗯，另一个例子看起来简单一点。
```
 var cursor = collection.aggregate([
    {$match: {'date': {$lt: endDate, $gt: startDate}}},
    {
        $group: {
            _id: {
                time: {$dateToString: {format: "%Y-%m-%d %H:%M", date: '$date'}}
            },
            count: {$sum: 1}
        }
    },
    {$sort: {"_id": 1}}
]);
```
其实这里一点都不简单，这里使用的是一个聚合查询，同时使用了聚合通道，具体的话，可以参考[官方的说明文档][14]，这里做一个说明，group，mapReduce这两个都是聚合查询的，但是group是不支持分布的，mapReduce使用的是map-reduce框架，但是实现的比较复杂，而且性能比聚合通道低。英文看不懂的话，可以看这篇[博文][15]。


## <a name="web" href="#web">badjs-web</a>

badjs-storage还是比较清晰的，嗯，然后是管理端，这里话，先上图。
### 结构图
![此处输入图片的描述][16]
实际上整体的结构很复杂。。。这里画的比较简单，把worker和service放在了一起，整体说明。
在和mysql连接这块使用了orm，用数据岛的模式来做对象化的数据处理。简单的说，就是可以像操作对象一下操作数据库。

### 请求逻辑
请求逻辑是指从url过来的请求解析逻辑，包括请求html，接口请求，还有静态请求。

- 静态资源请求

这个最简单，通过express框架，直接指向相应的资源文件。单独拿出来，是因为，这个地方的js是使用的模块化开发，webpack打包。每个页面的逻辑代码在static\module下的对应`文件夹`里，这里的文件其实是source文件，因为在发布前，要执行webpack命令，通过webpack将页面的内容打包到module下的`接口文件`（entry.*.js）里。主页面的逻辑是基于事件的，因为，渲染逻辑在请求html的时候就已经走完了。所有的事件都是委托给document.body去执行的。具体的实现方式是这样的。
```
//页面中
<a data-event-click="xxx">

//js中

$(document.body).on('click','xxx',function(){})
```
还有一部分是websocket的，因为自己不是很懂，就不详述了。

- 页面渲染逻辑

嗯，实话实说，这个页面渲染的逻辑相对比较简单，在badjs-web中，使用的页面渲染引擎是一个内部人员自行开发的micro-tpl引擎，说明文档嘛，看[这个][17]吧。请求走的是express工作流，从router出来，简单的没有复杂的页面逻辑的请求，直接渲染模板，并返回，又复杂页面渲染逻辑的，则会通过action调用不同的service来实现逻辑获取，并渲染模板。

- 接口请求逻辑

这里着重讲一下我们对于既有的二次开发的接口。原有的接口请求是这样的。
```
app.use("/",function(req, res , next){
        //controller 请求action
           
           //....此处略去若干
           
            //根据不同actionName 调用不同action
            try{
                switch(action){
                    case "user": UserAction[operation](params,req, res);break;
                    case "apply": ApplyAction[operation](params,req,  res);break;
                    case "approve": ApproveAction[operation](params,req, res);break;
                    case "log" : LogAction[operation](params,req, res); break;
                    case "userApply": UserApplyAction[operation](params,req, res);break;
                    case "statistics" : StatisticsAction[operation](params, req, res); break;

                    default  : next();
                }
            }catch(e){
                res.send(404, 'Sorry! can not found action.');
            }
            return;
    });
```
这个请求是通过正则匹配来获取不同的请求参数，进行请求分流。嗯，相对而言，我做的就比较简单粗暴了。直接的监听，get请求。
```
app.get('/xxx',function(){})
```
捕捉到相应的请求之后，将请求的参数传递给相应的action去处理。例如，log页面的请求交给LogAction，在action中，对不同动作，如果需要数据流，则会调用不同的底层服务（service）来实现和数据流的交互，那么，在logAction中，我们使用的就是LogService。在service中发出http请求去拉去badjs-storage的数据，或者，通过数据岛（DAO）来实现和mysql的交互。

整理一下就是，action处理动作，service处理数据流，dao负责和数据库（mysql）交互。

### 自行逻辑
这个主要是调用staticService去处理一些发送邮件啊，统计数据啊，之类的服务，因为不在改动逻辑中，所以没有深究。

----
# <a name="tag" href="#tag">准备知识</a>
在对badjs进行二次开发之前，请确保，你已知道并了解如下的知识，如果不知道，也没关系，只要知道怎么写js就行了，如果这也不知道，也没关系，只要知道怎么写回调就ok了。准备知识是为完全没有接触过服务器的前端攻城狮准备的，如果，你很自信，那么请跳过这些~。
## node.js
[官网点这里~！][18]
[教程可以看这个。][19]
这是运行服务端的js，基于v8的高效io的服务端脚本语言。语法和浏览器端的javascript是一致的。基于原型的继承和面向对象，基于事件的单线程异步io，当然，你也可以优雅的使用ecmas6的class，虽然本质上还是一个function。。。和php这门全世界最好的语言最大的区别在于其基于事件的异步执行机制，所有的请求和数据处理都是围绕着事件来运行的。总之，要记得写回调~！

除了回调，node基于模块的开发。我们先看一个小例子。
```
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, '127.0.0.1');
console.log('Server running at http://127.0.0.1:1337/');
```
这是一个http服务器，它会监听本地回环的1337端口，当有请求进来的时候，回复一个hello world。so easy。提出来的原因在于，其`var http = require('http')`，这里引入了一个node的自有模块，变量声明，定义的同时，也被实例化了。之后直接用就好了，和seajs的模块化一样的，遵循[commonjs][20]的规定，实例化的内容是接口文件通过`modules.exports`暴露出来的接口。未暴露在接口上的变量和方法是私有的。


## express
[官网看这里][21]
node只是一门脚本语言，所以，我们不能向lnmp好基友里面一样将请求委托给nginx做转发，而是要自己做处理。就像上面给出的http服务器的例子一样。嗯，上面的例子明显太单薄，那么我们就需要一个服务器软件来帮我们处理复杂的http请求。在node世界里，有一个framework叫express，他封装了node上的http请求处理。我们可以通过其来处理http请求。一个标准的express的目录结构是这样的。
[简单的express目录结构][22]
`./bin`目录下的www文件是启动文件，可以执行
```
node www 
```
来启动程序。但是，在根目录下的app.js才是主线程程序入口，里面一般会进行express的初始化，定义一些全局变量，引入其他需要运行的程序，嗯，发现不清楚和莫名其妙的全局变量的时候，请来app.js，每次都会有新发现哦。嗯，router文件夹下面存放着的是程序的路由了，express的路由设计和ROR很像，可以理解为是以路由表的形式来分发流量，query的内容进行类似key/value的数据导向。我们来看下面这个例子。
```
/* GET home page. URL： http://www.example.com/  */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
});

//url: http://www.example.com/xxx

router.get('/xxx',function(req,res,next){
    // do some thing
})

```
至于路由更复杂的使用方式，可以参考[这里][23]哦。

## mongo

[官网看这里~！][24]
嗯，因为这里使用到了mongoDB做存储，所以，顺带说一下mongo的快速使用方法。嗯，首先mongo是一个nosql，典型的key/value的存储数据库，他们家还有一个叫redis的家伙，跑的快，体积小，全cache，谁用谁知道。mongo使用的存储数据格式是json，所以对js有天然的亲和力。嗯，它和其他的数据库一样，有两个很重要的概念，一个是数据库（db），一个是数据集合（collections）也就是我们常说的表（tables）。就像我们在其他的数据库中学到的那样，数据集合是放在数据库里面的，所以，在这上面，mongo还是跟其他的数据库是一样的。然后我们来讲不一样的地方。首先，存储的内容格式不一致。在mongo中没有表的概念，所有的数据都是以键值对的形式存在的，这意味着，数据的格式是不强制标准的。那么，我们就不能够使用基于结构化数据的SQL语句来查询和插入数据，为了提供数据库基本的增删改查，mongo提供了另外一套体系，我们称其为命令。同时，因为数据和数据之间是低耦合的，所以，mongo非常方便的用来分布部署，就是说，我可以把一个数据库放在两台不同的服务器上，进行扩容，同时分布对查询是半透明，半透明的意思是，对于简单的查询是透明的，但是如果，你使用了一部分复杂的查询功能的时候，这个东西就比较坑爹了，因为，有一些方法是不支持分布式的。
在node中，我们需要通过mongo的连接构件（mongo-node-native）来处理mongo的操作，包括连接，和常用的增删改查。嗯，文档请看[这里][25]。
嗯，简单的说一下常用的命令。
```
//建立连接
MongoClient.connect(url, function (err, db) {});
//查询
 mongoDB.collection('badjslog_' + id).find(queryJSON, function (error, cursor) {});
//插入
collection.insert(data , function (err , result){});
//删除
deleteMany(filter, options, callback)
//修改
db.collection.update(selector, document, options, callback)

```

## mysql
[官网看这里。][26]
mysql是一个经典的关系型的数据库，据说，是这个世界上最流行的开源数据库，因为他的好基友是世界上最好的语言。相比较于上面的nosql，mysql的基本存储结构是表，所有的数据都会变成二维的方式进行存储。二维的数据存储结构的优点是，如此结构化的数据，我们能够很方便的额使用各种sql范式进行管理，通过各种酷炫的sql语句进行增删改查。但是，问题的核心在于数据存储的方式和我们使用数据的方式是不一样的。在实际的使用中，我们希望得到的是一个可操作的对象，每一个字段是这个对象的属性。在nosql中，这一点天然成立，但是在sql中，这个问题就比较尴尬了，因为，我们拿到的数据就是一堆的零散的数组，要把这些零散数组组织成一个对象，然后像操作对象一样的操作数据库，对，这货叫做ORM（Object Relational Mapping），在badjs中，使用了[orm][27]作为orm。
## zmq 
官网看[这里][28]。
这货就是一个封装好的socket套件，用来更加优雅的操作socket。用西加加写的，据说性能杠杠的，用来进行上报数据的传输。


  [1]: #tag
  [2]: http://i3.tietuku.com/faccb59a53dacb06.png
  [3]: https://github.com/BetterJS
  [4]: http://i1.tietuku.com/4e539619f5a97d4d.png
  [5]: https://github.com/anjuke/zguide-cn
  [6]: #storage
  [7]: #web
  [8]: http://i3.tietuku.com/763424b9eed874ba.png
  [9]: http://i1.tietuku.com/c85026b2d9ddbbef.png
  [10]: https://github.com/dominictarr/map-stream
  [11]: https://github.com/dominictarr/event-stream
  [12]: http://i1.tietuku.com/0afd76ac8b53583a.png
  [13]: http://i3.tietuku.com/0e879c480f5518eb.jpg
  [14]: http://docs.mongodb.org/v3.0/reference/method/db.collection.aggregate/#db.collection.aggregate
  [15]: http://blog.csdn.net/zhangzhebjut/article/details/16341823
  [16]: http://i3.tietuku.com/389ef26e063d2fa6.png
  [17]: https://github.com/miniflycn/micro-tpl
  [18]: https://nodejs.org/
  [19]: http://nqdeng.github.io/7-days-nodejs/
  [20]: http://javascript.ruanyifeng.com/nodejs/commonjs.html
  [21]: http://expressjs.com/
  [22]: http://i1.tietuku.com/2b87cf3549e8b254.png
  [23]: http://longpo.iteye.com/blog/2195035
  [24]: https://www.mongodb.org/
  [25]: http://mongodb.github.io/node-mongodb-native/2.0/api/
  [26]: https://www.mysql.com/
  [27]: https://www.npmjs.com/package/orm
  [28]: http://zeromq.org/
