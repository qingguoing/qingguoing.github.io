---
title: H5 History API
date: 2017-04-21 23:42:10
tags: H5
category: JavaScript
---

通过H5的 history 和 location.hash 操作浏览器地址栏。

<!--more-->

HTML5规范早在2014年就已版本，而现在各大主流浏览器厂商早已实现对其的部分或者完全支持。网上记录关于 history 模块的文章也有很多。而最近在拜读 react-router v4 的源码时，发现了更多的一点东西，遂记录在案。

### History

window 对象通过 history 对象提供对浏览器历史的访问。它暴露了有用的方法和属性，让您可以来回浏览用户的历史记录，以及从 HTML5 开始操纵历史堆栈的内容。

#### length

浏览器当前回话面板中历史元素的数目。例如，在 Chrome 中新开一个 tab 页，此时该面板中的 history.length 为1。

#### state

返回一个表示历史堆栈顶部的状态的值，即当前回话面板中的当前浏览记录绑定的状态值，默认是 null，会与当前会话面板一直存在中。可以通过下文中的 pushState 和 replaceState 方法手动更改该值。

#### scrollRestoration

允许Web应用程序在历史导航上显式地设置默认滚动恢复行为。此属性可以是自动的（auto）或者手动的（manual）。默认是自动的。

#### back()

类似于浏览器返回按钮，返回到上一页。

#### forward()

类似于浏览器前进按钮，进入到下一页。

#### go(index)

相对于当前所在位置，导航到浏览器历史堆栈中的任一位置。例如 history.go(0) 刷新当前页，history.go(1) 向前一页，等同于 history.forward()，history.go(-1) 返回到前一页，等同于 history.back()。

#### pushState(state, title, url)

向当前面板浏览历史记录中推进一条新的记录，其中 state 参数对应上文中的 history.state，给下一条历史记录绑定state数据。参数 title 目前各大浏览器已经忽略这个参数，可以传 null 替代。参数 url 是浏览器地址栏将要显示的地址，可以是相对地址或者绝对地址。相对地址是相对于当前地址，绝对地址则要与当前地址处在同一源下，否则浏览器会直接报错，涉及跨域问题。

该方法只是该变当前的历史堆栈，不会去主动请求页面 url 的内容和判断该页面是否存在。即通过调用改方法，只是浏览器地址栏中的地址更新了，当页面内容并没有刷新。

#### replaceState(state, title, url)

参数原理和 pushState 一致。不过是在在浏览历史堆栈中用新的 url 替换掉当前的 url。

### Hash

window.location.hash 是指 URL 中的锚部分，即从 # 号开始之后的部分，表示这个位置的标识符。例如通过 HTML 实现一个简单的回到顶部按钮的效果原理：首先在页面顶部指定一个标识符，有如下两种方法：

```html
<a name="top"></a>
<div id="top"></div>
```

然后在页面右下角放置一个回到顶部的按钮：`<a href="#top">回到顶部</a>`

#### HTTP请求不包括HASH部分

因为 hash 部分只在浏览器端生效，表示锚点，所以不会被传送到服务器端。但是我们可以通过 encodeURIComponent 将 # 转成 %23 ，例如百度搜索就是采用这种方式。

#### HASH的改变

改变 hash 部分的效果等同于上文中的 history.pushState 的方式，会生成新的浏览历史记录，但不会刷新或者重载网页的内容。

#### HASH的替换

通过 window.location.replace(newUrl) 来模仿 history.replaceState ，替换当前页面的 hash 部分而又不生成新的浏览历史记录。

### popstate

当激活状态的历史记录条目发生变化时，window.popstate 事件就会在对应 window 对象上触发。如果当前处于激活状态的历史记录条目是由 history.pushState() 方法创建,或者由 history.replaceState() 方法修改过的, 则 popstate 事件对象的 state 属性包含了这个历史记录条目的 state 对象的一个拷贝.

但是调用 history.pushState() 或者 history.replaceState() 不会触发 popstate 事件。popstate 事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者在JavaScript中调用 history.back()、history.forward()、history.go() 方法)。另外，拖过其他方式更新页面的 hash 部分也会触发 popstate 事件（IE10、IE11除外）。

### hashchange

hashchange 事件在当前页面中的 hash 值改变时触发。hashchange 事件对象有如下两个属性：

```js
{
    oldURL, // 当前页面旧的url
    newURL, // 当前页面新的url
}
```

### 最后的最后

虽然上文中包含了很多 HTML5 新增的一些事件和方法，但随着最近几年各大浏览器厂商对 HTML5 的普及，hashchange 这种 HTML5 中新增的事件早已在 IE8 中实现了。不过对于 history.pushState() 和 popState 事件等其他方法，则只能在 IE10+ 中运行。而目前对于一些路由框架，则都提供了这两种实现方式：通过 history.pushState 等方式实现的 createBrowserHistory 和通过 window.location.hash 等方式实现的 createHashHistory。

本文参考链接：

1. https://developer.mozilla.org/zh-CN/docs/DOM/Manipulating_the_browser_history

2. https://developer.mozilla.org/zh-CN/docs/Web/API/Window/onpopstate

3. https://developers.google.com/web/updates/2015/09/history-api-scroll-restoration