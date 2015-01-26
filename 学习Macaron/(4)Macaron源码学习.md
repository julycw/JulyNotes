#Macaron源码学习篇 （四）

上回说到Router的ServeHTTP才是真正处理了请求。那么，沿着这条线路继续看源码先。

	func (r *Router) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
		if t, ok := r.routers[req.Method]; ok {
			h, p := t.Match(req.URL.Path)
			if h != nil {
				if splat, ok := p[":splat"]; ok {
					p["*"] = p[":splat"] // Better name.
					splatlist := strings.Split(splat, "/")
					for k, v := range splatlist {
						p[com.ToStr(k)] = v
					}
				}
				h(rw, req, p)
				return
			}
		}

		r.notFound(rw, req)
	}

这就是Router的ServeHTTP。整个处理流程大概是这样的：

1. 首先根据请求的Method（GET/POST等等之一）获取routers中对应的tree。tree保存在哪里呢？Router结构体中存在一个routers字段，类型是键为字符串、值为*Tree的map。Router结构体扮演了Macaron中的路由层。

2. 获取到的Tree调用Match方法去匹配用户请求的路由，匹配到了会返回一个handler处理函数，以及包含在路由中的参数。如果用户设置的路由是 *（也就是任意匹配），那么路径会被以"/"号分割，其中每一段都将作为参数值保存起来，下标以数字字符串“0”开始，“1”、“2”以此递增。

3. 执行handler，并传入三个参数：http.ResponseWriter、*http.Request、params

4. return结束。如果没有匹配到路由，则报出404错误。

写到这边，Macaron的执行逻辑基本上已经清楚了，剩下的无非就是路由如何注册、中间件如何注册、路由具体如何去匹配这些问题。等等，还没看看handler里面做了什么呢，继续。

先来看看匹配到对应路由后，返回的第一个值Handle是什么

	// Handle is a function that can be registered to a route to handle HTTP requests.
	// Like http.HandlerFunc, but has a third parameter for the values of wildcards (variables).
	type Handle func(http.ResponseWriter, *http.Request, Params)

我可不曾记得我曾经（也就是注册中间件和路由的时候）传入过带有这种函数签名的函数指针给Macaron，那这究竟是怎么来的呢，现在该回过头去看看注册路由的时候Macaron做了哪些处理。

Macaron中路由注册非常简单，通过Macaron上下文对象调用`Get(pattern string, h ...Handler)`、Post、Put、Delete等等即可注册相应Method的路由。第一个参数必备，为路由规则，之后是不定长个数的Handler，Handler是什么呢，回过头去入门篇看看，在Macaron中Handler被称为处理器，是整个Macaron的灵魂和核心。有一种情况是，如果我只写了路由规则，而不传入任何处理器，那这个路由有意义吗？当然有，别忘了Macaron的中间件。任何路由都会被所有已注册的中间件处理一遍，也许中间件已经做了足够的事情，因此处理器不是必须的。

继续看，无论Get、Post等等方法，都是调用了`Handle(method string, pattern string, handlers []Handler) `这个方法。

	// Handle registers a new request handle with the given pattern, method and handlers.
	func (r *Router) Handle(method string, pattern string, handlers []Handler) {
		if len(r.groups) > 0 {
			groupPattern := ""
			h := make([]Handler, 0)
			for _, g := range r.groups {
				groupPattern += g.pattern
				h = append(h, g.handlers...)
			}

			pattern = groupPattern + pattern
			h = append(h, handlers...)
			handlers = h
		}
		validateHandlers(handlers)

		r.handle(method, pattern, func(resp http.ResponseWriter, req *http.Request, params Params) {
			c := r.m.createContext(resp, req)
			c.params = params
			c.handlers = append(r.m.handlers, handlers...)
			c.run()
		})
	}

看到注释咯，原来是Handle方法接受了请求的method、pattern以及处理器之后，为我们注册了一个签名为`func(http.ResponseWriter, *http.Request, Params)`的方法，这个方法会在用户请求发送并被路由表匹配后执行。

再来仔细看看其中的代码，首先判断了r.groups的长度，注意这里很容易被误导，会不假思索的以为这个长度是groups的个数，如果这么想了，那后面的代码就看不懂了。。。 其实，这里的长度是当前Handle所在groups的深度。于是乎就很好理解了，将pattern和各层group的pattern组合成新的正确的pattern，将当前处理器组（处理器可能不止一个）和各层group的handlers组合成新的处理器组。

而后，验证所有handlers符合要求（其实只要是函数，就符合要求 , **Panic if the handler.Kind() is not reflect.Func**）。

再然后，猜测一下
	
	r.handle(method, pattern, func(resp http.ResponseWriter, req *http.Request, params Params) {
		c := r.m.createContext(resp, req)
		c.params = params
		c.handlers = append(r.m.handlers, handlers...)
		c.run()
	})

这段代码应该是真正把路由和相应的处理函数存入到tree（之前提过，还记得Match的过程么？）中。

注意第三个参数，也就是这个匿名的处理函数他会做哪些事：
1. 首先创建了一个context对象，这个对象的生命周期基本上就是伴随用户的一次请求。
2. 为这个context配置参数和处理器。
3. 运行run()。

注意，当前只是注册这个处理函数而已，并没有运行。具体什么时候运行，上面有提到！

好了，离揭开macaron最后的面纱越来越接近了....^_^
