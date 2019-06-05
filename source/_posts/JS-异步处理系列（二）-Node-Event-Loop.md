---
title: JS 异步处理系列（二）- Node Event Loop
date: 2019-05-14 23:07:16
tags: ["JS 异步处理系列", "Node", "EventLoop"]
category: JavaScript
---

本文是 JavaScript 中的异步处理系列第二篇，介绍 NodeJS 中的事件循环（event loop）概念和原理。

<!--more-->

### 前言

第一篇中，我们介绍了浏览器环境下的 event loop 模型，而 NodeJS 也有对应的 event loop 模型，并且和浏览器下的事件循环完全不同。本文着重介绍 NodeJS 下的事件循环模型。

### 事件循环

事件循环是 Node.js 处理非阻塞 I/O 操作的机制——尽管 JavaScript 是单线程处理的——当有可能的时候，它们会把操作转移到系统内核中去。

![NodeJS 事件循环机制](./eventloop.jpeg)

注：每个框框里每一步都是事件循环机制的一个阶段。

每个阶段都有一个 FIFO 队列来执行回调。虽然每个阶段都是特殊的，但通常情况下，当事件循环进入给定的阶段时，它将执行特定于该阶段的任何操作，然后在该阶段的队列中执行回调，直到队列用尽或最大回调数已执行。当该队列已用尽或达到回调限制，事件循环将移动到下一阶段。

 - `timers`：本阶段执行已经安排的 setTimeout() 和 setInterval() 的回调函数。
 - `pending callbacks`：执行延迟到下一个循环迭代的 I/O 回调。
 - `idle, prepare`：仅系统内部使用。
 - `poll`：检索新的 I/O 事件;执行与 I/O 相关的回调（几乎所有情况下，除了关闭的回调函数，它们由计时器和 setImmediate() 排定的之外），其余情况 node 将在此处阻塞。
 - `check`：setImmediate() 回调函数在这里执行。
 - `close callbacks`：一些准备关闭的回调函数，如：socket.on('close', ...)。

在每次运行的事件循环之间，Node.js 检查它是否在等待任何异步 I/O 或计时器，如果没有的话，则关闭干净。

### `setImmediate()` vs `setTimeout()`

`setImmediate()` 和 `setTimeout()` 很类似，但何时调用行为完全不同。

 - `setImmediate()` 设计为在当前 轮询 阶段完成后执行脚本。
 - `setTimeout()` 计划在毫秒的最小阈值经过后运行的脚本。

执行计时器的顺序将根据调用它们的上下文而异。如果二者都从主模块内调用，则计时将受进程性能的约束。

例如，如果运行的是不属于 I/O 周期（即主模块）的以下脚本，则执行两个计时器的顺序是非确定性的，因为它受进程性能的约束：

```js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

但是，如果你把这两个函数放入一个 I/O 循环内调用，`setImmediate` 总是被优先调用：

```js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

使用 `setImmediate()` 超过 `setTimeout()` 的主要优点是，如果在 I/O 周期内，`setImmediate()` 总是比其它计时器先执行，不管有多少个其它计时器。

### `process.nextTick()`

任何时候在给定的阶段中调用 `process.nextTick()`，所有传递到 `process.nextTick()` 的回调将在事件循环继续之前得到解决。即无论事件循环的当前阶段如何，都将在当前操作完成后处理 `nextTickQueue`。

### 总结

本文介绍了 NodeJS 中的 event loop 模型。

参考链接：
1. [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)