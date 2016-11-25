chat-socket.io
==============

<h1>前端抢后端饭碗 -- Node.js + Socket.io 制作简易聊天室</h1>

<author>
<a href="http://leehey.org/homepage" target='_blank'>LeeHey 个人主页</a>
</author>

<p>
	version1 Demo Page <a href="http://128.199.194.125:8081/" target="_blank">点击进入</a>
</p>
<p>
	version2 Demo Page <a href="http://128.199.194.125:8082/" target="_blank">点击进入</a>
</p>
<p>
	请多开几个浏览器窗口进行聊天模拟。
</p>

<h3>目录</h3>
<p>
	<ul>
		<li><a href="#intro">1. 前言</a></li>
		<li><a href="#knowledge">2. 基本知识</a></li>
		<li><a href="#main">3. 正文</a></li>
		<li><a href="#pieces">4. 坑，坑，还是坑</a></li>
	</ul>
</p>

<h3><a name="intro">1. 前言</a></h3>

<p>看到这个题目的时候干后端的别打我。在接触Socket.io之前曾经用PHP + jQuery写了一个低效的长轮询只有消息同步功能的小聊天室就已经耗尽心力，更不用说利用PHP的Socket接口写WebSocket的聊天室，那更是灾难。</p>

<p>刚才一口气说了一堆大家都困惑的术语，接下来等本菜鸟解释一下。</p>

<p>一般来说，浏览器在接收到客户端请求(request)的时候，会发送给服务器端进行处理，服务器端处理完毕后会返回结果(response), 这样就完整地完成了一次HTTP的请求。当客户端想再更新页面信息的时候，需要刷新浏览器，再完成一次HTTP请求。</p>

<p>后来， AJAX的流行貌似使网站更加用户友好，增强了互动性，但实际上在效率上并没有提高，AJAX与后台的交互实际也是在一次又一次地完成HTTP请求。终于， 人们为了更好地实现实时应用，想出了一些办法，老办法叫Comet, 新办法叫Websocket.</p>

<p>老办法Comet是改良自AJAX（轮询）。拿PHP作为例子，其原理就是，利用AJAX给PHP后台发送请求，后台PHP设置运行时间为无限，并用一个死循环使HTTP请求不返回而达到持续监听后台的效果。一旦死循环里的代码检测发现数据/文件被修改，则跳出循环，返回更新后的数据。这种办法简明易懂，后台代码不需要写太多，但因为连接不断开而大量服务器资源。</p>

<p>新办法就是HTML5新的API叫做WebSocket.WebSocket采用了一些特殊的报头(Header)，使得浏览器和服务器只需要做一个握手的动作，就可以在浏览器和服务器之间建立一条连接通道，而毋须消耗大量服务器资源。更进一步的原理，大家可以参考下面两篇文章:</p>

<p>
(1) <a href="https://github.com/xionglun/goWebBook/blob/master/docs/08.2.md" target="_blank">https://github.com/xionglun/goWebBook/blob/master/docs/08.2.md</a>
</p>

<p>
(2) <a href="http://www.html5rocks.com/zh/tutorials/websockets/basics/" target="_blank">http://www.html5rocks.com/zh/tutorials/websockets/basics/</a>
</p>

<p>
今天我们会介绍用Node.js和Socket.io来轻松完成这个任务。
</p>

<p>
Socket.io是使用得非常普遍的前端调用Websocket的库，其<a href="http://socket.io/" target="_blank">官方网站</a>里面还自带了后台实现的代码。我们用Node.js也是奔着这一点去的，因为你们会发其前后端代码居然一模一样，我们毋须再为PHP，JAVA那些繁杂的代码而忧心。
</p>

<h3><a name="knowledge">2. 基本知识</a></h3>

<p>
<ul>
	<li>Websocket基础知识</li>
	<li>Javascript</li>
	<li>Node.js</li>
	<li>Express.js（Node.js中最流行的网页框架)</li>
</ul>
</p>


<h3><a name="main">3. 正文</a></h3>
<p>
	运行环境: Mac OSX 10.9.5
</p>
<p>
	首先我们需要安装Node.js以及其包管理npm。在此不赘述，请参考官方安装指南：
</p>

<p>
	<a href="https://github.com/joyent/node/wiki/Installation">官方官装指南</a>
</p>

<p>
	安装完之后，我们要新建一个文件夹。
</p>
<pre>
mkdir chat
cd chat
</pre>

<p>
	然后我们新建一个/chat/package.json文件，并输入以下代码。这个文件存放Node.js应用的依赖环境及模块。
</p>

<pre>
{
	"name": "chat",
	"version": "0.0.1",
	"description": "first socket.io app",
	"dependencies": {
	   "express": "^4.9.6",
	   "socket.io": "^1.1.0"
	}
}

</pre>

<p>
	然后输入以下命令，系统就会帮你自动安装所有需要的模块。并在/chat下面多出一个node_modules的文件夹存放所需的模块。
