#先说好
这不是Step by step的路子，只不过是我自己记录一些值得注意的点和一些不是很明显的细节的备忘录。

#基本知识
Pomelo是网易的一个开源游戏服务端框架，基于Node.js开发。

我的学习路线主要是基于其一个名叫ChatOfPomelo的聊天室程序而展开的。

关于Node.js、Pomelo以及ChatOfPomelo环境搭建的文章很多，这边就不写了，我自己忘了的话会去Pomelo的github上看一下，总之很简单。

另外，由于条件有限，我把环境搭载在Ubuntu虚拟机中，一切例子我都在本机进行访问，所以对ChatOfPomelo的访问都并非127.0.0.1或是localhost，这导致ChatOfPomelo的原本配置无效，需要进行一些小小的改动。

首先定位到ChatOfPomelo根目录，然后
	
	sudo vi game-server/config/servers.json
	:%s/127.0.0.1/虚拟机IP/g
	:wq

现在，分别启动game-server和web-server，就可以体验这个聊天室了。

# 启动 game-server
查看app.js:

	var pomelo = require('pomelo'); // 从node_modules文件夹中加载pomelo模块
	var routeUtil = require('./app/util/routeUtil'); // 从app.js相同目录下加载./app/util/routeUtil模块

当pomelo模块被加载时，下面的逻辑会被执行：

	/**
	 * Get application
	 */
	Object.defineProperty(Pomelo, 'app', {
	    get:function () {
	        return self.app;
	    }
	});

Object.defineProperty函数：向对象添加属性或修改现有属性的特性。通俗一些理解，就是当我们访问 Pomelo.app时，将会调用get函数，返回self.app。

另一个需要注意的地方是：

	/**
	 * Auto-load bundled components with getters.
	 */
	//遍历目录下的所有文件
	fs.readdirSync(__dirname + '/components').forEach(function (filename) {
		//如果不是.js结尾的文件，直接return跳过
	    if (!/.js$/.test(filename)) {
	        return;
	    }
	    //获取文件名，不包含.js
	    var name = path.basename(filename, '.js');
	 	//定义函数对象
	    function load() {
	        return require('./components/' + name);
	    }
	    //绑定gatter
	    Pomelo.components.__defineGetter__(name, load);
	    Pomelo.__defineGetter__(name, load);
	});
	
	//和上面差不多
	fs.readdirSync(__dirname + '/filters/handler').forEach(function (filename) {
	    if (!/.js$/.test(filename)) {
	        return;
	    }
	    var name = path.basename(filename, '.js');
	 
	    function load() {
	        return require('./filters/handler/' + name);
	    }
	    Pomelo.filters.__defineGetter__(name, load);
	    Pomelo.__defineGetter__(name, load);
	});

基本作用就是载入相应的模块吧，具体原理写在注释里了。

接下来的代码是：

	var app = pomelo.createApp();

这里边代码比较多，基本上都是读取配置文件等等的工作，为开启服务做好准备工作。
