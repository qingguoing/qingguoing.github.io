---
title: JS 异步处理系列（四）- async & await
date: 2019-05-20 21:24:12
tags: ["JS 异步处理系列", "Async"]
category: JavaScript
---

本文是 JavaScript 中的异步处理系列第四篇，介绍异步编程的终极使用方法 async await。

<!--more-->

### 前言

前面我们介绍了 JavaScript 异步编程中的常见写法，callback 和 Promise。现在我们看下异步编程的终极大招 - async & await。不过再介绍 async & await 之前，我们需要首先了解 Generator 函数和用法。

### Generator

Generator 是 ES6 中提到的一种异步编程解决方案，详细的使用和介绍可以参考之前的文章 [ES6 学习笔记(2)](https://qingguoing.com/2017/09/25/ES6-%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/)。下面是一个简单的 Generator 的例子：

```js
function * a() {
  yield 1;
  yield 2;
  return 3;
}

const test = a();
test.next(); // { value: 1, done: false }
test.next(); // { value: 2, done: false }
test.next(); // { value: 3, done: true }
```

而说起 Generator 就不得不提 [co](https://github.com/tj/co)。一个可以让 Generator 函数自动调用返回异步结果的库，具体使用可以点击上面 co 链接。此处只给出对应的 co 原理解释。

### co

从上文中的 Demo 可以看到，Generator 函数的每次调用都需要调用 `next` 方法返回对应的结果，再结合 JS 的一个函数可以作为另一个函数参数的特性，我们可以手动实现出类似下面的自动调用代码：

```js
function test(fn) {
  const run = (gen) => {
    let res = gen.next();
    console.log(res.value);
    if (res.done) {
      return;
    }
    run(gen);
  }
  run(fn());
}
```

设想以下，如果 Generator 函数中的 `yield` 关键字后面跟的是一个异步函数，我们就可以利用 `thunk` 的原理，在每一个异步函数的回调里去调用 `next` 方法，实现多个异步方法的串行调用。

```js
var fs = require('fs');

function read(file) {
  return function(fn){
    fs.readFile(file, 'utf8', fn);
  }
}

co(function *(){
  var a = yield read('.gitignore');
  var b = yield read('test.js');
  var c = yield read('package.json');
  console.log(a);
  console.log(b);
  console.log(c);
});
```
下面是我们针对上面的代码调用实现的一个粗略定制版的 co ：

```js
const co = function (fn) {
  const gen = fn();

  const ret = gen.next();
  ret.value((err, res) => {
    if (err) {
      gen.throw(err);
      return;
    }
    const ret = gen.next(res);
    ret.value((err, res) => {
      if (err) {
        gen.throw(err);
        return;
      }
      const ret = gen.next(res);
      ret.value((err, res) => {
        if (err) {
          gen.throw(err);
          return;
        }
        const ret = gen.next(res);
      })
    });
  });
}
```

而下面是 co 官方 v1.5.1 版本的核心源码，即通过递归调用的方式实现。

```js
function co(fn, done, ctx) {
  var gen = isGenerator(fn) ? fn : fn.call(this);
  ctx = ctx || this;

  function next(err, res) {
    var ret;

    // multiple args
    if (arguments.length > 2) {
      res = [].slice.call(arguments, 1);
    }

    // error
    if (err) {
      try {
        ret = gen.throw(err);
      } catch (e) {
        if (!done) throw e;
        return done(e);
      }
    }

    // ok
    if (!err) {
      try {
        ret = gen.next(res);
      } catch (e) {
        if (!done) throw e;
        return done(e);
      }
    }

    // done
    if (ret.done) {
      if (done) done(null, ret.value);
      return;
    }

    // normalize
    ret.value = toThunk(ret.value, ctx);

    // run
    if ('function' == typeof ret.value) {
      try {
        ret.value.call(ctx, next);
      } catch (e) {
        setImmediate(function(){
          next(e);
        });
      }
      return;
    }

    // invalid
    next(new Error('yield a function, promise, generator, or array'));
  }

  if (done) next();
  else setImmediate(next);

  return function(fn){
    done = fn;
  }
}

function toThunk(obj, ctx) {
  var fn = obj;
  if (Array.isArray(obj)) fn = exports.join.call(ctx, obj);
  if (isGeneratorFunction(obj)) obj = obj.call(ctx);
  if (isGenerator(obj)) fn = function(done){ co(obj, done, ctx) };
  if (isPromise(obj)) fn = promiseToThunk(obj);
  return fn;
}
```

### Promise

上面是通过在异步 api 的 callback 的里面递归地调用 `next` 方法，实现多个异步 api 的串行调用，结合上一篇的 Promise，我们也可以通过在 Promise 的 `then` 方法内部，递归的调用 `next` 方法实现类似的效果，具体的源码大家可以参考 co v4.0 以后版本的源码。

### async & await

结合上文的 Generator、co 和 Promise，其实就可以实现 async 的异步编程。下面是 babel 转换前后的代码对比。

```js
async function test() {
  let res = null;
  try {
  	res = await get('https://baidu.com');
  } catch(err) {
  	return null;
  }
  return res;
}
```

转换后：

```js
function asyncGeneratorStep(gen, resolve, reject, _next, _throw, key, arg) {
  try {
    var info = gen[key](arg);
    var value = info.value;
  } catch (error) {
    reject(error);
    return;
  }
  if (info.done) {
    resolve(value);
  } else {
    Promise.resolve(value).then(_next, _throw);
  }
}

function _asyncToGenerator(fn) {
  return function () {
    var self = this, args = arguments;
    return new Promise(function (resolve, reject) {
      var gen = fn.apply(self, args);
      function _next(value) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "next", value);
      }
      function _throw(err) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "throw", err);
      }
      _next(undefined);
    });
  };
}

function test() {
  return _test.apply(this, arguments);
}

function _test() {
  _test = _asyncToGenerator(function* () {
    let res = null;

    try {
      res = yield get('https://baidu.com');
    } catch (err) {
      return null;
    }

    return res;
  });
  return _test.apply(this, arguments);
}
```

### 总结

本文是 JS 异步处理系列最后一篇，也是书写本系列的初衷，从 Generator 函数和 Co 原理入手，到 async/await 原理结束，重点描述异步编程的前因后果，以及透过现象看到底层的本质。