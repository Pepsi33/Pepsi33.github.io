---
title: new Object发生了什么
date: 2019-03-11 23:53:36
tags: ["javascript"]
---

### new XXX()发生了什么？

1. 创建一个新对象
2. 将构造函数的作用域赋给新对象（因此this就指向了这个新对象）
3. 执行构造函数中的代码（为这个新对象添加属性）
4. 返回新对象
   
```
function creatObj() {
    var obj = new Object(),
        Constructor = [].shift.call(arguments);
    obj.__proto__ = Constructor.prototype;
    var ret = Constructor.apply(obj, arguments);
    return typeof ret === 'Object' ? ret : obj;
}

function Person(name){
    this.name = name;
}

Person.prototype.getName = function(){
    return this.name;
}

var a = creatObj(Person, 'seven');
console.log(a.name)   // seven
console.log(a.getName)   // seven
console.log(Object.getPrototypeOf(a) === Person.prototype)   // true

下面两句代码产生了一样的结果
var a = creatObj(Person, 'seven')
var a = new Person('seven')

// 另一种写法
function creatObj() {
    var Constructor = [].shift.call(arguments),
        obj = Object.create(Constructor.prototype);
    var ret = Constructor.apply(obj, arguments);
    return typeof ret === 'Object' ? ret : obj;
}

```