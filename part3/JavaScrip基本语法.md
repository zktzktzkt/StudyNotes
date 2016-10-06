#JavaScrip基本语法#
---

##一、变量类型以及基本运算##

js属于动态语言，变量是无类型的，可以被赋予任何类型的值使用关键字`var`来声明变量，值得注意的js会将var声明的变量提前至函数顶部，变量在声明前就可以使用，使用typeof关键字可以获取变量类型，变量可分为以下两种类型

* 基本类型或者原始类型：
	数字(js不区分整数与浮点值，统一使用浮点值)、字符串、布尔值、null(空)、undefined(未定义)

* 引用类型或者对象类型:
	初基本类型之外的数据类型都是引用类型，引用类型是属性的集合，而属性由键值对构成,***此外函数在js中也是一种对象***，此外对象的属性除了通过***.***访问之外还可以通过**[属性]**的形式来访问

* 部分预定义的全局变量与函数:
	Number、String、Date、Object、JSON、Math、Infinity、NaN、isFinite、isNaN、RegExp(正则式的工具对象)等等

* 算术运算：
	基本运算类似于java，同时也提供了全局对象Math来进行复杂的运算，同时js预定了全局变量Infinity、NaN表示无穷大与非数字值,相应的判断由全局函数IsFinite、IsNaN来做(Number对象中有该属性)

* ==与===的区别:
	前者在作比较时或进行变量类型的转换，而后者判断相等时不做转换

* 数组初始化[]:
	[]可以用作对象初始化表达式-var matrix=[[1,2,3],[4,5,6],[7,8,9]];

* 对象初始化{}:
	例如；var user={"name":"hello",age:18}，基本形式类似于JSON的格式

##二、js中函数的一些特殊性质##

1、函数调用的问题:除了类似于其他语言的调用方式之外，js还提供了**call、apply方式**调用,需要注意的是就是各种方式this的引用

```

	function add(p1,p2){
       console.log("result="+(p1+p2)+"this="+this);
   }
   add(1,2);//this绑定到全局对象
   var math={method:add};
   math.method(3,4);//this绑定到math
   var t=new Date();
   add.apply(t,[1,2,3]);//第一个参数为this,第二个参数为参数数组，this绑定到Date
   var caller="I am caller";
   add.call(caller,1,2,3);//与apply功能类似，参数形式不同而已

```

2、函数的参数问题：js中函数调用的语法相当宽泛，我们传入的参数可以比函数声明的参数多，即使函数没有声明参数我们也可以传入参数，额外的参数可以在函数中使用**关键字augments**来获取

```

	function showAugments(first){
	    for(var i=0;i<arguments.length;i++){
	        console.log(arguments[i]);//打印所有参数
	    }
	}
	showAugments(1,2,"hello","params","more");

```

##三、js中的对象##

首先对象是属性的集合通过键值对属性进行定义基本行为同java有些类型，但是在js中对象是动态的-它的属性可以动态删除和增加。


* js对象中的属性
	
	1、增加对象：如果需要在定义后增加对象，直接引用一个属性赋值即可

	2、删除对象：使用delete关键字，只是断开属性与类对象之间的联系，而不操作属性，此外delete不能删除可配置性为false的属性

	3、检查对象是否有某个属性：**in**关键字、hasOwnProperty方法

	4、属性的特性：值(value)、可写性(writable)、可枚举性(enumerable)、可配置性(configurable，表示是否可以修改或者删除该属性)

	```

	var user={"name":"hello",age:18,relative:{x:1,y:1}};
    var show=function(Obj){
        for(var key in Obj){//枚举对象的属性，包括继承得到的
            console.log(key);
        }
        console.log("-----------------------");
    };
    show(user);
    user.job="software";//增加一个属性
    show(user);
    var rela=user.relative;
    delete user.relative;//删除一个属性
    console.log("x="+rela.x+"y="+rela.y);
    console.log(user.hasOwnProperty("job"));//判断一个属性是否存在
    console.log(Object.getOwnPropertyDescriptor(user,"name"));//获取name字段的属性
    Object.defineProperty(user,"name",{enumerable:false});//通过defineProperty函数设置属性的特性为不可枚举
    show(user);//可以预见的name属性不会打印出来

	```

* 对象的继承:
	
	js的对象通过原型链进行实现继承,每个对象都有一个原型对象(Object对象除外)，并且可以从原型中继承属性，这类似与java的类的结构所有的类继承自Object类，所以我认为原型是一种森林实现了继承机制。同时可以确定的是js不像java、c++一样对于类和实例有严格的区分，js的类和实例都是一样的。当我们创建一个对象时就指定了该对象的原型(**prototype**)的值，**每个对象的原型值即创建该对象的环境**

	* 创建对象的几种方式

		1、直接赋值:

		2、通过new进行创建,此时跟在new后的函数称之为构造函数:

		3、使用Object.create():

		```
		var phone={"brand":"魅族","number":121};//直接创建，
	    var user={"name":"js","sex":"男","age":18};
	    alert(phone.__proto__==Object.prototype);//验证原型指向
	    var list=new Array();//构造函数形式
	    alert(list.__proto__==Array.prototype);//验证原型指向
	    var obj=new Object();
	    var dat=new Date();
	    alert(dat.__proto__==Date.prototype);//验证原型指向
	    function Person(name,age){//定义函数
	        this.name=name;
	        this.age=age;
	        this.showInfo=function(){
	            alert("name="+this.name+"age="+this.age);
	        }
	    }
	    var person=new Person("Smith",18);//关键字new 使得Person成为一个构造函数
	    person.showInfo();
	    var o1=Object.create({x:1,a:2});//create方法创建

		```

	* 如何实现继承：继承的实现方法就是让对象的原型链存在正确的引用

	```

	function Shape(color){//定义构造函数
        this.color=color;
        this.showColor=function(){
            console.log("Color "+this.color);
        }
    }
    function Triangle(shape){
        this.profile=shape;
    }
    Triangle.prototype=new Shape("blue");//改变prototype
    Triangle.prototype.constructor=Triangle;//恢复构造属性
    var tri=new Triangle("triangle");
    tri.showColor();//测试是否得到Shape的属性
    function SkinTri(skin){
        this.skin=skin;
    }
    SkinTri.prototype=tri;
    SkinTri.prototype.constructor=SkinTri;//恢复构造属性
    var stri=new SkinTri("skin");
    for(var key in stri){//打印属性
        console.log(key);
    }

	```
	



