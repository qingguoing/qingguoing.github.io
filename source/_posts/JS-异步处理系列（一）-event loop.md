---
title: JS 异步处理系列（一）—— Event Loop
date: 2019-05-13 22:40:18
tags: ["JS 异步处理系列", "EventLoop"]
category: JavaScript
---

本文是 JavaScript 中的异步处理系列第一篇，介绍 JS 中的事件循环（event loop）概念和原理。

<!--more-->

### 背景

众所周知，JS 的执行是单线程的。主要是由于 JS 可以操作 DOM 和 CSS 样式，而为了防止多线程之间操作出现的冲突，所以 JS 是单线程的。
但 JS 中还是需要『并发』机制的，例如网络请求 xhr，如果结果返回比较慢，浏览器一直等待的话，就会出现页面假死的状态，即用户点击无反应。而 JS 的并发模型就是基于『事件循环』。

### 事件循环

![浏览器事件循环基本模型](./eventLoop.png)

 - heap: 存储对象数据。
 - stack: JS 代码调用栈，按照 FILO 原则。例如当执行函数 a 时，函数 a 会被放入调用栈中，调用栈中维护的是当前函数的执行上下文等。当该函数执行完毕时，被弹出栈。这是 JS 同步代码的执行过程。
 - webapis: 浏览器封装提供的强大异步 API，可被 JS 直接调用执行。
 - callback queue: 回调事件队列。

```js
function main(){
  console.log('A');
  setTimeout(
    function display(){ console.log('B'); }
  ,0);
	console.log('C');
}
main(); // A C B
```

JS 引擎执行上面代码的过程：
![execution](./execution.png)

1. 首先 `main` 的调用作为一帧（frame）被放入调用栈中。然后 JS 引擎把函数的第一条调用语句 `console.log('A'）` 推入调用栈中，执行完成后出栈。
2. 接下来 `setTimeout` 被放入调用栈中开始执行。执行完毕之后出栈。
3. `console.log('C')` 入栈。由于上面的计时器函数是 0ms，回调函数 `exec` 被放入消息队列中。
4. `main` 函数的最后一条语句执行完毕后被推出调用栈。此时，调用栈为空，主进程开始询问回调消息队列是否有回调需要执行。
5. 现在会调函数被推入调用栈中开始执行，打印出 C。这就是 JS 的事件循环。

而由于异步任务类型的多样性，不同的异步任务根据执行的优先级，被分为两类：宏任务（macro task）和微任务（micro task）。

### 宏任务 vs 微任务

宏任务代表：

 - `setTimeout`
 - `setInterval`

微任务代表：
 - `new MutationObserver`
 - `new Promise()`

上面介绍过，异步回调首先会被放到回调消息队列中。而根据异步事件的类型，该回调实际上是会被放到宏任务队列或者微任务队列中。在调用栈为空时，主线程会先去查看微任务队列是否有回调存在。如果不存在，那么再去查看宏任务队列；如果存在，则会依次取队列中的回调到调用栈执行，直到微任务队列为空，然后再去宏任务队列中，取相应的回调执行。如此反复，进入循环。

```js
function main() {
  console.log('A');
  setTimeout(() => console.log('B'));
  Promise.resolve().then(() => console.log('C'));
  console.log('D');
}

main(); // A D C B
```

### 总结

虽然 JS 的执行是单线程的，但是 JS 中还是有异步的回调和并发的概念。本文主要介绍浏览器中的 JS event loop，同时分析了相关调用步骤。

本文参考链接：
1. [JavaScript Event Loop Explained
](https://medium.com/front-end-weekly/javascript-event-loop-explained-4cd26af121d4)