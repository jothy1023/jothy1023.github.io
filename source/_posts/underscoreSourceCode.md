---
title: underscore源码解析之类型判断函数
date: 2016-07-13 18:06:56
tag: [underscore, 源码]
---
&emsp;首先将内置对象的原型链以及内置对象原型中的常用方法缓存在局部变量中

```javascript
// 内置对象原型链
var ArrayProto = Array.prototype,
    ObjProto = Object.prototype,
    FuncProto = Function.prototype;

// 内置对象原型中的常用方法 e.g: toString
var toString = ObjProto.toString,
    hasOwnProperty = ObjProto.hasOwnProperty;
```

&emsp;这样做的好处除了简洁代码之外，还有两个好处，首先是利于代码的压缩。而原生的对象原型无法进行压缩，e.g: Object.Protype压缩之后宿主就不认识了，但是objProto可以压缩为a，之后的调用也可以正常进行；然后是可减少在原型链中的查找次数(提高代码效率)。

------

&emsp;再定义一组javascript原生支持的判断函数，若宿主环境（浏览器/nodejs）支持，则直接返回。

```javascript
var nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind,
    nativeCreate       = Object.create;
```

&emsp;若不支持以上函数（es5之后才支持），则返回underscore自己写的判断函数。

------

- 是否为DOM

```javascript
_.isElement = function(obj) {
  // 首先确保不是 null/undefined 等假值（!!obj），
  // DOM 的 nodeType 为1，!!用于强制类型转换为 Boolean 值
  return !!(obj && obj.nodeType === 1);
};
```

- 是否为数组Array

```javascript
_.isArray = nativeIsArray || function(obj) {
  // call 方法可以使得任意 obj 都可以调用 toString 方法，即使是没有 toString 方法的对象
  return toString.call(obj) === '[object aray]'
};
```

- 是否为对象Object

```javascript
_.isObject = function(obj) {
  var type = typeof(obj);
  // 除了普通对象之外，函数也是对象
  // !!obj 是为了排除 null 的情况，因为 typeof null 也为 object
  return type === 'function' || type === 'object' && !!obj;
};
```


- 是否为布尔值Boolean

```javascript
_.isBoolean = function(obj) {
  // 除了 true 与 false 之外，布尔值还可能是 new Boolean() 哦
  // 不过似乎直接用最后一种判断就可以了？
  return obj === 'true' || obj === 'false' || toString.call(obj) === '[object boolean]';
};
```


- 是否为arguments

```javascript
if (!_.isArguments(arguments)) {
  _.isArguments = function(obj) {
    // 通过是否有 callee 方法判断
    // 因为IE < 9 下对 arguments 调用 Object.prototype.toString.call 方法
    // 返回的是 [object Object] ，而非 [object Arguments]
    return _.has(obj, 'callee');
  };
}
```


- 是否为NaN

```javascript
_.isNaN = function(obj) {
  // NaN是一个Number类型，但是它不等于它本身
  // ‘+’ 放在变量前面一般作用是把后面的变量变成一个数，
  // 在这里已经判断为一个数仍加上 ‘+’，是为了把 var num = new Number() 这种没有值的数字也归为 NaN
  return _.isNumber(obj) && obj !== +obj;
};
```


- 是否为undefined

```javascript
_.isUndefined = function(obj) {
  // 一般我们都用 if(obj) 来直接判断undefined
  // undefined 只是全局对象的一个属性，在局部环境能被重新定义
  // 但是 void 0 始终是 undefined
  return obj === void 0;
}
```


- 是否有has指定key

```javascript
_.has = function(obj, key) {
    // obj 不能为 null 或者 undefined
    return obj != null && hasOwnProperty.call(obj, key);
  };
```


参考自：<http://www.kancloud.cn/digest/underscore-source/82316>
