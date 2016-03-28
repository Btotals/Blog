#JS学习随手记2 - What's this
---
JS作为一门脚本语言，曾经一度被人称为*“玩具语言（Toy Language）”*，很大一部分原因就是因为不了解它的人，总是以为JS有多么简单易上手。无可否认的是，如果有着一定的C/C++程序设计基础，会发现的确如此，因为它提供了更多强有力的方法。

但是，JS里面像闭包（Closure）、基于原型链（Prototype）的继承以及像函数式编程（Functional Programming）这样的高级特性，却并不是那么容易搞清楚的。接下来就说一下这几天来关于this的一些学习所得。文章有些长，但我相信看完并理解它之后，能在之后节省下来的到处Google的时间，将远远多于我码出这篇文章的时间。

## C++中的this
在C++这样的OO语言中，this是经常被使用的关键字。因为它能起到一个命名空间的作用，比如下面的代码：

```
class Point {
public:
  Point(x, y) {
    this.x = x;
    this.y = y
  }
private:
  int x, y;
};
```
这样的代码，this含义很明确，就是当前对象。

## JavaScript中的this
但是，在JavaScript中，由于它运行时绑定的特性，this的指代会产生多种多样的情况。如下表：

| 调用方式 | 示例 | this |
| ------------ | ------------- | ------------ |
| 普通函数调用 | var a = foo()  | 全局对象 |
| 构造函数调用 | var foo = new Foo()  | 新构造的对象foo |
| 对象方法调用 | foo.bar()  | 调用bar的对象foo |
| call/apply | foo.bar.call(foo2)| call/apply的参数foo2 |

大概说明一下每一种情况：

### 1. 普通函数调用
这时，函数内部的this被绑定到了全局对象：

```
function foo(x) { this.x = x; }
console.log(x);
// window (if you run this in your browser)
// global (if you run this in your node REPL)
```
### 2. 构造函数调用
这时，this被绑定到新构造的对象foo上。你可能会发现这里的`Foo`和上面的`foo`看似一模一样。的确是的，在不使用`new`关键字的情况下，构造函数中的this就会绑定到全局对象上。因此，作为约定俗成的规范，构造函数（比如Array、Object、RegExp等等）都使用大写字母开头，提醒调用者需要使用`new`关键字正确的调用。

```
function Foo(x) { this.x = x; }
var foo = new Foo(5);
console.log(foo.x);
// 5
```

### 3. 对象方法调用
这种情况和C++、Java等语言中this的表现类似，它会绑定到调用该方法的对象上。

```
function Foo(x) {
  this.x = x;
  this.setX = function(x) { this.x = x; }
}
var foo = new Foo(5);  // 5
console.log(foo.x);
foo.setX(10);
console.log(foo.x);  // 10
```
但是，如果不正确地使用this，将会带来一个问题：this丢失。后文将详细说明。

### 4. call/apply调用
在JS中函数也是对象，call和apply就是函数对象的方法。这两个方法在JS中极其强大，因为它们能够自由切换函数执行的上下文（Context），这里极大程度的体现了JS这门语言的动态性所在。关于两个函数的不同这里就不啰嗦了，大家可以自行Google之。

```
function Foo(x) {
  this.x = x;
  this.setX = function(x) { this.x = x; }
}
var foo = new Foo(5);  // 5
var hehe = { x: 100 };
foo.setX.call(hehe, 10);
console.log(hehe.x);  // it is 10, not 100!
```

## this使用常见误区
上面提到过，在对象方法调用中，不正确的使用this会导致this丢失的问题，这里将结合代码简要说明一下：

```
var point = {
  x : 0,
  y : 0,
  moveTo : function(x, y) {
    // console.log(this);  // What's this?
    var moveX = function(x) {
      // console.log(this);  // What's this?
      this.x = x;
    };
    var moveY = function(y) {
      this.y = y;
    };
  moveX(x);
  moveY(y);
  }
};
point.moveTo(1, 1);
console.log(point.x, point.y);  // 0 0
console.log(x, y);  // 1 1
```
能看出为什么这样子么？

