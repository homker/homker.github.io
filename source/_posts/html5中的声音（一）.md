title: "html5中的声音（一）"
date: 2015-05-18 22:57:34
tags: html5
---

额，这是一个很久之前的项目了，但是因为这是第一个完全属于我自己的项目，所以我还是记录一下，用来怀冕一下，曾经那个少不更事的我，所做的那些事情。[项目地址][1]
#如何在页面上播放声音
嗯，传统的方式，`flash`，额，因为我不会flash编程，所以直接放弃，我们来找点高大上一点的东西好不好～！所以，ie6,7,8什么的都去死吧～！
## html5 audio 标签 
在html5中，对于声音的定义很方便哦，用了一个标签来解决这个问题。嗯，简单看一下用法。
```
<!--最简单的-->
<audio src="path/to/audio"> 
    你的浏览器好像并不支持audio标签
</audio>
<!--附加点功能怎么样？-->
<audio autoplay loop contorls preload muted src="path/to/audio">
    你的浏览器好像不支持audio标签
</audio>
```
看，是不是很简单，我去，这个标签简直简单到像`<img src="path\to\image" \>`～！来解释一下那几个属性。
 
- autoplay 是否在音频加载完毕的时候自动播放。 
- loop 是否循环播放
- contorls 是否显示浏览器的默认播放样式
- preload 是否预加载
- muted 是否静音
- src 这个就不用解释了吧，大家都懂

根据html5的规范，只要在标签内注明了的属性，其属性值则为true，不然则为false。当然这些属性自然是可以用js来修改的，我们稍后来解释。
## html5 source 标签
嗯，其实还有扩展，我们知道啊，mp3是上个世纪的产物啦，无论从哪个角度来说，都已经远远不能满足我们的需求了。为了佐证，可以参考[这个论坛的帖子][2]。
嗯，所以为了使我们的音频能有更好的音质和更cool的元素，我们可以这样写我们的`<audio>`标签。
```
<audio>
    <source src="path/to/aac" type="audio/mp4" />
    <source src="path/to/ogg" type="audio/ogg" />
    <source src="path/to/wav" type="audio/wav" />
    <source src="path/to/mp3" type="audio/mp3"  />
    你的浏览器并不支持这个东西。
</audio>
```
通过`<source>`标签来讲音频源引入，而且，这个地方可以引用多个音频源哦，加载顺序是由上至下寻址。根据渐进增强的原则，支持性最为广泛的mp3应该放在最底下，音质最好，但是支持性并不是很强的放在最顶上，当然，如果是在移动端，要重新思考这个问题，因为涉及到码率和取样率，文件大小等一堆的问题，这里就不赘述了。我比较推崇的方式是，当文件大小相差不大的时候尽量取音质好的。这里涉及到的几个采样率啊，码率啊，具体的解释可以参考[知呼上的解释][3]，和这个[博客][4]，还有[这个][5]。
<!--more-->
## javascript接口

上面讲了纯粹的html5对音频的支持，现在我们来看看浏览器在js API上对这个还有什么有意思的东西。
### audio
第一个就是audio咯，这个用法的话，并没有什么很特别的东西。先说一下常用的方法。
#### 方法
嗯，还是直接举例子比较直观，Don’t BB ，give me your code。
```
 //最简单的
 var player = new Audio("path\to\audio").play();
 
 //加上类型判断
 var player = new Audio();
 if(player.canPlayType('audio/ogg')){
    player.src = "path\to\ogg";
    player.play();
 }
 
 //tips:
 console.log(new Audio === getElementsByTagName("audio")[0]); //true
 
 //加上事件监听,当页面加载完成时开始播放
 windows.addEventListener('load',function(){
    document.getElementsByTagName("audio")[0].play();
 },false);
 
//点击id为pause的button时，播放暂停
var player = new Audio("path\to\audio").play(),
    pause = document.getElementById("pause");
    pause.addEventListener('click',function(){
        player.pause();
    },false)
//这里要注意一下，audio本身并没有提供stop()方法，如果想实现这个功能，我们要手动实现这个方法。
 Audio.prototype.stop = function(){
    this.pause();
    this.currentTime = 0;
};
            
//获取加载百分比
player = new Audio(); //不要写成了 player = new Audio().play();
var  loaded = Math.floor(player.buffered.end(0) / player.duration  * 100 ); // 当前加载的百分比
var played = Math.floor(player.played.end(0) / player.duration * 100); // 当前播放的百分比
```
正如所见，audio 本身就给了很多专属方法，比如说`play()`,`pause()`,`canPlayType()`额，好像其实并不多，还有就是对于`played`,`buffered`，`seekable`这三个TimeRanges对象，有`start()`和`end()`方法，用来获取时间节点。对于这两个函数的具体含义可以参考[w3c对它的定义][6]。
#### 属性
接下来，看看属性。
```
var player = new Audio();
player.src = "path\to\audio";
player.controls = true;
player.autoplay = true;
player.loop = true;
player.muted = true;
player.preload = true; //如果值为metadata则加载元数据（时长，数据帧）
player.buffered; // 已加载的数据大小
player.played; // 已播放的数据大小
player.currentTime;// 播放器应该跳过的时间
player.volume;//音量大大小
player.playbackRate;//播放的速度
player.defaultPlaybackRate;//默认播放速率。
player.duration;//音频的持续时长

```
感觉好像并没有什么好解释的。。。要说的，就是playbackRate，这个值的话，正常情况下是1.0 , 0 ~ 1表示慢速，>1 是加速 ，<0 表示回放。然而并有哪家浏览器实现了回放。。。= =，反正我在chrome 和 FF下测试时这样的。

