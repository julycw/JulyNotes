## javascript语言精粹读书笔记（1）

### 符号 || 可以用来填充默认值

比如
var name = $(“get”).val() || “unknown”;
它的作用就是当左边的表达式为undefined时，用右边的值替代左边

能这么做的原因如下：

在js中，|| 和 && 并不会返回一个boolean值，而是会返回一个符合结果字面值的某一个运算元。具体规则很简单，这2个运算符都是短路运算符，每个参数依次判断，何时短路（或全部判定结束），就返回短路的那个值（或最后那个值）。
比如
var a = 3, b = false, c;
a||b === 3 // a为true,直接返回3
b||a === 3 // b为false,继续判断a,a为true,返回3
b||c === undefined // b为false,继续判断c,c也为false，此时返回c，为undefined

&&也是这个规则
var d = 5;
a&&d === 5 // a为true,继续判断d,d也为true, 此时返回d，为5
b&&d === false // b为false，直接返回false
a&&b&&c&&d === undefined // c时条件不成立短路，返回c


### 最好不要用new关键字直接去实例化对象
     if (typeof Object.beget !== 'function') {
          Object.beget = function(o){
               var F = function(){};     
               F.prototype = o;
               return new F();
          }
     };

     var Car = {
     }

     var myCar = Object.beget(Car);


### 关于this
this在不同情况下被绑定到不同的对象

* [1]方法调用
如：
     obj.func = function(){
          //this
     }
     obj.func();

此时this绑定到该方法所绑定的对象,即obj

* [2]函数调用
如：

     var func = function(){
          //this
     }
     func();

此时this绑定到全局变量,这是一个js语言的设计错误,按理说应该绑定到最近的外层的this。

* [3]构造器调用
如:

     var Car = function(){
          //this
     }
     var myCar = new Car();
此时this会指向新构造出来的对象，即myCar.
new关键字会构造一个新的对象实例，同时背地里会将这个实例连接到这个函数的prototype成员。即当Car的prototype改变时，同时影响Car的所有实例。

* [4]apply/call调用
apply方法接受2个参数，第一个参数是要被绑定到this的对象，第二个参数是一个参数列表。
举个例子。

     var Car = function(name){
          this.name = name;
     }
     Car.prototype.run = function(){
          console.log(this.name + “ is running");
     }

     var boy = {
          name:”Xiao Ming"
     }

     Car.prototype.run.apply(boy);

虽然boy并没有继承自Car.prototype，但还是可以正确执行，并且this绑定到了boy
输出：Xiao Ming is running

apply和call的区别是apply第二个参数只能接受一个数组，而call可以传入任意参数


### 函数内部特殊成员变量 
除了this以外还免费附赠arguments对象，它看起来像参数列表，但是他并不是真正的Array( instanceof Array === false ).他有一个length成员，但是除此之外没有任何数组的方法.


### 函数必定会有返回值。
如果没有return，返回undefined，如果函数调用前有new关键字，返回this。


### js中抛出异常：

     throw{
          name:””,
          message:””
     }
     throw会中断函数执行.异常会被catch，和一般的try>catch异常处理相似

     try{
     //do something
     }catch(e){
          //e.name
          //e.message
     }


### typeof 和 instanceof 

typeof是一元运算符，返回对象的类型的字符串表示

     var a = 1; // typeof a === ’number'
     var b = “hello”; // typeof b === ’string'
     var c = true; // typeof c === ‘boolean'
     var d = {}; // typeof c === ‘object'
     var e = []; //typeof e === ‘object'
     var f = function(){} //typeof f === ‘function'
     var g; // typeof g === ‘undefined'

instanceof是二元运算符，返回布尔值
a instanceof b,我的理解为判断b是否是a的原型链中的一环。用面向对象的话说就是判断a是不是b的子类。

     var list = [];
     //list instanceof Array === true
     //list instanceof Object === true
     //list instanceof Number === false

这里还能发现一个规律，js里面的基本类型”type”都是小写的字符串，而原型类都为首字母大写。



### 利用闭包特性的单例模式简单实现

     var getSingleTool = (function(){
          var unique;
          function Construct(){
               //private
               var name = "private name";

               //public
               this.getName = function(){
                    return name;
               }
               this.setName = function(newName){
                    name = newName;
               }
          }

          unique = new Construct();
          return function(){
               return unique;
          }
     })();

     var tool = getSingleTool();


### Array.prototype.slice可以让function中的arguments变成真正的数组
用法：

     var func = function(){
          var slice = Array.prototype.slice;
          arguments = slice.apply(arguments)
     }

### 坷里化例子

     var aPlusBplusC = function(a, b, c){
          var result = a + b + c;
          return result;
     }

     Function.prototype.curry = function() {
          var slice = Array.prototype.slice;
          var args = slice.apply(arguments);
          var that = this;
          return function(){
               return that.apply(null, args.concat(slice.apply(arguments)));
          }
     };

     var fivePlusAPlusB = aPlusBplusC.curry(5);
     var result = fivePlusAPlusB(4, 6); // 15


