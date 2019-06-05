---
title: Koa2-源码解读(2)
date: 2018-07-08 18:50:50
tags: ['koa']
category: Node
---

本篇文章主要针对 Koa 内部源码进行解读，当前 Koa 最新的版本是 2.5.1。

<!--more-->

在上篇[Koa2-源码解读(1)]()中，我们了解到 Koa2 将一个 Node 应用简单划分为 Application、Context、Request 和 Response 四个部分。而对比 Koa2 源码发现，lib 下面的目录也就这个文件。

### Application

Application 的主要代码框架如下：

```js
const Emitter = require('events');
module.exports = class Application extends Emitter {
    constructor() {
        // 构造函数，初始化一些变量
    }
    listen(...args) {
        // 启动服务，监控传入端口号
    }
    toJSON() {}
    inspect() {}
    use(fn) {
        // 新增 middleware
    }
    callback() {
        // 返回一个 callback，用于处理 http 请求
    }
    handleRequest(ctx, fnMiddleware) {}
    createContext(req, res) {
        // 初始化一个新的 context 对象
    }
    onerror() {
        // 错误处理的回掉
    }
}
```

首先，暴露出来的 `Application` 类是继承自 node 原生的 `events` 类，主要是为了对服务过程中的错误进行监听和触发。

#### listen 方法

```js
const server = http.createServer(this.callback());
return server.listen(...args);
```

正如官方文档所介绍，该方法是包裹 `http.createServer(callback).listen()` 的语法糖。

#### callback 方法

```js
// 聚集所有相关中间件函数
const fn = compose(this.middleware);
// 错误监控
if (!this.listenerCount('error')) this.on('error', this.onerror); 
// 真正的服务会掉
const handleRequest = (req, res) => {
    // 每次请求都生成一个新的 context
    const ctx = this.createContext(req, res);
    // 结合中间件函数处理响应
    return this.handleRequest(ctx, fn);
};

return handleRequest;
```

#### createContext 方法

```js
const context = Object.create(this.context);
const request = context.request = Object.create(this.request);
const response = context.response = Object.create(this.response);
context.app = request.app = response.app = this;
context.req = request.req = response.req = req;
context.res = request.res = response.res = res;
request.ctx = response.ctx = context;
request.response = response;
response.request = request;
// ...
return context;
```

从上面代码中，我们可以看出，挂在 `context` 上的 `req` 和 `res` 对象就是原生 Node 服务中的 req 和 res。而 request 和 response 则是 Koa 内部封装的对象。

#### handleRequest 方法

```js
const res = ctx.res;
res.statusCode = 404; // 默认响应 404 状态码
const onerror = err => ctx.onerror(err);
const handleResponse = () => respond(ctx);
onFinished(res, onerror); // const onFinished = require('on-finished');
return fnMiddleware(ctx).then(handleResponse).catch(onerror);
```

上面的 `respond` 方法是处理的响应的辅助函数，主要是决定 HTTP 响应体的内容和响应提长度。

#### Middleware 处理

接下来，我们专门说下 Koa2 中所采用的中间件机制。

首先，内部把所有通过 `app.use(fn)` 添加的中间件全部维护在一个 `this.middleware` 的中间件数组内部。然后在上文中的 callback 方法里通过 `koa-compose` 揉和使用。compose 方法如下：

```js
// koa-compose
function compose(middleware) {
    // ...(省略参数检验)
    return function (context, next) {
        // last called middleware #
        let index = -1
        return dispatch(0)
        function dispatch (i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))
            index = i
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

再结合 `handleRequest` 方法里的使用 `fnMiddleware(ctx).then(handleResponse).catch(onerror);`，不难猜出，compose 方法返回的函数的形参 next 和正常书写的中间件的 next 参数是一个，均指向中间件数组中，当前正在被执行的中间件函数所在位置的后一个即将要被处理的中间件函数。当最后一个中间件函数执行时，此时 next 方法就是 undefined，直接返回 Promise.resolve()。

至此，整个中间件链路已经完全清晰，即我们常说的洋葱切面中间件流程。

### context

上文已经解释了每次请求来临时，都会创建一个新的 context 对象实例。而源码内部，主要是通过 `delegates` 库，把相关的 request 和 response 方法挂在 context 上。此处，就不做过多解释。

### request 和 response

request 内部针对原生的 req 对象，提炼出了 header、method、url 等更简单和方便使用的 Api。response 基于 res 对象，提供了更简洁的 status、body、length 等方法。此处也不做过多解释。

### 总结

本文主要结合 Koa2 源码，针对内部的中间件相关处理过程进行了分析。
