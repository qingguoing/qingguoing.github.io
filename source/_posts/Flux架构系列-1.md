---
title: Flux架构系列(1)
date: 2017-10-15 15:57:55
tags: ['Flux']
category: React
---

分析 flux 架构系列的结构和流程，包括 flux、redux 和 mobx，本篇文章主要由讲述 fb 官方 flux 框架的用法。

<!--more-->

### 背景

现在再谈及 flux 框架有点过时了，但是随着 redux 和 mobx 框架的大量使用， flux 的架构依然大行其道。而谈及 flux 架构就不得不说 flux 框架。本篇文章主要通过一步步实现一个简单的 todo 讲述一下 flux 的用法。

### flux 架构实现

![flux 数据流程图](https://ws1.sinaimg.cn/large/cab372d4ly1fkkwxjmelcj21040axgm1.jpg)

flux 最大的特点就是单向数据流，相对于双向数据绑定，这种方式保证了数据流的清晰。下面是图例中的几种概念：

1. store 应用的所有数据存储部分。
2. Action 一个包含 type 属性的对象，描述将要修改 store 的行文。
3. Dispatcher 数据修改操作中心，接受 action 行为对象，返回新的 store 数据，也是唯一可以修改 store 的部分。
4. View 视图部分，本文中就是 react 组件。

接下来，我们通过 fb 官方出品的 flux 框架来简单写一个小的 todo 实例。

### View

视图部分是一个 React 无状态组件，接受 store 并渲染。

```js
function AppView(props) {
  return (
    <div>
        <AddTodo {...props} />
        <Main {...props} />
    </div>
  );
}
```

### Dispatcher

每个引用都会有一个 Dispatcher 实例，用于派发并接受所有 action 对象，操作 store 数据部分。

```js
import { Dispatcher } from 'flux';
export default new Dispatcher();
```

### Action

规定，每个 action 对象都有一个 type 属性，应用中的所有 action 类型如下：

```js
export default {
    ADD_TODO: 'ADD_TODO',
    DELETE_TODO: 'DOLETE_TODO',
};
```

所以 action 对象派发如下：

```js
export default {
    addTodo(text) {
        TodoDispatcher.dispatch({type: TodoActionTypes.ADD_TODO, text});
    },
    deleteTodo(id) {
        TodoDispatcher.dispatch({type: TodoActionTypes.DELETE_TODO, id});
    }
};
```

### Store
官方 3.0 版本通过继承 flux/utils 暴露出来的 ReduceStore 来实现 store 部分。

```js
class TodoStore extends ReduceStore {
    constructor() {
        super(TodoDispatcher);
    }
    getInitialState() {
        return [];
    }
    reduce(state, action) {
        switch (action.type) {
            case TodoActionTypes.ADD_TODO:
                if (!action.text) return state;
                // 新的 state
                return [...state, action.text];
            case TodoActionTypes.DELETE_TODO:
                state.splice(action.id, 1);
                return [...state];
            default: return state;
        }
    }
};
```

2.0 版本是通过 Dispatcher.register(reducer) 来注册 reducer，view 与 store 的结合是通过 EventEmitter 绑定与触发事件的方式，这个网上有好多例子，大家可以自行参考。

### View 与 Store 结合

到目前为止，flux 架构流程图中的几个部分我们都分别实现了，但是 view 尚未与 store 结合，即应用的视图和数据部分还没组合在一起。

```js
import {Container} from 'flux/utils';
function getStores() {
    // 必须是一个数组
    return [TodoStore];
}
function getState() {
    return {
        todos: TodoStore.getState(),
        onAdd: TodoAction.addTodo,
        onDelete: TodoAction.deleteTodo
    };
}
export default Container.createFunctional(AppView, getStores, getState);
```

### 最后

本文通过一个简单的 todo 实例来介绍 fb flux 框架，进而介绍 flux 这种结构理念。x本文中的示例借鉴了 flux 框架的 example。下篇文章将会分析 flux 理念的另一种实现，redux 框架的源码。