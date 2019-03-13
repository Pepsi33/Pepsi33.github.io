---
title: 模块
date: 2017-11-07 11:30:05
tags: ["javascript","YDKJS"]
categories: 学习
---

## 现代的模块

各种模块依赖加载器/消息机制实质上都是将这种模块定义包装进一个友好的API。

``` javascript
	
	var MyModules = (function Manager() {
		var modules = {};

		function define(name, deps, impl) {
			for (var i=0; i<deps.length; i++) {
				deps[i] = modules[deps[i]];
			}
			modules[name] = impl.apply( impl, deps );
		}

		function get(name) {
			return modules[name];
		}

		return {
			define: define,
			get: get
		};
	})();

```

> 这段代码的关键部分是 modules[name] = impl.apply(impl, deps)。这为一个模块调用了它的定义的包装函数（传入所有依赖），并将返回值，也就是模块的API，存储到一个用名称追踪的内部模块列表中。

<!--more-->

``` javascript
MyModules.define( "bar", [], function(){
	function hello(who) {
		return "Let me introduce: " + who;
	}

	return {
		hello: hello
	};
} );

MyModules.define( "foo", ["bar"], function(bar){
	var hungry = "hippo";

	function awesome() {
		console.log( bar.hello( hungry ).toUpperCase() );
	}

	return {
		awesome: awesome
	};
} );

var bar = MyModules.get( "bar" );
var foo = MyModules.get( "foo" );

console.log(
	bar.hello( "hippo" )
); // Let me introduce: hippo

foo.awesome(); // LET ME INTRODUCE: HIPPO
```

## 未来的模块

ES6 为模块的概念增加了头等的语法支持。当通过模块系统加载时，ES6 将一个文件视为一个独立的模块。每个模块可以导入其他的模块或者特定的API成员，也可以导出它们自己的公有API成员。

**注意：** 基于函数的模块不是一个可以被静态识别的模式（编译器可以知道的东西），所以它们的API语义直到运行时才会被考虑。也就是，你实际上可以在运行时期间修改模块的API。

相比之下，ES6 模块API是静态的（这些API不会在运行时改变）。因为编译器知道它，它可以（也确实在这么作！）在（文件加载和）编译期间检查一个指向被导入模块的成员的引用是否 实际存在。如果API引用不存在，编译器就会在编译时抛出一个“早期”错误，而不是等待传统的动态运行时解决方案（和错误，如果有的话）。

ES6 模块 没有 “内联”格式，它们必须被定义在一个分离的文件中（每个模块一个）。浏览器/引擎拥有一个默认的“模块加载器”（它是可以被覆盖的，但是这超出我们在此讨论的范围），它在模块被导入时同步地加载模块文件。

**bar.js**
``` javascript
function hello(who) {
	return "Let me introduce: " + who;
}

export hello;
```


**foo.js**
```javascript
// 仅导入“bar”模块中的`hello()`
import hello from "bar";

var hungry = "hippo";

function awesome() {
	console.log(
		hello( hungry ).toUpperCase()
	);
}

export awesome;
// 导入`foo`和`bar`整个模块
module foo from "foo";
module bar from "bar";

console.log(
	bar.hello( "rhino" )
); // Let me introduce: rhino

foo.awesome(); // LET ME INTRODUCE: HIPPO
```
**注意：** 需要使用前两个代码片段中的内容分别创建两个分离的文件 “foo.js” 和 “bar.js”。然后，你的程序将加载/导入这些模块来使用它们，就像第三个片段那样。

import 在当前的作用域中导入一个模块的API的一个或多个成员，每个都绑定到一个变量（这个例子中是 hello）。module 将整个模块的API导入到一个被绑定的变量（这个例子中是 foo，bar）。export 为当前模块的公有API导出一个标识符（变量，函数）。在一个模块的定义中，这些操作符可以根据需要使用任意多次。

