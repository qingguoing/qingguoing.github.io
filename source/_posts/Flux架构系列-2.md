---
title: Flux架构系列(2)
date: 2017-10-17 15:53:05
tags: ['Flux', 'Redux']
category: React
---

分析 flux 架构系列的结构和流程，包括 flux、redux 和 mobx，本篇文章主要分析 redux 3.7.2 版本的源码。

<!--more-->

### 背景

redux 作为当前流行的状态管理库，继承 flux 架构的思想，其实现思路也是值得研究。本文接下来分析 redux 的源码的思路和实现，该库的使用方法可参考官方网站。

### 概念

首先，熟悉下 redux 库里面的各种概念。

1. 单一 store： redux 中，整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。应用的 state 是只读的，而整个唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。
2. reducer：为了描述 action 如何改变 state tree。Reducer 只是一些纯函数，它接收先前的 state 和 action，并返回新的 state。
3. actionCreator：生成 action 对象的函数。
4. dispatch: 一个接收 action 或者异步 action的函数，该函数要么往 store 分发一个或多个 action，要么不分发任何 action。
5. Middleware：一个组合 dispatch function 的高阶函数，返回一个新的 dispatch function，通常将异步 actions 转换成 action。

### Middleware 原理

redux 默认只支持以同步方式维护 store 的功能，对于异步请求、日志打印等额外功能，采用 middleware 中间件的方式，而且这也是 redux 中的强大且难于理解之处。默认我们通过如下几种方式采用 Middleware：

```js
// with preloadedState
const store = createStore(reducer, preloadedState, applyMiddleware(...middleware));
// no preloadedState
const store = createStore(reducer, applyMiddleware(...middleware));
createStore 中关于 enhancer 的部分源码如下，执行了 enhancer 第一次执行返回的函数。

export default function createStore(reducer, preloadedState, enhancer) {
    // ...
    return enhancer(createStore)(reducer, preloadedState)
    //...
```

而 applyMiddleware 源码如下，结合上面的部分代码，我们可以看出 createStore 参数是原有的 createStore，args 参数是 [reducer, preloadedState]，最后返回的是一个包含了新的 dispatch 的 store 对象。另外，传给 middleware 中的新 dispatch 是一个包含原先 dispatch 的匿名函数。

```js
export default function applyMiddleware(...middlewares) {
    return (createStore) => (reducer, preloadedState, enhancer) => {
        const store = createStore(reducer, preloadedState, enhancer)
        let dispatch = store.dispatch
        let chain = []
        const middlewareAPI = {
            getState: store.getState,
            dispatch: (action) => dispatch(action)
        }
        chain = middlewares.map(middleware => middleware(middlewareAPI))
        dispatch = compose(...chain)(store.dispatch)
        return {
            ...store,
            dispatch
        }
    }
}
```

compose 源码如下，compose[f, g, h] 返回实际 f(g(h())) 的效果。

```js
export default function compose(...funcs) {
    //...
    return funcs.reduce((a, b) => (...args) => a(b(...args)));
}
```

下面是一个简单的 logger middleware 源码，此处 store 是上面中的 middlewareAPI，next 是 前一个 middleware 装饰后返回的 store.dispatch 或者 原始的 store.dispatch，action 参数是代码中实际 dispatch 的 action 对象。

```js
export const logger = store => next => action => {
    console.log('logger...', action);
    next(action);
}
```

至此，middle 的本质是包装了 store 的 dispatch 方法，达到实际想要的目的。多个 middleware 可以被组合到一起使用，形成 middleware 链。其中，每个 middleware 都不需要关心链中它前后的 middleware 的任何信息。下面是另一个 redux-thunk middleware 的部分源码。此时如果传入的 action 是一个函数类型，会直接打断后面的 middleware 链的调用，一般建议对于异步 action 的处理的 middleware 第一个传入，因为后面同步的 middleware 链期望传入的 action 是 plain objects。

```js
const createThunkMiddleware(extraArgument) => store => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}
```

### react-redux

redux 是一种状态管理库，如果需要与 react 结合开发，则需要用到 react-redux 库。react-redux 仅仅暴露出来两个 API，所以用法也很简单。首先，通过 Provider 包裹 react 应用的顶层容器，并将 redux 中的 store 作为 props 传入。然后在需要用到该 store 的组件上，通过 connect 连接。接下来我们分析 react-redux 5.0.6 版本的源码实现。

```js
<Provider store>
```

将接受的 props.store 转为 context 上的属性传递给子组件。

```js
class Provider extends Component {
    static childContextTypes = {
        [storeKey]: storeShape.isRequired,
        [subscriptionKey]: subscriptionShape,
    }
    getChildContext() {
        return { [storeKey]: this[storeKey], [subscriptionKey]: null }
    }
}
connect([mapStateToProps], [mapDispatchToProps], [mergeProps], [options])
```

connect 组件封住传进来的 WrappedComponent 组件，通过 store.subscribe api 监听 store 的变化。

// TODO: connect 源码

### reselect

从上面的 react-redux 中的 connect 可知，每次 dispatch 一个 action，state 改变之后都会触发 mapStateToProps，即使接受的新的 state 和之前的 state 数据一样，但是也会触发函数的执行，很明显这种方式会存在一定程度的性能浪费，尤其是当 state 树很庞大时，而 reselect 可以优化这一情况，避免很多重复的计算。

下面我们先从修改 redux 的 todos 示例入手介绍一下 reselect 的概念。

#### selector

reselect 提供了 createSelector 函数创建具有记忆功能的函数。接受一个由 input-selectors 组成的数组和一个 transform 函数作为参数。当 redux 的 state 改变时，会导致 input-selectors 的值发生改变，这时 selector 会把 input-selectors 的值传入到 transform 函数调用。如果传入的值和之前传入的参数一致，直接返回前一次的调用结果，不会再调用 transform 函数。另外，selector 每次只能缓存前一次的调用参数和结果。

```js
import { createSelector } from 'reselect'
const getVisibilityFilter = (state) => state.visibilityFilter
const getTodos = (state) => state.todos
export const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)
```

在 reselect 中，一个 selector 可以接受另一个 selector 作为 input-selectors 传入。

```js
const getKeyword = (state) => state.keyword
const getVisibleTodosFilteredByKeyword = createSelector(
  [ getVisibleTodos, getKeyword ],
  (visibleTodos, keyword) => visibleTodos.filter(
    todo => todo.text.includes(keyword)
  )
)
```

接下来，我们通过 react-redux 中 connect 方法的 mapStateToProps 参数将创建的 selector 和 redux 结合。

```js
import { getVisibleTodos } from '../selectors'
const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state)
  }
}
const VisibleTodoList = connect(
  mapStateToProps
)(TodoList)
```

#### reselect 原理

// TODO: 先分析 react-redux 中的 connect 原理

### 最后

本文简单分析 redux 源码的实践过程，针对 redux 中的中间件机制做了深入解析。顺带梳理了 react-redux 的思路。然而 redux 源码虽然简单，但真正使用起来会发现里面充斥着各种概念，改动一处需求至少改动三个文件（actionType、reducer 与 actionCreator）。而一款面向对象编程的框架 mobx 将不会存在这个问题。
