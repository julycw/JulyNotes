#Macaron源码学习篇 （三）

本文唯一的主线其实就是代码执行的顺序，偶尔穿插一些其他内容。第一次这么记录源码学习过程，能坚持下来就是胜利。

##从入口开始

打开macaron.go,找到`func (m *Macaron) Run(args ...interface{})`,这个方法在之前的hello world里面调用过。这边就不贴源码了，因为看一眼这个方法就会发现这也只是一个简单的封装而已，没什么好说的，关键点在于最后一行`logger.Fatalln(http.ListenAndServe(addr, m))` , macaron将自己的上下文对象作为一个参数传入了go的原生方法 `http.ListenAndServe(addr,handler)`。

为什么可以这么做呢，看看ListenAndServe的参数列表，第一个参数是一个字符串，用来指定监听地址（端口），第二个参数是一个handler，类型为 http.Handler接口。问题的答案很明显了，因为macaron实现了所有http.Handler接口，所以它可以无缝对接了：）。

补充一下，go语言中，实现一个接口不需要事先申明，只要实现了特定函数签名的函数，即认为拥有了这个接口。这算是go语言一个特色，我还是蛮喜欢的。

现在来看看http.Handler有哪些接口，其实就一个，`ServeHTTP(rw http.ResponseWriter, req *http.Request)`，OK，马上转向macaron.go文件中，果然也有这么一个函数签名完全一致的方法。

	// ServeHTTP is the HTTP Entry point for a Macaron instance.
	// Useful if you want to control your own HTTP server.
	// Be aware that none of middleware will run without registering any router.
	func (m *Macaron) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
		req.URL.Path = strings.TrimPrefix(req.URL.Path, m.urlPrefix)
		for _, h := range m.befores {
			if h(rw, req) {
				return
			}
		}
		m.Router.ServeHTTP(rw, req)
	}

看我对注释的渣翻译：
>对Macaron实例而言，ServeHTTP方法是HTTP的入口点。如果你想控制自己的HTTP服务，这会很有用。注意，如果没有注册任何路由的话，那么任何中间件都不会运行。

可以看到，每当一个请求发起的时候，req.URL.Path 都被砍一刀，不是很环保。。

然后，所有已注册的BeforeHandler被挨个调用，如果任何一个BeforHandler返回为true，那么这个请求就到此为止了。这一点在官方的文档中也有提到，但是在代码中看到会觉得更加真切。

接下来，路由开始发挥作用，可以预见，Router做了很多，至少在Router的ServeHTTP里边，这个请求全部被完成了。

为了学的更加透彻一点，我必须去看看http.ListenAndServe做了什么。结果总结如下：
1. 接受的两个参数被用来构造Server对象
2. Server对象调用ListenAndServe()方法
3. Server对象判断监听端口，如果为空，则端口被设置为:http（80端口）, 然后根据这个端口获取了Listener
4. Server对象调用Serve方法，进入主循环
5. 每当listener有了accept，起一个goroutine跑Server对象的serve方法
6. macaron的ServeHTTP开始运行（net/http/server.go 1170行左右）

