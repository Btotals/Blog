# JS学习随手记3 - Closure
---
## 0. 写在前面：
关于JS里面的闭包，我感觉刚开始接触到的时候总是云里雾里摸不着头脑，网上找的很多资料都语焉不详。正好最近在重新看《JS高级程序设计》一书，干脆就自己来写一个总结2333，也方便之后的复习。

#正文
在对JS里面的this、环境变量和作用域链有了一定了解之后，下一步就应该开始接触下一步应该掌握进阶内容：闭包（Closure）。

说闭包之前，来重新看一下关于作用域的一些东西……

## 1. JavaScript中的作用域划分
首先，在JS中，并不像C/C++等语言一样，作用域是块级的（也就是按照花括号划分的作用域）。只有**函数**才具有自身作用域，也就是说，JS不存在块级作用域。

因此，JS中函数的变量定义将会出现在整个函数中。比如下面的代码：

```
function foo() {
  console.log(hello);  // undefined
  for (var i = 0; i < 10; i++) {
    var hello = "Hello world!";
    console.log(hello);
  }
  console.log(i);  // 10
  console.log(hello);  // "Hello world!"
}
```
这段代码里面有几个地方需要说一下，第一个console.log会输出`undefined`而不是`Uncaught ReferenceError`，是因为JS会在编译（准确来说，应该是解释JS语句之前），做一个叫`变量提升`的工作。也就是说，这段代码会先变成这个样子：

```
function foo() {
  var hello, i;
  ......
}
```
所以，在第一个`console.log`里面输出的`hello`就是`undefined`，因为这个变量已被声明但未赋值。

然后，是后面的这两个`console.log`，这里就很好地解释了JS是函数作用域而不是块级作用域：在`for`循环块外的代码仍可以访问到`for`里面的变量。

这一部分总结来说，就是JS中**只存在函数作用域**，而且函数中的变量定义在**整个函数**中都有效。

## 2. 什么是闭包
### i. 闭包的定义
理解了作用域的划分规则之后，让我们来看几个关于闭包的定义……

