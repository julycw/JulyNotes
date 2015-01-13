#Macaron入门 （二）

##Hello World!

由于我对[Macaron](https://github.com/Unknwon/macaron) Web框架也是接触不久，那么就不要免俗了，抱着初学者的心态上个Hello world看看吧：）

新建一个main.go文件，内容：
	
	package main

	import (
		"github.com/Unknwon/macaron"
		"log"
	)

	func main() {
		m := macaron.New()

		m.Use(macaron.Logger())
		m.Use(macaron.Recovery())
		m.Use(macaron.Static("public"))

		m.Get("/", func(ctx *macaron.Context, logger *log.Logger) (int, string) {
			logger.Printf("Get request from : %s \n", ctx.RemoteAddr())
			return 200, "Hello world!"
		})
		m.Run()
	}

然后打开[刚写的网站](http://127.0.0.1:4000)，OK，运行良好。

##Hello World! (官方版)

[Macaron](https://github.com/Unknwon/macaron)官方给出的栗子要比我的更加简单。

	package main

	import (
		"github.com/Unknwon/macaron"
	)

	func main() {
		m := macaron.Classic()
	    m.Get("/", func() string {
	        return "Hello world!"
	    })
	    m.Run()
	}

main函数中第一行就不一样，我通过macaron.New()来获得macaron的实例，而官方栗子使用macaron.Classic()，两者有什么不同呢？答案是，除了用到的中间件不同，其他没什么不同的。

去看看源码，macaron.Classic()做了什么：

	func Classic() *Macaron {
		m := New()
		m.Use(Logger())
		m.Use(Recovery())
		m.Use(Static("public"))
		return m
	}

看到这些，我只想说，继续，一点也不难！由于这个篇章只是入门而已，就不深入源码分析了（其实是我昨天才把源码理顺，怕讲不清楚...），之后一定会把所有源码都过一遍。

##Macaron的处理器

继续看下面的代码。

	m.Get("/", func() string {
        return "Hello world!"
    })

以及

	m.Get("/", func(ctx *macaron.Context, logger *log.Logger) (int, string) {
		logger.Printf("Get request from : %s \n", ctx.RemoteAddr())
		return 200, "Hello world!"
	})

如果你接触过一类MVC框架，应该一眼就能看出是在进行路由绑定的工作了。即使不看任何文档和源码，也可以猜出Get方法是用来绑定一个Get请求的路由，其中第一个参数指定了路由匹配规则，第二个参数是当该路由被匹配后将被执行的处理器。

>处理器是 Macaron 的灵魂和核心所在.

到目前为止，这一切都显得理所当然，和其他MVC框架没什么两样。But！仔细一看，那么问题来了：这个处理器是什么情况，难道传入参数和返回值都那么任性么？这种奇怪的处理函数到底是怎么样被正确调用的呢？

如果还记得[上一篇](https://github.com/julycw/JulyNotes/blob/master/%E5%AD%A6%E4%B9%A0Macaron/Macaron%E7%9A%84%E5%9F%BA%E7%A1%80%20-%20%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5.md)中介绍的依赖注入框架Injector的话，应该很容易想到Invoker接口中的Invoke方法。没错，Macaron的处理器方法是通过Invoke方法来进行调用的。

那么，处理器中的传入参数究竟是从哪儿来的呢，这时候再去偷偷看一眼源代码。原来，当我们对网站发起访问，且路由被匹配的时候，Macaron实例会调用他自身的createContext方法来创建一个基本的请求上下文对象 macaron.Context。注意，请求上下文这个结构体 __继承__ (换个词可能更好，一时想不起来)了Injector，因此也具备了依赖注入的能力。

看看官方介绍：

> Macaron 会注入一些默认服务来驱动您的应用，这些服务被称之为核心服务。也就是说，您可以直接使用它们作为处理器参数而不需要任何附加工作。

>该服务通过类型 *macaron.Context 来体现。这是 Macaron 最为核心的服务，您的任何操作都是基于它之上。该服务包含了您所需要的请求对象、响应流、模板引擎接口、数据存储和注入与获取其它服务。

再来看看createContext方法的源代码：

	func (m *Macaron) createContext(rw http.ResponseWriter, req *http.Request) *Context {
		c := &Context{
			//构造参数
		}
		c.SetParent(m)
		c.Map(c)
		c.MapTo(c.Resp, (*http.ResponseWriter)(nil))
		c.Map(req)
		return c
	}

这下清楚了，看第六行 `c.Map(c)` ，这就是处理器方法中的传入参数 ctx (*macaron.Context) 的由来了。

继续看第七行和第八行，显然，`c.Resp` 和 `req` 也都可以直接在处理器方法中使用，只要将他们加入到传入参数列表就可以了，他们的类型分别是 `http.ResponseWriter` 和 `*http.Request`。为了证明这一点，我修改了之前的main.go文件，主要变动在于处理器方法。我将传入参数由`ctx *macaron.Context, logger *log.Logger`改为了`rw http.ResponseWriter, req *http.Request, logger *log.Logger`，并去掉了返回值，在方法体中打印了User agent信息，并通过ResponseWriter写入了头部状态信息“200”，以及内容“Hello world!”。main函数最终修改如下。

	m := macaron.New()
	m.Use(macaron.Logger())

	m.Get("/", func(rw http.ResponseWriter, req *http.Request, logger *log.Logger) {
		logger.Println("User agent:", req.UserAgent())
		rw.WriteHeader(200)
		rw.Write([]byte("Hello world!"))
	})
	m.Run()

运行，[访问](http://127.0.0.1:4000)，网页上显示 “Hello world!”，控制台也正确的打印出了User agent，和预期一模一样。这也就证明了依赖注入的工作顺利完成了!

这感觉，就像写脚本语言一般酸爽！