刚刚有说过，在对象方法调用的时候（point.moveTo），毋庸置疑，`moveTo`函数内部的`this`指向的是`point`对象。但是，在`moveTo`函数内部调用`moveX`和`moveY`的时候，却是普通的函数调用，也就是说它们的`this`都指向全局对象。

在JS中为了修复这样的Bug，会使用一个名为`that模式`的方法。具体如下：

```
var point = {
  x : 0,
  y : 0,
  moveTo : function(x, y) {
    var that = this;
    var moveX = function(x) {
      that.x = x;
    };
    var moveY = function(y) {
      that.y = y;
    };
  moveX(x);
  moveY(y);
  }
};
point.moveTo(1, 1);
console.log(point.x, point.y);  // 1 1
console.log(x, y);  // undefined undefined
```
对于新手而言，同样的问题还有可能会出现在`setTimeout/setTimeInterval`中。比如

```
var point = {
  x : 0,
  y : 0,
  printAfterOneSecond = function() {
    var that = this;
    setTimeout(function() {
      console.log(this.x, this.y);  // undefined undefined
      console.log(that.x, that.y);  // 0 0
    }, 1000);
  }
};
```
至于为什么增加一个that就能解决问题，同样会在下一节讲到。

## this如何被确定？
先提醒一下，接下来的内容比较枯燥，但却是理解this如何确定的重点所在……

### 要先从作用域讲起……
由之前的调用方式表格可知，JS中函数有着众多的调用方式，这就是this含义丰富的原因。要了解this如何确定，就要先了解JS中关于上下文作用域的一些知识。

JS中函数执行时，会先创造一个`执行环境（Execution Context）`，或者简称`环境`，它定义了变量或函数有权访问的其他数据。每个执行环节都会有一个与之相对的`变量对象（variable object）`，环境中定义以及能访问到的所有变量和函数都保存在其中。当某个执行环境中的所有代码执行完毕猴，该环境会被销毁，而这些变量和函数等等也会随之销毁。

每一个函数都有自己的执行环境，当控制流进入某一个函数时，该函数的环境会被压入一个`环境栈`中；在它执行完毕后，栈就会弹出这个环境并将控制权返回给之前的执行环境。

当代码在某个环境中执行时，会创建关于`变量对象`的一个`作用域链`。它的用途就是保证这段代码变量的访问顺序。作用域链的前端，永远是当前正在执行代码的环境对象。而它的下一个环境变量，就是来自高一级的环境，以此类推，直到最外层，也就是作用域链的最后一环：`全局环境`。

而我们通常说的`全局环境`，在Web浏览器里面被认为是`window`，而在node的REPL环境里面则是`global`。所有的全局变量以及函数都是作为这个全局对象的属性和方法被创建的。

举个作用域的栗子：

```
var x = 1;
function foo() {
  var y = 2;
  function swap() {
    // 这里可以访问到x, y, temp
    var temp = y;
    x = y;
    y = temp;
  }
  // 这里可以访问到x, y,但不能访问到temp
}
这里只能访问到x
```
它们的作用域链如下图所示：

```
  window
  | - x
  | - foo()
    | - y
    | - swap()
      | - temp
```
这里表现了上图代码中的作用域链。其中，内部环境可以访问到外部环境，但外部环境却不能访问到内部环境。他们之间顺序遵循上面说到的执行顺序。比如在`swap`函数内，它的环境中没有x和y，但是它可以先向上一级的`foo`环境中寻找，但是只有x没有y，于是它必须再向上一级的`window`环境中寻找。如果当它搜索到最外层仍然没找到这个变量，这时就会产生一个错误。

顺带一提，函数的参数也被同样当做变量对待。它所在的环境就是当前函数的环境。

### this的本质
想必在上文balabala说了一大通`执行环境`以及`变量对象`之后大家也应该猜到所谓的`this`，就是函数`执行环境`中对`变量对象`的一个引用。对`this`的求值，在执行的时候被转换成对当前`变量对象`的求值，回忆一下上面`that模式`的代码：

