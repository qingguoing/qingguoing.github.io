---
title: 何为虚拟 DOM (1)
date: 2017-08-21 23:19:44
tags: ["Vue"]
category: JavaScript
---

虚拟 DOM 系列，持续分析一些当前流行的虚拟 DOM 框架和库的思想，该篇主要分析的是 Vue 框架内部采用的虚拟 DOM 库 snabbdom。

<!--more-->

### 背景

当下随着 Vue、React 等框架的流行，虚拟 DOM 大行其道。第一次听到这个词还是在15年学习 React 的时候。当时 React 以其明显高于 Angular 的性能优势迅速走红，什么虚拟 DOM 啦？什么修改 DOM 完全是操作内存中虚拟 DOM，然后再一次性更新到浏览器的真实 DOM 中？当时一听，瞬间感觉好高大上，可要问虚拟 DOM 是什么？怎么实现的？那真是一脸懵逼。时隔两年，我们来说说到底何为虚拟 DOM。

### 为何采用虚拟 DOM

众所周知，频繁访问和操作 DOM 是一种很浪费性能的行为，即使浏览器已经采取了部分措施来优化。通常性能优化的经验就是减少 DOM 的访问 和 批量更新，例如使元素脱离文档流。 但随着应用规模的扩大，前端业务处理逻辑的增强，这些远远不够。而为了更好的优化性能，大牛们就联想到了虚拟 DOM。

如今有很多流行的前端框架都内置了虚拟 DOM 算法来优化性能，例如 React，preact，inferno等，而本文接下来的内容将从 Vue 框架中的所采用的虚拟 DOM 算法 snabbdom 入手来说明何为虚拟 DOM。

### 虚拟节点

首先，snabbdom 通过如下的数据结构来存储一个虚拟节点。

```js
{
    sel: 'div.page',
    key: 0,
    data: {
        "props": {
            className: "container"
        }
    },
    children: [],
    text: 'hello world',
    elm: Element
}
```

其中，sel 属性指代该虚拟节点映射到真实 DOM 中的容器节点。key 是唯一索引值，在列表排序中可以通过该唯一值节省非必要渲染的性能。data 是该节点的一些属性，例如上述显示该节点的 className 包含 container。children 存储该节点下面的所有子节点，每个子节点又是同样的虚拟节点结构。text 属性仅当该节点仅有一个文本子节点时，值为该文本内容。elm 属性指向该虚拟节点在对应的真实 DOM 元素。

所以，现在我们应该明白，所谓的虚拟 DOM 本质上是通过恰当的数据结构和算法，来达到优化 DOM 操作性能的目的。嗯，至少目前看来，它确实是这样的。

### DOM 节点的创建

首先我们通过框架暴露的 init 方法初始化一个指定的渲染函数 patch：

```js
var patch = snabbdom.init([
  require('../../modules/class').default,
  require('../../modules/hero').default,
  require('../../modules/style').default,
  require('../../modules/eventlisteners').default,
]);
```

然后与大多数虚拟 DOM 算法类似，需要指定渲染到真实 DOM 节点的容器 patch(container, vnode)，框架内部通过深度遍历上文中的虚拟节点结构创建真实的 DOM 树。最后通过 container.parentElement.replaceChild(vnode.ele, container) 方式成功转换成真实的浏览器 DOM 节点。

### 虚拟节点的更新

在第一次创建真实 DOM 节点之后，接下来的 DOM 更新 patch(oldVnode, vnode) 就需要 diff 新老虚拟节点，具体的处理逻辑如下：

```js
//...
var elm = vnode.elm = oldVnode.elm;
//...
if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
        // 更新子节点即 children 属性
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue);
    } else if (isDef(ch)) {
        // 添加新节点
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
    } else if (isDef(oldCh)) {
        // 删除没有的旧节点
        removeVnodes(elm, oldCh, 0, oldCh.length - 1);
    }
} else if (oldVnode.text !== vnode.text) {
    // 更新节点的 textContent
    elm.textContent = vnode.text;
}
//...
```

此处摘录过来的第一行代码含有一个隐藏逻辑：并没有通过 document.createElement() 为新的虚拟节点创建真实的 elm，仍然使用的是第一步创建过的真实 DOM 容器元素。而子节点数组的 diff 更新流程如下：

![DOM 节点更新流程图](https://ws1.sinaimg.cn/large/cab372d4ly1fj9ogejeooj21oq14atfc.jpg)

上述详细的处理流程在 updateChildren 函数中。而对于子节点再包含子节点，则是通过递归循环遍历所有情形。

### 最后

snabbdom 的核心是虚拟 DOM 库，其他例如样式和事件都是通过第三方模块的方式完成的，使用者通过 init 传入需要的功能模块。而这些第三方模块实际上是通过一种生命周期钩子的方式注入的，例如：class 模块内部源码就是下面这样

```js
export const classModule = { create: updateClass, update: updateClass};
```

在 create 和 update 生命周期阶段更新元素的 class。而框架内部支持如下几种生命周期钩子：['create', 'update', 'remove', 'destroy', 'pre', 'post']。

### 总结

本文简单的描述了 Vue 框架里面引用的 snabbdom 的虚拟 DOM 思想，而这只是当下流行的虚拟 DOM 的冰山一角，下篇我们会对 React 库里面采用的虚拟 DOM 思路进行分析，这也是是令 React 大行其道的最重要原因。
