#JS学习随手记1 - null & undefined

***
*2015-12-24 11:26:33*

今天遇到了像

`Array.apply(null, Array(3)).map((item, index) => index)`

这样的代码，其中map以及里面的[Arrow Function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)其实都可以理解，但是像Array.apply这样的用法却真的是闻所未闻。

很自然的第一想法便是比较上面代码和

`Array(3).map((item, index) => index)`

的不同，发现前者输出`[0, 1, 2]`而后者输出为`[undefined*3]`

分解一下这段代码，`Array(3)`其实就是调用Array的构造函数并返回一个长度为3的数组对象，log输出该数组为`[undefined * 3]`，但是`Array.apply(null, Array(3))`的输出值却为`[undefined, undefined, undefined]`，由此可见，两种方法得到的数组，其实是有着差别的。

仔细阅读了关于[Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)构造函数的代码之后，发现似乎直接以一个数字作为参数调用`Array`构造函数的时候返回的数组是一个[Sparse Array](https://en.wikipedia.org/wiki/Sparse_array)

在使用`hasOwnProperty`方法查看两个数组有没有`1`这个属性之后发现，`Array(3)`返回的数组是没有的，也就是说他仅仅像MDN上文档说的那样只初始化了`length`属性，所以在使用`map`等方法遍历的时候，因为里面没有合法的元素，所以遍历结果为空。

---
*Update at 2016-01-13 10:48:50*

既然前面说到了空槽（empty slot）和undefined，这里就补充一下关于null和undefined的区别：

在JS里面，`null`和`undefined`表示的意思非常接近，都是表示没有值，而且`null == undefined`的结果也是true。所以，区别在哪？

原来，JavaScript的设计者*Brendan Eich*最早设计的时候，只像C/C++那样添加了null用于表示0，但是JavaScript的最初版本没有`try - catch`错误处理机制，发生数据类型不匹配时，往往是自动转换类型或者默默地失败。*Brendan Eich*觉得，如果null自动转为0，很不容易发现错误。因此，他又设计了一个undefined。

但是，在目前的使用中，却没有起到他预想的效果。现在基本上可以认为null与undefined是差不多的。但是借我个人看来，还是有一些细小的差别：

1. null是表示空值（不是空对象，虽然类型为object），所以用来表示**此处不应该有值**，比如对象原型链条的终点`Object.getPrototypeOf(Object.prototype)`，或者不被推荐使用的`Object.prototype.__proto__`，它们的值都是null。
2. undefined则表示“缺少值”。举例说明：
	- 变量声明了，但未赋值
	- 调用函数的时候，没有传递的参数
	- 对象没有被赋值的属性
	- 没有返回值的函数

	```
	var a;
	console.log(a);  // undefined

	function foo(v) {console.log(v);}
	foo();  // undefined

	var obj = {};
	obj.p;  // undefined

	var x = foo();
	console.log(x);  // undefined
	```

所以，终上所述，我个人感觉，null和undefined区别还是不少的，就比如说将一个变量赋值为null，意思就是说这个变量的值已经被定义了，是“空值”；但是对一个变量赋值为undefined，应该是不合理的。

而且，在实际应用中，很容易发现，像JQuery这些使用广泛的库，里面提供的深拷贝（Deep Copy）方法，都会把对象值为null得字段属性保留，而忽略undefined。所以，这也从侧面证明了这一理解的正确。

另外，在浏览器环境下，如果一个dom元素不支持某些事件，那么它的'onxxx'属性是undefined；但是如果这是它支持的事件，却没有定义事件处理函数，则会是null。

最后，个人认为`typeof(null) === 'object'`应该属于设计的失误，在harmony中有人[提议](http://wiki.ecmascript.org/doku.php?id=harmony:typeof_null)应该修正这一失误，但是由于它会导致无数现有站点的JS脚本运行出错，因此被reject了。

