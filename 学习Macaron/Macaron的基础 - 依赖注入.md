#Macaron的基础 - 依赖注入（一）

最近想学习[Macaron](https://github.com/Unknwon/macaron)，但是不太清楚从哪方面入手比较好。看完文档，印象最深刻的是对中间件的应用。使用中间件的好处是功能模块化、代码结构清晰，我个人非常喜欢。

举个例子：

	m := macaron.New()
    m.Use(macaron.Logger())
    m.Use(macaron.Recovery())
    m.Use(macaron.Static("public"))
    m.Get("/", func() string{
	    return "hello world."
	})

在上面这段代码中，使用了Logger日志中间件、Recovery错误恢复中间件以及Static静态文件服务中间件，先不用纠结为什么这么写以及如何使用这些中间件，只要知道不同中间件可以随意拼装使用就可以了。

##1. 中间件技术的关键

终于到了正题了：）

直奔主题，中间件技术的关键所在就是依赖注入，Macaron使用了一个叫[Inject](https://github.com/codegangsta/inject)的依赖注入框架，非常轻量级，算上注释和空行，代码一共不到200行，而我的Macaron学习之路正是从这个小小的框架开始！

那么这个叫Inject的东西做些什么工作呢？简单来讲，它组织起了**类型**和**值**的映射(Mapping)关系，然后动态地去构造结构体、调用函数。实际上，Inject维护一个键、值分别为reflect.Type和reflect.Value的Map，然后根据结构体和函数体，去动态地构造或调用。举个通俗点的栗子，**值**好比一副牌，构造结构体(调用函数也一样)的过程好比你对牌局形式的判断，你要用哪个**值**，就得见招拆招。可能不是很贴切，还是看点实际的吧 =。=

##2. 源代码分析

直接看源代码：

	type Injector interface {
		Applicator
		Invoker
		TypeMapper
		SetParent(Injector)
	}

	type Applicator interface {
		Apply(interface{}) error
	}

	type Invoker interface {
		Invoke(interface{}) ([]reflect.Value, error)
	}

	type TypeMapper interface {
		Map(interface{}) TypeMapper
		MapTo(interface{}, interface{}) TypeMapper
		Set(reflect.Type, reflect.Value) TypeMapper
		GetVal(reflect.Type) reflect.Value
	}

对于我们初学者来说，先搞清楚这4接口之间的关系肯定是不会错的。
**Injector**是主要接口，它起到串联的功能，主要工作由**Applicator**、**Invoker**、**TypeMapper**来执行。**SetParent**就不提了，看这名字你应该也大概清楚它是什么作用，代码更是简单到只有一行。

然后主角登场

	type injector struct {
		values map[reflect.Type]reflect.Value
		parent Injector
	}

**injector**结构体包含了**Injector**接口，还有一个类型为map[reflect.Type]reflect.Value的字段。整个注入过程就好比装弹、射击，如果说**Applicator**和**Invoker**的行为是射击，那么**TypeMapper**就是装弹，而**values**则是弹匣。这个比喻应该没问题，那么我就按照装弹、发射的顺序来继续接下来的内容好了：）

###2.1 装弹

装弹的任务由**TypeMapper**来完成，这个接口实现了四个功能：

	Map(interface{}) TypeMapper
	MapTo(interface{}, interface{}) TypeMapper
	Set(reflect.Type, reflect.Value) TypeMapper
	GetVal(reflect.Type) reflect.Value

前三个方法将Type、Value键值对保存起来，最后一个方法根据Type取出Value。其中**Map(interface{})**传入一个类型为interface{}的值，直接通过reflect.TypeOf获取interface{}的类型，通过reflect.ValueOf获取interface{}的值，存入map。**MapTo(interface{}, interface{})**功能也差不多，注释上是这么写的：
>Maps the interface{} value based on the pointer of an Interface provided.This is really only useful for mapping a value as an interface, as interfaces cannot at this time be referenced directly without a pointer.

按我的理解就是，如果一个结构体包含N个类型为string的字段，这时就必须将各个string字段的类型当做不同的interface{}，否则“子弹”分不清种类。这个之后在学习Macaron的过程使用到了MapTo以后再进行验证吧。

**Set(reflect.Type, reflect.Value)**和**GetVal(reflect.Type)**非常简单直观，就不讲了。最后，注意到这几个方法的返回值都为**TypeMapper**，这就意味着可以进行Chain式调用，十分方便。


###2.2 发射

Injector既可以构造(Apply)结构体，也可以调用(Invoke)函数。依赖注入的神奇之处在于：

1. 构造结构体之前，我们不知道结构体有哪些字段，哪个字段的值该是什么。
2. 调用函数之前，我们不知道函数需要哪些参数，又会返回哪些参数。

这一切，都在运行时才能确定。

####2.2.1 发射方式一 Apply

Applicator接口只有Apply这一个功能，它将Type/Value映射表(injector中的values)中的值映射到具体结构对象的字段中。映射功能可以通过结构体字段中的tag来进行开关控制。如果注入操作失败，则返回error。通过Apply，可以动态创建结构体对象。代码如下：
	
	func (inj *injector) Apply(val interface{}) error {
		v := reflect.ValueOf(val)

		for v.Kind() == reflect.Ptr {
			v = v.Elem()
		}

		if v.Kind() != reflect.Struct {
			return nil // Should not panic here ?
		}

		t := v.Type()

		for i := 0; i < v.NumField(); i++ {
			f := v.Field(i)
			structField := t.Field(i)
			if f.CanSet() && (structField.Tag == "inject" || structField.Tag.Get("inject") != "") {
				ft := f.Type()
				v := inj.GetVal(ft)
				if !v.IsValid() {
					return fmt.Errorf("Value not found for type %v", ft)
				}

				f.Set(v)
			}

		}

		return nil
	}

代码很简单，首先检查val的类型，如果不是struct类型，直接返回nil。遍历对象字段，判断该字段是否“CanSet”(关于CanSet是什么意思，可以看[这里](http://my.oschina.net/ffs/blog/300267?p=1))，如果结果为真，且Tag标有“inject”，那么根据val的类型从“弹匣”中取出对应的“子弹”，调用Set方法装入对象中。


####2.2.2 发射方式二 Invoke

Invoker接口也只有一个功能，Invoke。它获取func的参数列表，同Apply一样，去“弹匣”中取出对应的“子弹”。最后调用Call来动态执行函数。代码如下：

	func (inj *injector) Invoke(f interface{}) ([]reflect.Value, error) {
		t := reflect.TypeOf(f)

		var in = make([]reflect.Value, t.NumIn()) //Panic if t is not kind of Func
		for i := 0; i < t.NumIn(); i++ {
			argType := t.In(i)
			val := inj.GetVal(argType)
			if !val.IsValid() {
				return nil, fmt.Errorf("Value not found for type %v", argType)
			}

			in[i] = val
		}

		return reflect.ValueOf(f).Call(in), nil
	}

OK，今天先到这了！


