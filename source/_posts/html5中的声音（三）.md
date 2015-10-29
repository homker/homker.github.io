title: "html5中的声音（三）"
date: 2015-05-21 12:48:47
tags: html5
---
在上一篇文章里，我们已经解释了如何脚本化一个音频文件，并对它做一个傅里叶变换，然后进行音频分析。我们可以把上次的代码做个模块化处理。这样比较方便调用。
```
define('wavemodule',['zepto','loadAudio'],function(require,exports,module){
    var loadAudio = require("loadAudio"),
        $ = require('zepto'),
        context = new AudioContext(),
        analyser = context.createAnalyser(),
        displayWave = function(waveByteDate){
            var canvas = $('#display')[0];
            var canvasContext = canvas.getContext('2d');
            canvasContext.fillStyle = "#428bca";
            canvasContext.clearRect(0,0,1024,100);
            for(var i = 0; i<waveByteDate.length; i++){
                canvasContext.fillRect(i,100,1,(0- (waveByteDate[i]/255 * 100)));
            }
            canvasContext.stroke();
        },
        playSoud = function(buffer) {
            console.log("work here");
            var source = context.createBufferSource();
            source.buffer = buffer;
            source.connect(analyser);
            analyser.smoothingTimeConstant = 0.8;
            analyser.connect(context.destination);
            var handler = setInterval(function(){
                var waveByteDate = new Uint8Array(analyser.frequencyBinCount);
                analyser.getByteFrequencyData(waveByteDate);
                displayWave(waveByteDate);//绘制波形
            },10)
            source.start(0);
            source.ended = function(){
                $("#wave").getContext("2d").clearRect(0,0,1024,100);
                clearInterval(handler);
            }
            source.stop = function(){
                if(!source.stop){
                    source.stop = source.noteOff;
                }
            };
        };

    exports.start = function(url){
        console.log("work here");
        loadAudio.setconfig(context,url,playSoud);
        loadAudio.start();
    };
})
```
# processor
好了，这次，我们讲的东西比较简单，实现一个将键盘输入的1,2,3,4,5,6,7播放成dou ri mi fa so la xi dou。
## 定义
这里涉及到一个可编辑音乐节点的问题。我们需要用到一个叫做`ScriptProcessorNode`（脚本处理节点）这个节点用于生成，处理，或使用JavaScript分析的音频节点。它有两个缓冲区，一个是包含所输入的音频数据，另一个则包含处理后的输出音频数据的audioNode节点。通过`Processing`来实现以下几个事件

- 每次输入缓冲器有新数据时发送给对象，
- 事件处理程序终止时，将已有的数据填充到数据输出缓冲器
<!--more-->
可以参考下图来理解这个东西：
![此处输入图片的描述][1]

嗯，和以前一样，英语很渣，所以给出英文原文：
>The ScriptProcessorNode interface allows the generation, processing, or analyzing of audio using JavaScript. It is an AudioNode audio-processing module that is linked to two buffers, one containing the input audio data, one containing the processed output audio data. An event, implementing the AudioProcessingEvent interface, is sent to the object each time the input buffer contains new data, and the event handler terminates when it has filled the output buffer with data.

你也可以点击[这里获取更多资料][2]。
还记得，我们之前是怎么播放音乐的吗？
XMLHttpRequset -> source.buffer -> analyser -> dest;
嗯，这次我们要把我们的processerNode加进去。把我们之前用的analyser节点替换掉。
于是乎就变成了以下：
XMLHttpRequest -> source.buffer -> analyser -> processerNode -> dest。
## 实例化
嗯，先说一下如何实例化。
```
var processor = context.createScriptProcessor(4096,1,1);
```
参数的话，第一参数是这个使用数据帧的大小，可以是256, 512, 1024, 2048, 4096, 8192 , 16384.数字越小，音频质量越差，越高则质量越好。第二参数是输入池的个数，第三个是输出池的个数。

## 输入到输出

嗯，这里我们要注意一下，在processNode里面，input池和output池是分离的，你要使用Processing事件来让这两个池子连在一起。这里简单实现一个这个函数。
```
processor.onaudioprocess = function(audioProcessingEvent) {
        var inputBuffer = audioProcessingEvent.inputBuffer, //输入帧
            outputBuffer = audioProcessingEvent.outputBuffer; //输出帧

        //循环输出通道
        for (var channel = 0; channel < outputBuffer.numberOfChannels; channel++) {
            var inputData = inputBuffer.getChannelData(channel),//输入数据
                outputData = outputBuffer.getChannelData(channel);//输出数据

            // proccesser = context.createScriptProcessor(4096,1,1), 这里的inputbuffer循环4096次
            for (var sample = 0; sample < inputBuffer.length; sample++) {
                outputData[sample] = inputData[sample];//将输入数据复制到输出数据
                // 如果想添加噪点的话，随机加上点数据就可以了~！
                //outputData[sample] += ((Math.random() * 2) - 1) * 0.2;
            }
        }
    }
```
嗯，函数体里面的解释应该比较清晰了，我就不多说什么了。
## 输入声音
好了，节点搞定了，我们要来考虑一下怎么往节点里面填充我们自己想要的数据。这里要提到一个叫做震荡器的节点。就是这个
```
var oscillator = context.createOscillator();
```
### 频率和音调
这个节点可以将频率值通过傅里叶变换转换成声音的电信号，然后在扬声器里播放出来。嗯，音频和音调是什么，请参考[这个][3]，还有[这个][4]，其实我们更在乎的是他们之间的关系什么。。。嗯，请参考[这个][5]，想更深入，请看[这个][6]，还是不够？那就看[这个][7]。嗯，等你看完了这些理论性很强的东西，我们就能实现下面这个对象：
```
var s ={48:261.6,49:330,50:370,51:392,52:440,53:494,54:277}
```
嗯，然后循环这个对象，把对象的值变成响应的震荡数值：
```
for( var i in s){
            var  value = s[i];
            s[i] = context.createOscillator();//提供了产生振荡器节点的方法，使用这个节点改变频率可以生成不一样的音调
            s[i].frequency.value = value;
            s[i].start();
        }
```
最后的，也是最简单的咯，监听键盘事件的`keydown`，`keyup`对应 1 2 3 4 5 6 7这个按键，down的时候链接到processor节点就行了。up的时候取消链接。
## 综合实现
最后将所有代码串在一起，就是下面这个模块，嗯调用的话，我就不展示了。

