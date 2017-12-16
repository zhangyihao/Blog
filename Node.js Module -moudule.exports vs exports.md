---
title: Node.js Module -moudule.exports vs exports
date: 2017-02-15 21:04:17
tags:
	- Node.js
	- module
	- exports
categories:
  - Node.js
comments: false

---

# 概览 #

先来看下官网对module.exports和exports的描述：
>The module.exports object is created by the Module system. Sometimes this is not acceptable; many want their module to be an instance of some class. To do this, assign the desired export object to module.exports. Note that assigning the desired object to exports will simply rebind the local exports variable, which is probably not what you want to do.
>
>The exports variable is available within a module's file-level scope, and is assigned the value of module.exports before the module is evaluated.
>
>It allows a shortcut, so that module.exports.f = ... can be written more succinctly as exports.f = .... However, be aware that like any variable, if a new value is assigned to exports, it is no longer bound to module.exports

<!--more-->
从上边我们可以得到以下几点信息：

  1. module.exports是由Module系统自动创建。
  2. **exports是module.exports的引用。**
  2. exports的作用域仅限所在文件内。
  2. export收集到的属性、方法会赋值给module.exports
  3.  **通过require得到的是module.exports中的内容，而不是exports的内容。** 

以上可以看出，exports收集的属性、方法最后会赋值给module.exports，但是需要注意的是，如果module.exports已经有属性或方法，那么exports收集的属性或方法将被忽略，不会被module.exports收集到。

# exports vs module.exports #
首先，我们先创建两个js文件：a.js和b.js，代码分别为：
**a.js文件内容:**

```javascript
function sayHello() {
	console.log('hello!');
}

/*输出相关信息*/
var eq = exports === module.exports;
console.log('相等？'+eq);
console.log(exports);
console.log(module.exports);
```
**b.js文件内容:**

```javascript
/*引用文件a*/
var a = require('./a.js');
```
在上述代码中，文件a中定义了sayHello方法，并在文件b中引入了文件a，下面通过示例，来了解下两者间的差别。

## module.exports和exports对外公开方法、属性都可以访问，但有区别 ##

### 示例1： ###

在a.js文件中添加代码，给exports赋值。

```javascript
function sayHello() {
	console.log('hello!');
}

/*把方法赋值给exports，moudule.exports也会有这方法*/
exports.sayHello = sayHello;

/*输出相关信息*/
var eq = exports === module.exports;
console.log('相等？'+eq);
console.log(exports);
console.log(module.exports);
```
运行b.js，输出结果如下：

```
E:\Projects\Test>node b.js
相等？true
{ sayHello: [Function: sayHello] }
{ sayHello: [Function: sayHello] }
```
从输出结果上可以看到，exports会把属性、方法传递给module.exports，二者没有区别。

### 示例2： ###

在a.js文件中添加代码，给module.exports的属性赋值。

```javascript
function sayHello() {
	console.log('hello!');
}

module.exports.exports = sayHello;

/*输出相关信息*/
var eq = exports === module.exports;
console.log('相等？'+eq);
console.log(exports);
console.log(module.exports);
```
运行b.js，输出结果如下：

```
E:\Projects\Test>node b.js
相等？true
{ sayHello: [Function: sayHello] }
{ sayHello: [Function: sayHello] }
```
从输出结果上可以看到，module.exports和exports指向相同，二者没有不同。

### 示例3： ###
在a.js文件中添加代码，给exports赋值。

```javascript
function sayHello() {
	console.log('hello!');
}

/*把一个方法赋值给exports*/
exports = sayHello;

/*输出相关信息*/
var eq = exports === module.exports;
console.log('相等？'+eq);
console.log(exports);
console.log(module.exports);
```
运行b.js，输出结果如下：

```
E:\Projects\Test>node b.js
相等？false
[Function: sayHello]
{}
```
从输出结果上可以看到，exports没有把属性、方法传递给module.exports，最终module.exports和exports指向不同，**说明二者是有区别的**。
#### 原因分析： ####
在一开始，exports为文件内部的一个变量，和module.exports指向相同，都指向一个空对象{}。而在文件最后，将一个函数赋值给exports时，改变了exports的指向，此时exports指向了该函数，而没有修改module.exports的指向，导致最后两者不再指向相同内容。如果想让两者指向相同，应这样写：module.exports = exports = function say(){...}

下面代码模拟一个文件被require时，node框架处理modul过程（摘自官网）：

```javascript
function require(...) {
  var module = { exports: {} };
  ((module, exports) => {
    // Your module code here. In this example, define a function.
    function some_func() {};

    exports = some_func;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.

    module.exports = some_func;
    // At this point, the module will now export some_func, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
```

### 示例4： ###
在a.js文件中添加代码，给module.exports赋值。

```javascript
function sayHello() {
	console.log('hello!');
}

module.exports = {
	say : 'say'
};
exports.say = 'say exports';

/*输出相关信息*/
var eq = exports === module.exports;
console.log('相等？'+eq);
console.log(exports);
console.log(module.exports);
```
运行b.js，输出结果如下：

```
E:\Projects\Test>node b.js
相等？false
{ say: 'say exports' }
{ say: 'say' }
```
从输出结果上可以看到，**module.exports和exports指向不相同**。原因为module.exports改变了指向，二者指向不再相同，即module.exports已经有属性或方法，那么exports收集的属性或方法将被忽略，不会被module.exports收集到。


# 总结 #
1. module.exports 初始值为一个空对象 {}，所以 exports 初始值也是 {}
2. exports仅仅是module.exports的一个地址引用。
3. nodejs只会导出module.exports的指向，如果exports指向变了，那就仅仅是exports不在指向module.exports，于是不会再被导出。

module.exports和exports之间无论如何赋值，只要分析清楚二者间引用关系，即可知道最后Module导出的对象是谁。
**最后，推荐在写代码时，只使用module.exports导出相关属性、方法或对象，避免和exports的混淆。**