* Closures are functions that refer to independent (free) variables. In other words, the function defined in the closure 'remembers' the environment in which it was created. - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
* 翻译：闭包是那些能够访问独立（自由）变量的函数，换句话说，这些定义在闭包内的函数可以“记住”它们被创建时候的环境。（注：我个人认为这个独立和自由应该作“外部”的意思来理解）
* 闭包是指能访问另一个函数作用域中变量的函数 - 《JS高级程序设计 第三版》
* 内部函数可以访问定义它们的外部函数的参数和变量（`this`和`argument`这两个特殊参数除外，原因可以参见我之前的那篇[What's this]()） - 《JS语言精粹》

### ii. 闭包的本质
用简单的话总结一下，闭包具有这两个特点：

i. 闭包是能访问外部函数作用域中的变量的函数
i. 在闭包中访问到的外部函数的变量可以**保留**在外部函数中而不会被垃圾回收器回收。这点非常重要，因为对闭包的不慎使用会导致**内存泄露**。

### iii. 闭包中的作用域链
先回想一下关于JS作用域链的知识：JS中函数被调用时都会创建一个属于它自身的`执行环境（Context）`以及一个`变量对象（Variable Object）`，前者被压入`环境栈`中创建`作用域链`，后者则记录函数中所创建的变量以及传入的参数和这些参数的引用`argument`。

有了这些知识，现在先创建一个简单的闭包：

```
function sayHello(name) {
    return function() {
        console.log(name, "says hello!");
    }
}
var say = sayHello('Btotals');
```
它的作用域链如下（假设运行环境是**浏览器**，全局对象为`window`）：

```
window
| - say
| - sayHello
  | - name（传入的参数）
  | - anonymous function（闭包）
    | - name
```
在这里面，当这个匿名函数（也就是闭包）在`sayHello`中被创建的时候，此时它的作用域链已经将`sayHello`的变量对象和`window`都包含在内了，因此，它就能访问到`sayHello`里面的参数和变量。

另外，还有一点也同样很重要：即使`sayHello`函数已经执行完，但由于存在`say`方法引用了这个被return的匿名函数，因此这个匿名函数不会被回收；与此同时，这个匿名函数内部也存在对sayName中变量对象的引用。因此，尽管在`sayHello`执行完后这一系列执行环境的作用域链被销毁了，但它的变量对象会继续留在内存中，直到这个匿名函数也被销毁；换言之，如果这个匿名函数一直不被销毁，就会造成**内存泄露**。

## 3. 闭包的实例
### i. 循环中使用闭包
想必很多初学者都会遇到过这样一个错误，就是循环中对计数器的使用：

```
for(var i = 10; i >= 0; i--) {
    setTimeout(function() {
        console.log(i);
    }, 1000);
}
```
假如我们想实现一个从10s到0s的倒计时功能，可能初学者就会一拍脑袋写出这样的代码，然而实际运行的时候发现它只会每隔一秒打印出一个0，这是为什么？

因为当`console.log`被调用的时候，循环已经结束，i的值已经为0。那么，如果想得到正确结果，就必须使用闭包来创建i的副本。

```
// method 1
for (var i = 10; i >= 0; i--) {
  (function(e) {
    setTimeout(function() {
      console.log(e);
    }, (10-e) * 1000);
  })(i);
}

// method 2
for (var i = 10; i >= 0; i--) {
  setTimeout((function(e) {
    return function() {
      console.log(e);
    };
  })(i), (10-i) * 1000);
}
```

方法1和2都使用了`IIFE（immediately-invoked function expression）`，或称`自执行函数表达式`。其中方法1中匿名函数会立即执行，并把i作为参数，此时函数内e就是i的一个拷贝。而这个值是不会再受外部循环中i影响，从而达到想要的效果。

方法2就是从IIFE中返回一个函数作为参数提供给`setTimeout`执行。这和上面的代码效果一样，可以尝试自行理解。


### ii. 实现访问控制
在实际的应用里面，经常可能需要用JS模拟面向对象语言中的某些特性，比如对象的`public/private`属性。虽然JS中没有这样的机制，但却可以通过使用闭包来模拟。

```
var Person = function() {
    this.name = "default";
    return {
       getName : function() {
           return name;
       },
       setName : function(newName) {
           name = newName;
       }
    }
};
var p = new Person();
console.log(person.name);  // 直接访问，结果为undefined
console.log(person.getName());  // default
person.setName("Btotals");
console.log(person.getName());  // Btotals
```

## 4. 内存管理
在一些像C/C++这样的语言里面，需要手动使用`malloc/new/free/delete`这样的方法做内存分配与回收。但是，在JS中，拥有垃圾回收机制。但这样的机制很容易给初学者留下**不需要关心内存管理**这样的不良印象。

### i. 内存生命周期
在大多数的编程语言里，都有着类似这样的内存生命周期：

1. 根据代码声明分配所需要的内存
2. 使用这部分内存（读/写）
3. 在这部分内存不再需要的时候回收它

在所有语言中，前两部分一般都是显式的声明；而像C/C++等语言的第三部分也是显式的，但JS里面是隐式的。

但是，要弄清楚的一点是，隐式的内存回收，并不代表我们不需要关注内存回收。特别是在弄明白了“闭包”这个概念之后，更是要警惕，比如以前面的`sayHello`为例子：

```
function sayHello(name) {
    return function() {
        console.log(name, "says hello!");
    }
}
var say = sayHello('Btotals');
```
在这里，`say`是对`sayHello`返回的匿名函数的引用，而这个匿名函数内又保存着`sayHello`内`变量对象`的引用，还记得之前所说么？这个变量对象会一直留存在内存里，直到这个匿名函数被销毁。

显而易见的是，不正确的使用闭包，会导致严重的内存泄露，从而导致页面的内存占用越来越高，反映在用户体验上可能就是页面越来越“卡”，甚至会引发页面崩溃等后果。

### ii. 垃圾回收算法
关于如何判定某段内存“不再需要”这一点，一般存在以下两种方法：`引用计数算法（Reference-counting）`和`标记-清扫算法（Mark-and-sweep）`。当然，两种算法有各自的优缺点，但现代浏览器的JS引擎都使用`标记-清扫算法`。详细说明可以浏览参考资料 - [Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)。

## 5. 总结
* 概念：闭包就是能访问外部函数作用域中变量和参数的函数。
* 优点：闭包可以使它外部函数作用域中的变量常驻内存而不会被回收。
* 优点：可以用于变量的封装，避免污染全局环境，或者用于私有化某些变量。
* 缺点：但闭包会保存对外部函数变量对象的引用，因此会比普通函数占用更多的内存。
* 缺点：如果忽略了内存管理，可能会造成内存泄露，从而影响页面性能。


## 6. 参考资料

1. [Closure on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) - MDN上关于闭包的解释。
2. [Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management) - 对内存生命周期及两种垃圾回收算法的说明。
