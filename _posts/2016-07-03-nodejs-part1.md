---
layout: post
title: "蕙泉斋"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## 《了不起的Node.js》读书笔记
###  ——命令行工具(CLI)以及FS API:首个Node应用


#### 创建一个命令行文件浏览工具——学习进程处理（stdio）及文件系统（fs）API

本文中，我们将尝试使用Node.js构建一个简单的应用：

1. 该应用通过终端与用户交互。
2. 启动时显示当前目录下信息；选择目录显示该目录下的信息，选择文件显示该文件内容。

##### 新建项目目录、package.json文件
1. 新建项目目录，并将其命名为file-explorer。
2. 在项目的根目录下定义package.json文件。定义package.json文件可以方便地管理对npm模块的依赖，同时可以对模块进行发布。

##### 引入项目中使用到的模块
1. 处理进程（stdio）（包括stdin及stdout）是全局process对象的一部分。
2. 引入核心模块fs。

*fs模块是**唯一**一个同时提供同步、异步API的模块。*

*使用fs.readdir获取文件列表，回调函数的第一个参数时err对象，第二个参数是files数组*

	fs.readdir(_dirname, function(err, files){
		console.log(files);
	})

*理解流（stream）*

*持续不断的对数据进行读写时，流就出现了。*  

*process对象中包含三个流对象，分别对应三个UNIX标准流：stdin标准输入、stdout标准输出、stderr标准错误。stdin是可读流，strout及stderr是可写流。*
![stream](http://o7bm68198.bkt.clouddn.com/stream.png)
*流的编码：如果在流上设置了编码，则会得到编码后的字符串。如utf-8、ascii等，而不是原始的buffer。*

这个简单的应用（node命令行程序）实现代码如下：[ （项目链接）](https://github.com/congtou221/file-explorer "file-explorer")

	var fs = require('fs');
	var stdout = process.stdout;
	var stdin = process.stdin;

	fs.readdir(process.cwd(), function(err, files){
		//process.cwd()返回当前目录
		console.log(''); //为了输出更加友好，先输出一个空格
		if(!files.length){
			console.log('\033[31m No files to show!\033[39m\n');//\033[31m和\033[39m是为了让文本呈现红色
		}
		console.log('Select which file or directory you want to see\n');

		var stats = [];
		function file(i){
			var filename = files[i];
			fs.stat(__dirname + '/' + filename, function(err, stat){ //fs.state返回文件或目录的元数据

				stats[i] = stat; //将stat对象保存下来
				//如果路径代表的是目录，则用不同的颜色标示
				if(stat.isDirectory()){
					console.log('	'+i+' \033[36m'+filename+'/\033[39m');
				}else{
					console.log('	'+i+'\033[90m'+filename+'\033[39m');
				}

				i++;
				if(i == files.length){
					console.log('');
					read();
				}else{
					file(i);
				}
			})
		}
		function read(){
			stdout.write('	\033[33mEnter your choice: \033[39m'); //使用process.stdout.write而不是console.log，不会输入换行符
			stdin.resume();//等待用户输入
			stdin.setEncoding('utf8');//设置输入流编码为utf8，这样就支持特殊字符了	
			stdin.on('data', option);	
		}
		function option(data){
			var filename = files[Number(data)];

			if(!filename){
				console.log('	\033[31mEnter your choice: \033[39m');
			}else{
				stdin.pause();//检查通过，再次将流暂停，便于程序顺利退出

				if(stats[Number(data)].isDirectory()){
					fs.readdir(__dirname+'/'+filename, function(err, files){
						console.log('');
						files.forEach(function(file){
							console.log('	-	'+file);
						})
						console.log('');
					})
				}else{
					fs.readFile(__dirname+'/'+filename, 'utf8', function(err, data){ //事先制定编码，得到的数据就是相应的字符串了
						console.log('');
						console.log('\033[90m' + data.replace(/(.*)/g, '	$1')+'\033[39m'); //添加辅助缩进
					})				
				}

			}
		}
		file(0);
	})


####  对CLI一探究竟
##### argv
process.argv以数组的形式，保存了node程序运行时的所有参数。

该数组的第一个元素是‘node’，第二个元素是执行文件的路径，随后才是真正的参数。
##### 工作目录
__dirname 用于获取文件系统中的目录;

process.cwd() 用于获取命令行中的当前工作目录；process.chdir方法允许灵活地更改工作目录。
##### 环境变量
process.env 用于访问shell环境下的变量。
##### 退出
process.exit(退出代码)  用于让一个应用退出。 
当发生错误时，退出程序最好使用退出代码1.
##### 信号
进程和操作系统通过信号通信。在node中，在process对象上以事件分发的形式发送信号。
##### ANSI转义码
使用ANSI转义码在终端下控制文本的格式、颜色等。
	
	console.log('\033[90m' + data.replace(/(.*)/g, '	$1')+'\033[39m');
	
	/*
		\033表示转义序列的开始
		[表示开始颜色设置
		90表示前景色为亮灰色
		m表示颜色设置结束
	*/
#### 对fs一探究竟
##### Stream
通过fs模块的Stream API对数据进行读写操作时，对内存对分配不是一次性完成的。

	//回调函数在整个文件读取完毕后才会触发
	fs.readFile('my-file.txt', function(err, contents){
		//对文件的处理
	});	
	
	//fs.createReadStream为文件创建一个可读的Stream对象
	var stream = fs.createReadStream('my-file.txt');
	stream.on('data', function(chunk){
		//处理文件部分内容
	});
	stream.on('end', function(chunk){
		//文件读取完毕
	});
*可读stream fs.createReadStream*

*可写stream fs.WriteStream*

##### 监视
当文件系统中的文件（或目录）发生变化时，分发一个事件，触发指定的回调函数。
	
	//监视文件内容
	fs.watchFile(process.cwd()+'/'+file, function(){
		//回调函数的操作
	})
	
	//监视目录
	fs.watch(process.cwd()+'/'+file, function(){
		//回调函数的操作
	})