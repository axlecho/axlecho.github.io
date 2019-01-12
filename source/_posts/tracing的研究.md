---
title: google tracing的研究
date: 2019-01-12 20:44:14
tags: android_tools
cover: /images/systrace.png
---

最近使用systrace做性能测试，在systrace上统计红绿灯，因为数到眼残，所以做了个自动数帧的工具。
主要做法是修改chrome tracing的源码

话说tracing这么好用，不出个cmdline真不合理。

[tracing on github](https://github.com/catapult-project/catapult)

幸运的看到了一段代码
```javascript
/*./tracing/tracing/extras/android/android_auditor.html*/
pushFramesAndJudgeJank_: function()
```

一看就知道是在判断帧的状态
然后就加了统计代码
```javascript
var redFrame = 0;
var yellowFrame = 0;
var greenFrame = 0;

app.process.frames.forEach(function(frame) {
    if (frame.totalDuration > EXPECTED_FRAME_TIME_MS * 2) {    // 红帧
        badFramesObserved += 2;
        frame.perfClass = FRAME_PERF_CLASS.TERRIBLE
        redFrame += 1;  
    } else if (frame.totalDuration > EXPECTED_FRAME_TIME_MS 
        || frameMissedDeadline(frame)) {    // 黄帧
        badFramesObserved++;
        frame.perfClass = FRAME_PERF_CLASS.BAD;
        yellowFrame += 1;
    } else {    // 绿帧
        frame.perfClass = FRAME_PERF_CLASS.GOOD;
        greenFrame += 1;
    }
});
```

最后用log输出
```javascript
console.log("%d %d %d",redFrame,yellowFrame,greenFrame);
```

后面又要搞自动化，就是吧js里面的东西打出来
有四个思路

*   用nodejs运行，将结果写文件
*   html5写文件
*   重定向log
*   建本地服务器，用post提交数据

nodejs的很简单，下了node-webkit的源码，然后建了个helloword的工程，发现打不开，突然就想到tracing这个东西跟chrome有很强关系，于是用火狐打开trace2html生成的html发现打不开，应该就是只有chrome能解析systrace的数据

html5的没怎么尝试，因为写文件会写到chrome的安装目录上的，无法写到html文件所在的目录（安全机制，这个限制是必须的），即使写出来用法也很奇怪

建本地服务器，这个应该是可以的，感觉就复杂了点，而且流程太多也不好，容易出bug

重定向一开始以为没希望，到google查了一下还真有这个用法
用下面的命令可以将许多log包括控制台的log输出到错误输出流上
```
google-chrome --enable-logging=stderr
```

很奇葩的是错误输出流的重定向好像有点问题
象如下的代码就输出不了，以后还得研究下，
```
google-chrome --enable-logging=stderr $1 2>&1 | grep "\[Frame\]"  > log
```

后来用nodejs移植了tracing,主要用了vm.runInThisContext,来运行tracing的各个模块,nodejs和javascript有好多不兼容多东西,坑死爹了.

改代码时用到的一坨正则，匹配html中的js代码
```shell
sed -n '/<script */,/<\/script>/p' base/base.js sed -e 'a#"progressMeter.update"#"}"#g' \
    -e 'i#"progressMeter.update"#"if(tr.silent == false) {"#g' \
    -e 's#"progressMeter.update"#"console.log"#g' tracing/importer/import.js
```

trace抓下来的数据是压缩过的，用下面命令可解压
```
zlib-flate -uncompress < trace >out
```

发现了一个极好多正则表达式
```
echo "asdfkjasldjkf\"shiner\"df" | sed 's/\(.*\)"\(.*\)"\(.*\)/\2/g'
```

开始port到windows上了,用的还是nodejs
因为用到的同步执行命令要求node最新版,所以用下面的命令来安装最新的工具
```
curl --silent --location https://deb.nodesource.com/setup_0.12 | sudo bash -
```

自动化打包的功能，先了解先 [Node.js - Zip/Unzip a folder](http://stackoverflow.com/questions/15530435/node-js-zip-unzip-a-folder)

只能说windows的cmd去屎
