---
title: Koa2 源码解读(1)
date: 2018-07-07 21:50:35
tags: ['koa']
category: Node
---

本篇文章简单介绍 Koa2 的使用方法和相关 Api。当前 Koa 最新的版本是 2.5.1。

<!--more-->

截至本文编写时，当前 Koa 最新的版本是 2.5.1。本篇为 Koa 源码解读系列第一篇，主要讲述 Koa 的概念和使用方式。如果你已经对此很熟悉，可完全跳过该篇文章，或者直接移步至[官方文档](https://koajs.com/)。

Koa 源码同 redux 源码类似，简洁直观，扩展功能均是通过中间件的方式来实现。核心概念名字主要有 Application、context、request、response。

### Application

顾名思义，一个 Application 就是初始化 Koa 后生成的一个对象。

```javascript
const Koa = require('koa');
const app = new Koa();
```

这个 app 对象含有一系列方法，以及一些内置的中间件函数用于处理 HTTP 请求。下面是最典型的一种方式：

```javascript
app.use(ctx => {
    ctx.body = 'hello world';
});

app.listen(3000);
```

上述中 `app.use(function)` 就是一种专门用于往应用中增加中间件的方式。

#### Middleware

Koa2 默认提供了很少的功能，绝大部分情况下我们都需要通过中间件的方式来增强 Koa 的处理能力。而中间件的使用方式也很简单，其中 `ctx` 就是下文中的 `context`，`next` 是可以理解成下一个中间件函数，通过执行 next 函数来跳出当前中间件函数而进入下一个中间件函数。可以通过 [官方 middleware 链接](https://github.com/koajs/koa/wiki#middleware) 查看已有的中间件列表，或者新增一个 Middleware。

```javascript

// x-response-time

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});

// logger

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}`);
});

// response

app.use(async ctx => {
  ctx.body = 'Hello World';
});

```

#### 错误处理

koa 中应用层错误的监听通过如下方式：

```javascript
app.on('error', (err, ctx) => {
  log.error('server error', err, ctx);
});
```

### Context

Context 可以看成是下文中的 request 和 respond 结合体，提供了许多辅助性函数，主要用于提升开发体验。针对每一个请求，都会初始化一个新的 Context 对象，而且可以贯穿整个应用，方便快速的设置和获取相关属性。例如上文中间件函数接受的 `ctx` 参数。

### Request

由 Koa 抽象出的一个请求对象，专门针对 Node 服务端开发而增加了需要辅助性 api。例如 `request.header`、`request.url` 等。

### Response

类似于 Request 对象，主要是包含对应的响应对象。例如 `response.status`、`response.body` 等。

### 总结

Koa 的核心概念和使用方式简洁明了，主要是采用了中间件的处理方式。下一节，我们将通过深入 Koa 源码内部剖析其相关的应用机制。