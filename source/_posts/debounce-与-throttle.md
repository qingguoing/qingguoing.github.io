---
title: debounce 与 throttle 源码分析
date: 2018-10-29 23:39:44
tags: ["Performance"]
category: JavaScript
---

`lodash` 中的 `debounce` 和 `throttle` 源码分析。

<!--more-->

今晚写了个 node 脚本去拉去后端数据库的某些记录，可能同一时刻并发量太高，造成了 QPS 短时间内过大，引起了后端报警，尴尬。。。而为了防止再次出现这种短时间内 QPS 过高的现象，就想到了 `debounce` 和 `throttle` 函数。此处只记录个人见解和 lodash 中的对应源码分析，不做其它过多解释，详情大家可以参考 [David Corbacho's article](https://css-tricks.com/debouncing-throttling-explained-examples/)


### debounce(防抖)

第一次了解防抖函数时，是拿弹簧的弹性效果做对比。即在压缩或者拉伸弹簧时，为了防止中途弹簧出现弹性抖动的现象，我们可以采取一直延迟某种『事件』的触发，直到满足某个理想条件后的某个时刻才开始触发。
不过刚刚看了上面 David Corbacho 的文章，发现用日常生活中的电梯来必须更形象。下面是 debounce 的源码。

```js
function debounce(func, wait, options) {
  let lastArgs,
    lastThis,
    maxWait,
    result,
    timerId,
    // 最近调用 debounce 封装的函数时间
    lastCallTime
  // 最近调用传入 func 函数的时间
  let lastInvokeTime = 0
  // 是否立即调用
  let leading = false
  let maxing = false
  let trailing = true

  // wait 不存在时采用 requestAnimationFrame 方法，而不是 setTimeout 0
  const useRAF = (!wait && wait !== 0 && typeof root.requestAnimationFrame === 'function')

  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  wait = +wait || 0
  if (isObject(options)) {
    leading = !!options.leading
    // 等多久，必须调用一次
    maxing = 'maxWait' in options
    maxWait = maxing ? Math.max(+options.maxWait || 0, wait) : maxWait
    trailing = 'trailing' in options ? !!options.trailing : trailing
  }

  // 调用原始函数
  function invokeFunc(time) {
    const args = lastArgs
    const thisArg = lastThis

    lastArgs = lastThis = undefined
    lastInvokeTime = time
    result = func.apply(thisArg, args)
    return result
  }

  // 计时器
  function startTimer(pendingFunc, wait) {
    if (useRAF) {
      return root.requestAnimationFrame(pendingFunc)
    }
    return setTimeout(pendingFunc, wait)
  }
  // 取消计时器
  function cancelTimer(id) {
    if (useRAF) {
      return root.cancelAnimationFrame(id)
    }
    clearTimeout(id)
  }
  // 立即调用原函数
  function leadingEdge(time) {
    lastInvokeTime = time
    timerId = startTimer(timerExpired, wait)
    return leading ? invokeFunc(time) : result
  }
  // 计算继续等待时间
  function remainingWait(time) {
    const timeSinceLastCall = time - lastCallTime
    const timeSinceLastInvoke = time - lastInvokeTime
    const timeWaiting = wait - timeSinceLastCall

    return maxing
      ? Math.min(timeWaiting, maxWait - timeSinceLastInvoke)
      : timeWaiting
  }
  // 是否需要执行：（1）首次执行（2）超过等待时间（3）时间后腿的情况（4）超过最长等待时间
  function shouldInvoke(time) {
    const timeSinceLastCall = time - lastCallTime
    const timeSinceLastInvoke = time - lastInvokeTime
    return (lastCallTime === undefined || (timeSinceLastCall >= wait) ||
      (timeSinceLastCall < 0) || (maxing && timeSinceLastInvoke >= maxWait))
  }
  // setTimeout 的计时器回调
  function timerExpired() {
    const time = Date.now()
    if (shouldInvoke(time)) {
      return trailingEdge(time)
    }
    // 计时器到了，但不满足执行的条件
    timerId = startTimer(timerExpired, remainingWait(time))
  }
  // 调用原函数
  function trailingEdge(time) {
    timerId = undefined
    if (trailing && lastArgs) {
      return invokeFunc(time)
    }
    lastArgs = lastThis = undefined
    return result
  }
  // 取消 debounce 效果
  function cancel() {
    if (timerId !== undefined) {
      cancelTimer(timerId)
    }
    lastInvokeTime = 0
    lastArgs = lastCallTime = lastThis = timerId = undefined
  }
  // 立即执行
  function flush() {
    return timerId === undefined ? result : trailingEdge(Date.now())
  }
  // 是否仍在等待中
  function pending() {
    return timerId !== undefined
  }
  // 封装后对外暴露的函数
  function debounced(...args) {
    const time = Date.now()
    const isInvoking = shouldInvoke(time)

    lastArgs = args
    lastThis = this
    lastCallTime = time

    if (isInvoking) {
      if (timerId === undefined) {
        return leadingEdge(lastCallTime)
      }
      if (maxing) {
        timerId = startTimer(timerExpired, wait)
        return invokeFunc(lastCallTime)
      }
    }
    if (timerId === undefined) {
      timerId = startTimer(timerExpired, wait)
    }
    return result
  }
  debounced.cancel = cancel
  debounced.flush = flush
  debounced.pending = pending
  return debounced
}
```

### throttle(节流)

throttle 的效果可根据字面意思理解，类似滴水的水龙头效果，会在一定时间内积累一滴水后滴落下来。所以 throttle 的效果可以基于 debounce 的 maxWait 效果实现。

```js
// maxWait 和 wait 同值
function throttle(func, wait, options) {
  let leading = true
  let trailing = true

  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  if (isObject(options)) {
    leading = 'leading' in options ? !!options.leading : leading
    trailing = 'trailing' in options ? !!options.trailing : trailing
  }
  return debounce(func, wait, {
    'leading': leading,
    'maxWait': wait,
    'trailing': trailing
  })
}
```