</p>

<pre>
npm install
</pre>

<p>
	/chat/index.js文件，并输入以下代码。这个文件主要是Node.js主要的服务器搭建，模块处始化的地方。如果有Node.js基础的同学相信会比较熟悉。没有的同学也不要紧，可以先用着。
</p>

<pre>
/**
 * index.js
 * @author LeeHey 
 * @email lcxfs1991@gmail.com
 * @description
 * This file is to setup the basic Node.js server and require necessary server module
 */

//express.js framework
//express.js 框架
var express = require('express');
var app = express();

//create an http object based on express.js
//基于express.js新建http对象
var http = require('http').Server(app);

//module manage folder path
//这是一个管理文件路径的模块
var path = require('path');

//main processing file
//主要处理聊天室系统的文件，稍后详细解释
var chat = require('./routes/chat');

//set /public as the folder to serve static files
//设置/public作为存放静态文件（如模块，样式）的文件夹
app.use(express.static(path.join(__dirname, 'public')));

//route / to /public/index.html
//使‘/’的请求都会输出./public/index.html文件内容
app.get('/', function(req, res){
  res.sendFile('./public/index.html');
});


//initialize all http setting in chat object
//初始化聊天室
chat.initialize(http);


//listen to port 2501
//监听2501端口
http.listen(2501, function(){
  console.log('listening on *:2501');
});
</pre>

<p>
	新建/chat/public 文件夹:
</p>
<pre>
	mkdir public
</pre>

<p>
	接下来新建/chat/routes 文件夹:
</p>

<pre>
	mkdir routes
</pre>

<p>
	然后新建 /chat/routes/chat.js以存放聊天室的基本业务逻辑。在chat.js里面，我们需要实现的有以下的几个基本的功能。
</p>

<p>
	<ul>
		<li>自动分配聊天室</li>
		<li>更改聊天室</li>
		<li>自动分配名字</li>
		<li>更改名字</li>
		<li>用户在相应聊天室里面发送消息</li>
		<li>用户在加入、离开的时候会自动发送系统消息</li>
	</ul>
</p>

<p>
网上大部份教程只会教怎么样搭建立一个消息发送的简单聊天室，而我今天打算包含更丰富的功能例如新建聊天室以及改名字。
</p>

<p>首先在chat.js中加入以下代码:</p>

<pre>
//express.js framework
//express.js框架
var express = require('express');
var app = express();

//socket io module
//socket io 模块
var socketIo = require('socket.io');

// create a new ojbect chat
//新建一个新的chat对象
var chat = {};

//chat property
//io object
chat.io = false;
//user name
chat.userName = {};
//name has been used
chat.usedName = [];
//user number
chat.userNum = 0;
//current room name
chat.currentRoom = {};
</pre>

<p>
	新建initialize方法来初始化	
</p>

<pre>
//chat initialization with the passing http object
chat.initialize = function(http) {
	this.io = socketIo(http);
	this.ioListen();
}
</pre>

<p>
	新建ioListen方法作为最主要的监听方法
</p>

<pre>
// major socket listening method
chat.ioListen = function() {
	
	var that = this;

	this.io.on('connection', function(socket){
		
		that.assignRoom(socket);

		that.changeRoom(socket);

		that.sysMsg(socket);

		that.userMsg(socket);

		that.assignGuestName(socket);

		that.changeName(socket);

		that.disconnect(socket);

	});
}
</pre>

<p>
	下面，我们对每个处理的方法进行剖析。
</p>

<pre>
//assign room 'Lobby' once they enter
chat.assignRoom = function(socket) {
	
	var that = this;
	socket.join('Lobby', function(){
		that.currentRoom[socket.id] = 'Lobby';
	});
}
</pre>

<p>
	assignRoom方法顾名思义就是自动给新加入的用户分配房间。有socket.join直接输入聊天室的名字就可以了，之后可以直接调socket.io自带的方法来得知道应该向用户所在的聊天室进行消息广播。然后用currentRoom存放用户聊天室的名字。
</p>

<pre>
//chat.js change room
chat.changeRoom = function(socket) {

	var that = this;

	socket.on('change room', function(msg){

		var sysMsg = that.userName[socket.id] + ' left room ' + that.currentRoom[socket.id];

		that.io.to(that.currentRoom[socket.id]).emit('sys message', sysMsg);

		socket.leave(that.currentRoom[socket.id], function(){

			socket.join(msg);

			that.currentRoom[socket.id] = msg;

			sysMsg = that.userName[socket.id] + ' join room ' + that.currentRoom[socket.id];
			
			socket.emit('sys message', sysMsg);

			socket.emit('change room name', msg);

		});

	});

}
</pre>

<p>
如果要更改房间，就可调用changeRoom方法。this.io.to(roomName)就是利用socket.io自带的函数给用户所在的聊天室进行消息广播。然后socket.leave即是离开聊天室的意思。
</p>