```
var point = {
  x : 0,
  y : 0,
  moveTo : function(x, y) {
    this.x;
    var moveX = function(x) {
      this.x;
    };
    ......
    moveX(x);
  }
};
```
在`moveTo`内的this.x由于它的`this`引用的是`point`，故`this.x`的求值会被自动转换成`point.x`。相应的`moveX`里面则会被转换成`window.x`。

## 附录：JS中call/apply的高级应用
之前提到过call/apply能极大体现JS的动态性，这里就大概举两个栗子。希望能起到抛砖引玉的作用。更多的神奇用法可以Google之。
### 0. bind方法
和call/apply一样，bind也是函数对象的方法之一。它的作用就是创建一个新函数，bind的第一个参数会被作为新函数的this，而从第二个开始的所有参数连同运行时输入的参数会按顺序作为参数调用原函数。

因此，在JS中，结合bind和call/apply，以及函数式编程（Functional Programming）的思想，就能写出很多优雅实用的代码。

### 1. Math.max用于数组
JS中的Math.max仅支持从若干个数字取出最大值，如果需要从某数组中取出一个最大值的话，它就无能为力了。在之前，你可能只会用循环的方式逐个比较，但现在，我们有`apply`，可以更加优雅简洁的实现这个功能，代码如下：

```
var maxOfArray = Function.prototype.apply.bind(Math.max, null);
maxOfArray([1,2,3,4,5,6]);  // 6
// 相当于这样调用：
Math.max.apply(null, [1,2,3,4,5,6]);
// 又相当于这样调用：
Math.max(1,2,3,4,5,6);
```
大概解释：在对apply调用bind方法后，相当于用`Math.max.apply(null, [1,2,3,4,5,6])`这样的方式调用apply（回想一下bind的作用），对于`Math.max`方法来说，它不关心执行时的this环境，所以apply的第一个参数可以为任意。而第二个参数在传入apply之后，会在JS底层实现被分解成一个个单独的参数输入到max中。因此我们才能这样优雅的实现max功能。

### 2. forEach用于类数组
在Web浏览器环境中，document.getElementByXXXX这样的方法返回的都是一个类数组（意为可用数字下标访问，但Array.isArray方法会返回false）。常见的情景是我们需要对这一系列的dom添加某些事件监听器，虽然ES5标准中新增加了数组的迭代方法forEach、map等等，但是类数组却没有这样的方法。于是在此之前，可能都是只能用循环实现，但现在，我们有`call`（这话怎么听起来有点熟orz）……

```
var each = Function.prototype.call.bind([].forEach);
each(document.getElementsByClassName('button'), function(item) { item.onclick = xxx; });
// 相当于这样调用：
[].forEach.call(domSet, fun);
```
原理和上面说的相似，由于domSet这个集合与数组类似都可以用数字下标访问，而forEach内部也用到了下标实现遍历。因此就能这样优雅的实现。

注：这里使用到了Python中[鸭子类型](https://en.wikipedia.org/wiki/Duck_typing)的思想。因为它们的表现相似，仅仅是缺少数组相关的方法而已。

## 后记
码了好几个小时终于码完了这一篇东西。东扯西扯的说了最近一些学习this的收获，虽然这仅仅是JS概念中很小的一部分，但是我们却可以借助它了解JS中作用域的特性，从而为学习闭包、高阶函数等高级特性打好基础。也只有深入学习这些地方，才能真正了解JS中强大的一面，而不是简单地认为它是一门玩具语言。

## 参考资料
1. [Understanding JavaScript Function Invocation and "this"](http://yehudakatz.com/2011/08/11/understanding-javascript-function-invocation-and-this/?cm_mc_uid=46590394127414517438545&cm_mc_sid_50200000=1457875773)
2. [var statement in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var)
3. [深入浅出 JavaScript 中的 this](http://www.ibm.com/developerworks/cn/web/1207_wangqf_jsthis/index.html)
4. JavaScript高级程序设计 第三版(Professional JavaScript for Web Developers, 3rd)
    * 4.2章 执行环境及作用域
    * 5.5章 Function类型