```
define('wavemodule',['zepto','loadAudio'],function(require,exports,module){
    var loadAudio = require("loadAudio"),
        $ = require('zepto'),
        context = new AudioContext(),
        analyser = context.createAnalyser(),
        processor = context.createScriptProcessor(4096,1,1), //单帧大小 4096 一个输入池，一个输出池。
        s ={48:261.6,49:330,50:370,51:392,52:440,53:494,54:277},
        displayWave = function(waveByteDate){
            var canvas = $('#display')[0];
            var canvasContext = canvas.getContext('2d');
            canvasContext.fillStyle = "#428bca";
            canvasContext.clearRect(0,0,1024,100);
            for(var i = 0; i<waveByteDate.length; i++){
                canvasContext.fillRect(i,100,1,(0- (waveByteDate[i]/255 * 100)));
            }
            canvasContext.stroke();
        },
        playSoud = function(buffer) {
            console.log("work here");
            var source = context.createBufferSource();
            source.buffer = buffer;
            source.connect(analyser);
            analyser.connect(processor);
            processor.connect(context.destination);
            var handler = setInterval(function(){
                var waveByteDate = new Uint8Array(analyser.frequencyBinCount);
                analyser.getByteFrequencyData(waveByteDate);
                displayWave(waveByteDate);//绘制波形
            },10)
            source.start(0);
            source.ended = function(){
                $("#wave").getContext("2d").clearRect(0,0,1024,100);
                clearInterval(handler);
            }
            source.stop = function(){
                if(!source.stop){
                    source.stop = source.noteOff;
                }
            };

        };
    processor.onaudioprocess = function(audioProcessingEvent) {
        var inputBuffer = audioProcessingEvent.inputBuffer, //输入帧
            outputBuffer = audioProcessingEvent.outputBuffer; //输出帧

        //循环输出通道
        for (var channel = 0; channel < outputBuffer.numberOfChannels; channel++) {
            var inputData = inputBuffer.getChannelData(channel),//输入数据
                outputData = outputBuffer.getChannelData(channel);//输出数据

            // proccesser = context.createScriptProcessor(4096,1,1), 这里的inputbuffer循环4096次
            for (var sample = 0; sample < inputBuffer.length; sample++) {
                outputData[sample] = inputData[sample];//将输入数据复制到输出数据
                // 如果想添加噪点的话，随机加上点数据就可以了~！
                //outputData[sample] += ((Math.random() * 2) - 1) * 0.2;
            }
        }
    }

    exports.start = function(url){
        console.log("work here");
        loadAudio.setconfig(context,url,playSoud);
        loadAudio.start();
        for( var i in s){
            var  value = s[i];
            s[i] = context.createOscillator();//提供了产生振荡器节点的方法，使用这个节点改变频率可以生成不一样的音调
            s[i].frequency.value = value;
            s[i].start();
        }
        window.addEventListener("keydown",function(e){
            //console.log(e.keyCode);
            if(e = s[e.keyCode]){
                //console.log(e);
                e.connect(proccesser);
            }
        });
        window.addEventListener("keyup",function(e){
            if(e=s[e.keyCode]){
                e.disconnect();
            }
        })
    };
})

```


  [1]: https://mdn.mozillademos.org/files/5157/WebAudioScriptProcessingNode.png
  [2]: https://developer.mozilla.org/en-US/docs/Web/API/ScriptProcessorNode
  [3]: http://amuseum.cdstm.cn/AMuseum/perceptive/page_2_ear/page_2_3/page_2_3_1_g.htm
  [4]: http://www.phy.ntnu.edu.tw/demolab/html.php?html=modules/sound/section2
  [5]: http://wenku.baidu.com/view/164cf78fd0d233d4b14e69da.html?re=view
  [6]: http://wenku.baidu.com/view/bb3e3ffa700abb68a982fb46.html
  [7]: http://wenku.baidu.com/view/d42fe5838762caaedd33d4a8.html
