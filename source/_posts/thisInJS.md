---
title: javascript中函数的this指向以及apply/call/bind函数的联系与区别
date: 2016-07-28 10:36:09
tag: [javascript, this, apply, call, bind, nodejs]
---

### 联系  
&emsp; 三者的联系就在于，都可以用来改变函数中 this 指向的值，且第一个参数为要指向的 this 的值，apply的第二个参数（或 bind 与 call 的不定参数）为要传入的参数。这就不得不提及 javascript 中函数的 this 的指向了。this 的指向大概有以下几种。

1.全局作用域下或正常的函数调用的 this  
此时 this 指向的是全局对象。这时有两种情况，如果是在浏览器环境下运行，则 this 指向全局的 window 对象；而如果是在 nodejs 环境下执行，命令行中指向的是 global 对象。有一点需要注意的是，严格模式 "use strict" 下的 this 为 undefined 。

```javascript
// 浏览器
this.a = 10;
console.log(window.a); // 10
```

```javascript
// nodejs
this.a = 20;
console.log(global.a); // 20
```

```javascript
// 非严格模式
function f1(){
  return this;
}
console.log(f1() === window); // 浏览器 true
console.log(f1() === global); // nodejs true
```

```javascript
// 严格模式下
function f2() {
  "use strict";
  return this;
}
console.log(f2() === undefined); // true
```

2.当函数作为某对象的方法被调用时
 this 指向的是调用该函数的对象，以下代码指的是对象 a。

```javascript
var a = {
  b: 30,
  c: function () {
    console.log(this.b);
  }
}
a.c(); // 30
```

3.构造器函数调用
以构造器函数的形式声明函数，再用 new 关键字声明一个新的函数对象，此时函数中的 this 指向这个构造出来的对象。在 jslint 比较严格的要求下，这种构造函数的函数名必须是首字母大写的形式。

```javascript
function A() {
  this.b = 40;
}
var obj = new A();
console.log(obj.b); // 40
```

4.利用 apply/call/bind 方法强制改变 this 的指向
这时可以将要指向的对象 O 作为第一个参数传给以上任意三个函数之一，用某 F 函数调用这三个函数，这样 F 函数中的所有 this 的指向就都变成了对象 O，并且此时 O 就算没有声明 F 函数中的方法，也可以正常调用 F 函数中的方法。

```javascript
function a(arg) {
  console.log(this.b + arg);
}
var obj = {
  b : 1
};
a(); // NaN
a.apply(obj, [50]); // 51
a.call(obj, 60); // 61
var c = a.bind(obj, 70);
c(); // 71
```

### 区别
其实区别从上面的第四种也可见一斑。
1. apply 与 call： 第二个参数的形式 apply 为数组形式，函数会自动帮我们把数组展开，而 call 与 bind 为不定参数，需要传递几个就传递几个。
2. bind 与 （apply 和 call）：apply 与 call 都是在被调用的时候就执行函数体内容，而 bind 绑定之后返回绑定完成的函数，需要再显式执行一次此函数才能完成调用。