<pre>
// send user message
chat.userMsg = function(socket) {

	var that = this;

	socket.on('chat message', function(msg){
		msg = that.userName[socket.id] + ' said: ' + msg;

		that.io.to(that.currentRoom[socket.id]).emit('chat message', msg);
	});

}

//send system message
chat.sysMsg = function(socket) {

	var that = this;

	socket.on('sys message', function(msg){
		that.io.to(that.currentRoom[socket.id]).emit('sys message', msg);
	});	
	
}
</pre>

<p>
chat.userMsg 和 chat.sysMsg分别是发送用户消息和系统消息的方法。他们都用到了上文提到的this.io.to进行聊天室广播。
</p>

<pre>
//assign a guest name to new joining user
chat.assignGuestName = function(socket) {

	this.userName[socket.id] = 'Guest' + this.userNum;
	this.usedName.push('Guest' + this.userNum);
	this.userNum++;

	var msg = this.userName[socket.id] + ' enter the room! Welcome!';

	this.io.emit('new user', msg);

}

//change user name
chat.changeName = function(socket) {

	var that = this;

	socket.on('change name', function(msg){
		if (that.usedName.indexOf(msg) == -1) {

			var nameIndex = that.usedName.indexOf(that.userName[socket.id]);
			that.userName[socket.id] = msg;
			that.usedName[nameIndex] = msg;
			socket.emit('sys message', 'Your name has been changed as ' + msg);
		}
		else {
			socket.emit('sys message', 'Your name has been used');
		}

	});
}
</pre>

<p>
	assignGuestName和changeName和之后assignRoom的changeRoom相当类似，。通过userName存放用户姓名，usedName判断是否有重名，然后利用this.io.emit去解放系统消息。
</p>

<p>
	后台chat.js的代码基本介绍完毕。然后我们将处理前端的代码。
</p>

<p>
	然后新建/chat/public/index.html并录入HTML代码（由于github会过滤部份HTML的关系，请到代码管理的/public/html/里面抓取代码。
</p>

<p>
前后端的代码基本录入完成后，我们将结合两者进行说明。首先，我们要理解,方法socket.emit和socket.on是相对应而存在的，on是对event的监听，emit是对event的触发。例如，在前端，我们通过
<pre>
socket.emit('chat message', msg);
</pre>
触发聊天消息的事件。
在后端我们通过
<pre>
socket.on('chat message', function(msg){
	that.userMsg(socket, msg);
});
</pre>
进行聊天消息事件的监听。在chat.userMsg中，我们通过
<pre>
this.io.to(this.currentRoom[socket.id]).emit('chat message', msg);
</pre>
进行聊天消息事件的触发，在前端的代码中，我们通过
<pre>
socket.on('chat message', function(msg){
    showMsg(msg);
});
</pre>
进行聊天消息的监听并显示到浏览器中。socket.emit和socket.on是一一对应的，上面前后端代码结合起来形成了整个消息从前端输入到后台处理，再从后台推送到其它用户前台的整个流程。
</p>

<h3><a name="pieces">4. 坑，坑，还是坑</a></h3>
<p>
	在做的过程中，我还是遇到不少坑值得大家以后留意的。
</p>

<h4>坑1. 对于socket以及io两种对象的理解</h4>
<p>
能过下面语句其实我们可以发现socket实质是io.socket。所以socket其实只是io对象的一个子对象。Socket中文译作插座的意思，实质上只是负责监听io对象的某一通道而已。
</p>
<pre>

this.io.on('connection', function(socket){
	// doing something
});
</pre>
<p>通过下面代码相信大家可以知道两者区别</p>
<pre>
	this.io.emit('xxx', msg);
	//触发对所有通道发送信息的事件
</pre>
<pre>
	this.io.socket('xxx', msg);
	//触发仅对自己通道的事件
</pre>
<pre>
	this.io.to(room).emit('xxx', msg);
	//触发对某一房间的事件
</pre>

<h4>坑2. 对输入内容的处理</h4>
<p>
由于这是一个需要处理用户输入输出的聊天室，因此，非常需要对用户的输入输出进行过滤,否则很容易会受到xss攻击。这里socket.io似乎已经对用户的输入输出过滤，但并不代表开发者不需要注意这一点。尤其是前端的同学，可能会忽略这个通常由后端进行处理的点。
</p>

<h4>坑3. 并不是所有浏览器都支持Websocket</h4>
<p>
虽然socket.io自诩支持所有浏览器，其实只是支持比较新的现代浏览器，如果想测试你的浏览器是否支持websocket可以访问下面网站进行测试。如果不支持，只好用comet作为替代方案。
<a href="http://www.websocket.org/aboutwebsocket.html" target="_blank">http://www.websocket.org/aboutwebsocket.html</a>
</p>
<p>
<img src="https://github.com/lcxfs1991/chat-socket.io/blob/master/websocket_support.png" >
</p>








