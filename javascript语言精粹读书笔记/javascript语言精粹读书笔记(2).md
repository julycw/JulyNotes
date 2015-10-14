## javascript语言精粹读书笔记（2）

### 关于new
如果 new 关键字是一个函数，那它大概做了下面这些工作

	Function.prototype.new = function(){
	     var that = Object.create(this);//根据当前原型创建一个新的对象
	     var other = this.apply(that,arguments);//用原型构造方法初始化新的对象
	     return (typeof other === ‘object’ && other) || that;
	}

### 一个约定
约定需要用new关键字去构造的function对象，其原型名称必须以首字母大写命名。
因为如果忘记写new，就会变成运行函数，里面的this也会变成全局this。这个错误在编译和运行时都难以发现，所以要首字母大写人肉排错。
