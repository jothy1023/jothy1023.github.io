---
title: javascript 原生 bind() 的 ES6 + ES5 实现
date: 2016-10-16 20:19:06
tag: [javascript, es6, es7]
---
> 前言： 本来只是想写一下简单的 bind 函数实现，没想到写着写着还能牵出 js 中继承的知识，其实研究原生函数的实现总是能学到很多新东西

在实现之前呢，我们首先要知道 `bind` 是做什么的。JS MDN 给出的定义是 *The bind() methods creates a new function that, when called, has its this keyword set to the provided value, with a given sequence of arguments preceding any provided when the new function is called.* ，简单来说就是 `bind()` 函数创建了一个新函数（原函数的拷贝），这个函数接受一个提供新的 `this` 上下文的参数，以及之后任意可选的其他参数。当这个新函数被调用时，它的 `this` 关键字指向第一个参数的新上下文。而第二个之后的参数会与原函数的参数组成新参数（原函数的参数在后），传递给函数。  

弄清楚这个之后，我们再来分析看看要怎么做。  

首先，调用 bind() 会返回一个闭包，这个闭包中创建了一个新函数，这个函数首先包含原函数的属性与方法，并且这个函数的 this 值是传给 bind() 函数第一个参数，所以自然而然我们想到用 call 或者 apply 来改变原函数的 this ，这里我们选择 apply， 理由是我们新函数的第一个之后的参数是由传给 bind() 的第二个及之后的参数（代码中的 formerArgs ）再加上原函数的参数（代码中的 laterArgs ）构成的，我们把它们拼接成一个数组就完事儿了~具体代码如下：

```js
Function.prototype.bind = function (ctx) {
    // 保存原函数的 this 至 _this
    var _this = this
    // slice 使用了两次，保存到变量中
    // slice 主要用于类数组对象 arguments 的浅复制
    var slice = Array.prototype.slice
    // 传给 bind() 函数的第二个至之后的参数，从  arguments 的第二位开始
    var formerArgs = slice.call(arguments, 1)

    // bind() 本身就是一个函数，返回
    return function (){
        // 传给原函数的参数
        let laterArgs = slice.call(arguments, 0)
        // 返回一个函数，这个函数调用了原函数，并且 this 指向 bind 的第一个参数，
        // 第二个参数由 formerArgs  与 laterArgs组成 
        return _this.apply(ctx, formerArgs.concat(laterArgs))
    }
}
```

#### ES6 实现
上面的代码是基于 ES5 实现的，当时对于不确定的参数的一般处理方法都是利用类数组对象 `arguments` （其中包含了传递给函数的所有参数），也就免不了使用 `call` 或者是 `apply` 对其进行数组操作。代码也就显得比较冗长。但是 ES6 不一样了呀~我们有了**不定参数**这个神器。无论有无参数，有几个参数都可以简单地处理。

> 不定参数： 传递给函数的最后一个参数可以被标记为不定参数，当函数被调用时，不定参数之前的参数都可正常被填充，剩下的参数会被放进一个数组中，并被赋值给不定参数。而当没有剩下的参数时，不定参数会是一个空数组，而不会被填充为 `undefined` 。

同样的功能，只要几行代码就可以实现：

```js
// formerArgs 为传递给 bind 函数的第二个到之后的参数
Function.prototype.bind = function (ctx, ...formerArgs) {
    let _this = this
    
    // laterArgs 为传递给原函数的参数
    return (...laterArgs) => {
        // bind 函数的不定参数在原函数参数之前，formerArgs 本身就是数组，可以直接调用数组的 concat 方法，无需借助 call 或 apply
        return _this.apply(ctx, formerArgs.concat(laterArgs))
    }
}
```

至此，我们就实现了简单的 bind() 函数的功能，接下来我们给它做点优化。
#### 优化 upupup..
- 当 Function 的原型链上没有 bind 函数时，才加上此函数

```js
if (!Function.prototype.bind) {
    // add bind() to Function.prototype
}
```

- 只有函数才能调用 bind 函数，其他的对象不行。即判断 this 是否为函数。

```js
if (typeof this !== 'function') {
    // throw NOT_A_FUNCTION error
}
```