#### 事件
事件就比较多了，除了DOM本身所带的那些事件，还有下面一堆的事件。

- loadstart 开始请求加载
- stalled 尝试加载数据，但是未能请求到
- progress 加载中。。。
- loadedmetadata 时长等元数据加载完毕
- waiting 由于未缓冲到足够的时长而不能够播放
- canplay 已经缓冲到足够播放的时长
- canplaythrough 已全部缓冲，可顺利播放
- suspend 已经缓冲大量数据
- play 开始播放，调用了play()方法或者autoplay
- timeupdate currentTime 属性发生改变时触发该事件
- pause 调用了pause()方法
- seeking 当currentTime到了一个为缓冲的时间点上
- seeked  currentTime所在时间点已经缓冲
- ended 播放结束 这里是currentTime == duration，而不是我们实现的stop()
- durationchange duration数据发生改变，这个东西，一般在loadedmetadata之前会发生。
- volumechange 音量改变时
- ratechange 速率改变时
- abort 用户要求停止加载媒体。额，我的理解是把页面关了。。。
- error 由于网络错误导致的音频未加载
- emptied 发生了错误

嗯，好像很多的样子，事件的话，用addEventListener监听一下就好了嘛~！
```
 var count = 0,
    player = new Audio("path\to\audio");
    player.play();
    player.addEventListener("loadstart",function(){
            count++;
            console.log(count + "- loadstart");
        },false);
    player.addEventListener("stalled",function(){
        count++;
        console.log(count + "- stalled");
    },false);
    player.addEventListener("progress",function(){
        count++;
        console.log(count + "- progress");
    },false);
    player.addEventListener("loadedmetadata",function(){
        count++;
        console.log(count + "- loadedmetadata");
    },false);
    player.addEventListener("waiting",function(){
        count++;
        console.log(count + "- waiting");
    },false);
    player.addEventListener("canplay",function(){
        count++;
        console.log(count + "- canplay");
    },false);
    player.addEventListener("canplaythrough",function(){
        count++;
        console.log(count + "- canplaythrough");
    },false);
    player.addEventListener("suspend",function(){
        count++;
        console.log(count + "- suspend");
    },false);
    player.addEventListener("play",function(){
        count++;
        console.log(count + "- play");
    },false);
    
/*
2015-05-18 22:44:42.885 test.html:38 1- loadstart
2015-05-18 22:44:42.924 test.html:50 2- loadedmetadata
2015-05-18 22:44:42.968 test.html:46 3- progress
2015-05-18 22:44:42.969 test.html:66 4- suspend
2015-05-18 22:44:43.892 test.html:58 5- canplay
2015-05-18 22:44:43.892 test.html:62 6- canplaythrough
2015-05-18 22:44:51.200 test.html:70 7- play
2015-05-18 22:44:53.029 test.html:46 8- progress
2015-05-18 22:44:53.030 test.html:66 9- suspend
2015-05-18 22:44:55.091 test.html:46 10- progress
2015-05-18 22:44:55.091 test.html:66 11- suspend
2015-05-18 22:44:57.129 test.html:46 12- progress
2015-05-18 22:44:57.130 test.html:66 13- suspend
2015-05-18 22:44:59.189 test.html:46 14- progress
2015-05-18 22:44:59.190 test.html:66 15- suspend
2015-05-18 22:45:00.603 test.html:58 16- canplay
2015-05-18 22:45:00.604 test.html:62 17- canplaythrough
...
*/
    
```
嗯，今天就先写这么多吧，明天开始写audioConetext接口。


  [1]: https://github.com/homker/audioPlayer
  [2]: http://bbs.imp3.net/thread-879493-1-1.html
  [3]: http://www.zhihu.com/question/20351692
  [4]: http://blog.csdn.net/wghhdzwzqbx02/article/details/7392059
  [5]: http://blog.sina.com.cn/s/blog_7032e6960100zzhn.html
  [6]: http://www.w3school.com.cn/tags/av_prop_buffered.asp
