---
title: node.js同步获取request请求内容
date: 2016-07-09 20:19:06
tag: [nodejs, request, Promise, 同步, 异步]
---
&emsp;nodejs的异步执行给我们的编程带来了数不尽的好处，以至于我已经习惯了异步编程，某一次在执行request请求想要将其返回的body内容体传到request函数外时，发现怎么也获取不了，由于对node理解的不够深入透彻，我首先怀疑的竟然是**会不会request不支持返回操作 or 难道操作仅在函数作用域内有效**，后来才搞明白是异步执行惹的事儿，取值那会儿其实还没赋值呢。现在想想自己还是too naive..

&emsp;搞明白了是异步的问题之后，便想着如何把异步执行变同步执行，实现的方法有很多，我用的是Promise来实现。具体例子如下：

&emsp;简单地请求一下github并输出状态码 = =

```javascript
"use strict"

let request =  require('request');
let test = 'I am test';

// async
request({
    url: 'https://github.com',
    method: 'get'
}, (err, res, body) => {
    if (res && res.statusCode === 200) {
      test = res.statusCode + 'ok!';
      console.log('inside request: ' + test);
    } else {
      test = 'error - -';
    }
});

console.log('outside request: ' + test);
```

&emsp;执行结果如下：
```
$ node request.js
outside request: I am test
inside request: 200 ok!
```
&emsp;其实单从结果就可以看到，outside的执行是先于inside的，但是懵逼的我当时并没有发现，oops..

&emsp;言归正传，用上了Promise之后的画风是这样的：

```javascript
"use strict"
let test = 'I am test';
let request = require('request');

// sync
new Promise((resolve, reject) => {
    request({
        url: 'https://github.com',
        method: 'get'
    }, (err, res, body) => {
        if (res && res.statusCode === 200) {
            resolve(res.statusCode + ' ok!');
        } else {
            reject(' error - -');
        }
    });
}).then(result => {
    test = result;
    console.log("outside request: " + test);
}).catch(err => {
    console.log("error: " + err)
})

```

&emsp;执行结果：
```
$ node request.js
outside request: 200 ok!
```

&emsp;这样一来，就能实现在request外也获取其内容的功能啦~~只要把代码都丢到then里面就可以了。

&emsp;好吧，写完之后发现，这他喵的这么简单我怎么会折腾这么久！ T T

&emsp;最后的最后，想告诫自己

> 前端路阻且长，且行且珍惜。