- **压轴戏： 关于继承**
我们上面的代码使用了借用 apply 继承的方式。用了 apply 来改变 this 的指向，继承了原函数的基本属性和引用属性，并且保留了可传参优点，但是新函数无法实现函数复用，每个新函数都会复制出一份新的原函数的函数，并且也无法继承到原函数通过 prototype 方式定义的方法或属性。
> 为解决以上问题，我们选用 **组合继承** 方式，在使用 apply 继承的基础上，加上了原型链继承。
所以我们可以这么改。

```js
if (!Function.prototype.binds) {
  Function.prototype.binds = function (ctx) {
    if (typeof this !== 'function') {
      throw new TypeError("NOT_A_FUNCTION -- this is not callable")
    }
    var _this = this
    var slice = Array.prototype.slice
    var formerArgs = slice.call(arguments, 1)
    // 定义一个中间函数，用于作为继承的中间值
    var fun = function () {}

    var fBound = function (){
      let laterArgs = slice.call(arguments, 0)
      return _this.apply(ctx, formerArgs.concat(laterArgs))
    }

    // 先让 fun 的原型方法指向 _this 即原函数的原型方法，继承 _this 的属性
    fun.prototype = _this.prototype
    // 再将 fBound 即要返回的新函数的原型方法指向 fun 的实例化对象
    // 这样，既能让 fBound 继承 _this 的属性，在修改其原型链时，又不会影响到 _this 的原型链
    fBound.prototype = new fun()

    return fBound
  }
}
```

在上面的代码中，我们引入了一个新的函数 fun，用于继承原函数的原型，并通过 new 操作符实例化出它的实例对象，供 fBound 的原型继承，至此，我们既让新函数继承了原函数的所有属性与方法，又保证了不会因为其对原型链的操作影响到原函数。用图来表示应该是下面这样的：

![image](http://note.youdao.com/yws/public/resource/80df4dc9bc62843524147237b56b847e/xmlnote/115F6B5CD1A74571A185AECF18F957EA/153)

这样我们对新函数的 prototype 修改只会应用在它自己身上，而不会影响到原函数。

- 其他
MDN 中还提到，若是将 bind 绑定之后的函数当作构造函数，通过 new 操作符使用，则不绑定传入的 this，而是将 this 指向实例化出来的对象。所以我们最终的代码为：

```js

if (!Function.prototype.binds) {
  Function.prototype.binds = function (ctx) {
    if (typeof this !== 'function') {
      throw new TypeError("NOT_A_FUNCTION -- this is not callable")
    }
    var _this = this
    var slice = Array.prototype.slice
    var formerArgs = slice.call(arguments, 1)
    var fun = function () {}

    var fBound = function (){
      let laterArgs = slice.call(arguments, 0)
      // 若通过 new 调用 bind() 之后的函数，则这时候 fBound 的 this 指向的是 fBound 实例，
      // 而下面又定义了 fBound 是 fun 的派生类（其 prototype 指向 fun 的实例），
      // 所以 this instanceof fun === true ，这时 this 指向了 fBound 实例，不另外绑定！
      return _this.apply(this instanceof fun ? this : ctx || this, formerArgs.concat(laterArgs))
    }

    fun.prototype = _this.prototype
    fBound.prototype = new fun()

    return fBound
  }
}
```

打完收工~~欧耶 ( •̀ ω •́ )y

> P.S: ES7 中已经淘汰了 .bind 的写法，而是使用两个冒号的方式 :: 来代替，（要不要这么可爱！不过呢，人家还只是个提案而已，到时候会不会被采用还不一定的呢

用法有两种，第一种  `::对象.方法名`

```js
let obj = {
  val: 'value',
  method: function () {
      console.log(this.val)
  }
};

// ::对象.方法名
// 等价于 obj.method.bind(obj)
// 将 method 的 this 绑定为 obj 
// 这里的 method 必须是 obj 的方法
::obj.method;

// call
obj.method(); // value

```

第二种， `对象::方法名()`

```js
let obj = {
  val: 'value'
};

function method () {
  console.log(this.val);
}

// 对象::方法名() 
// 等价于 method.call(obj) 或 method.apply(obj)
// 会直接调用并输出 'value'
// obj::method();


// 对象::方法  （无括号
// 等价于 method.bind(obj)
// 不会自动执行，必须赋值给 method 才能实现绑定
// 手动调用 method ，输出 vaue
method = obj::method;
method();

```

嘛~ 有说的不对的地方，欢迎指正..