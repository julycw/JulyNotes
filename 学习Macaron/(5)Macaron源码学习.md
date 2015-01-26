#Macaron源码学习篇 （五）

这一篇，对Macaron源码的学习接近尾声了，之后会学习几个常用的中间件。

##上篇回顾
在上一篇文章中，说到特定方法的路由和相应的处理函数已经正确的存入到了路由树之中了，因此可以将整个Macaron对用户请求的处理流程最简化成2步：

1. 匹配路由获取相应处理器
2. 执行处理器

就是这么简单。

##处理器
上面说到的处理器，其实是包括了所有中间件的处理方法以及用户在注册路由时注册的处理器。这个包络了其他处理器的处理器首先创建了Macaron Context对象，之后将Params和Handlers统统绑定到这个对象上面，然后调用Context的run方法，run方法完成用户的业务逻辑，最终将输出信息写入到Response的流之中，整个请求完成。

另外，所有的处理器最终依次执行。是的，依次执行，因此如果中间件A依赖中间件B的话，需要先注册中间件B，再注册中间件A，不然的话会报错！

#Macaron Context
直接看run方法：
	
	for c.index <= len(c.handlers) {
		vals, err := c.Invoke(c.handler())
		if err != nil {
			panic(err)
		}
		c.index += 1

		// if the handler returned something, write it to the http response
		if len(vals) > 0 {
			ev := c.GetVal(reflect.TypeOf(ReturnHandler(nil)))
			handleReturn := ev.Interface().(ReturnHandler)
			handleReturn(c, vals)
		}

		if c.Written() {
			return
		}
	}

总的来看，是一个while循环，注意循环跳出条件是c.index <= len(c.handlers)，是小于等于而不是小于。这是因为除了遍历handlers以外，context还有一个额外的handler叫action，action会在所有的中间件执行完之后再执行。

循环内部，总的来看分为三步。代码的空行也刚好把代码块分成了三个部分，不错的代码分隔：） 

1. context的handler()方法会根据当前的index返回对应handler。如果index超出handlers的下标返回，则返回action。
Invoke方法会动态调用handler，返回的结果保存在vals和err中。之后index自增。
2. 如果handler有返回值，则调用专门的返回值处理函数去处理返回值。这里，我想可以将HTML的渲染器放这，不知道是不是个好主意。
3. 最终，判断是否向response写过值，如果写过，就不继续之后的中间件。

#小技巧
在处理器中调用context的Next方法，会迫使Next之后的代码延期到所有中间件之后执行。
用伪代码解释：
	
	func_1(){
		FuncAIn1()
		c.Next()
		FuncBIn1()
	}

	func_2(){
		FuncAIn2()
		c.Next()
		FuncBIn2()
	}

调用顺序：
1. FuncAIn1()
2. FuncAIn2()
3. FuncBIn2()
4. FuncBIn1()

实现的方式很巧妙（至少对我来说）：

	func (c *Context) Next() {
		c.index += 1
		c.run()
	}

真是超级简单！都懒得解释啦。
