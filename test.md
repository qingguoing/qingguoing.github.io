

babel-plugin-pipe

<!--more-->

### 背景

日常撸码过程中，我们不免会遇到类似这种报错，`Uncaught TypeError: Cannot read property 'a' of null`，而出错的原因很明确，我们错误地从某个值为 `null` 或者 `undefined` 的变量读取属性 `a`。数据的来源也多重多样，例如无意中声明的变量，或者接口返回的字段。

### 解决方案

常见的避免上述问题的解决方案主要有两种：

1. `&&` 运算符过滤：`const c = test && test.a && test.a.b;`
2. 带有默认值的对象解构方式：`const { a: { b : { c } = {} } = {} } = test || {};`

上面方案的弊端都很明显，对于可能为 `null` 或 `undefined` 变量的每一次读取都需要做个兜底。

### babel-plugin-pipe

一款自动进行兜底处理的 `babel` 插件，完全可以假设将要读取属性的变量的值不会是 `null` 或 `undefined`。**前提是结合 ES6 的对象解构语法**。

插件利用或运算符 `||`，对解构代码作如下编译转换：

```js
const { a : { b: { c = 'test' } } } = test;
```

编译后

```js
const { a } = test || {};
```

插件虽然解决了从 `null` 或者 `undefined` 上取值的问题，但对于`叶子节点`为 `null` 的情况却无法优化。

### @babel/plugin-proposal-pipeline-operator

插件可以与 [@babel/plugin-proposal-pipeline-operator] 插件配合使用，利用 `pipeline-operator` 语法做过滤转换处理。

### 如何使用

最后，如果你觉得好用，欢迎 star :)