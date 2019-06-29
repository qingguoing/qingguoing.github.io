### 背景

日常撸码过程中，我们不免会遇到类似这种报错，`Uncaught TypeError: Cannot read property 'a' of null`，而出错的原因很明确，我们错误地从某个值为 `null` 或者 `undefined` 的变量读取属性 `a`。数据的来源也多重多样，例如无意中声明的变量，或者接口返回的字段。

### 现有解决方案

常见的避免上述问题的解决方案主要有两种：

1. `&&` 运算符过滤：`const c = test && test.a && test.a.b;`
2. 带有默认值的对象解构方式：`const { a: { b : { c } = {} } = {} } = test || {};`

上面方案的弊端都很明显，对于可能为 `null` 或 `undefined` 变量的每一次读取都需要做个兜底。

### babel-plugin-pipe

一款自动进行兜底处理的 `babel` 插件，完全可以假设将要读取属性的变量的值不会是 `null` 或 `undefined`。**前提是结合 ES6 的对象解构语法**。

插件利用 babel 插件的 `VariableDeclaration` 类型，匹配文件中的对象解构方式的变量声明语句，再结合运算符 `||`，对解构代码作如下编译转换：

```js
const { a : { b: { c = 'test' } } } = test;
```

编译后

```js
const { a } = test || {};
```

虽然插件解决了从 `null` 或者 `undefined` 上取值的问题，但对于`叶子节点`为 `null` 的情况却无法优化。稍有不慎，可能出现下面这种低级错误：


![](https://user-gold-cdn.xitu.io/2019/6/27/16b9963ca1d41364?w=385&h=191&f=jpeg&s=9524)

### @babel/plugin-proposal-pipeline-operator

而很多时候，我们需要对上面的 null 进行兜底，或者对某些字段进行格式化。通常写法如下：

```js
let { num } = test;
num = filterA(num);
```

现在，我们的插件可以与 [@babel/plugin-proposal-pipeline-operator](https://babeljs.io/docs/en/babel-plugin-proposal-pipeline-operator) 插件配合使用，利用 `pipeline-operator` 语法做过滤转换处理。下面是采用了 `pipe` 插件后的写法：

```js
const { num = 0 |> filterA } = test;
```

出此之外，支持过滤函数的串联调用执行：

```js
// 假设 filterA/filterB/filterC 函数已经声明
const { num = 0 |> filterA |> filterB |> filterC } = test;
```

> 当前仅支持与 `minimal proposal` 配合使用。

ps: 更多的 pipeline-operator 语法请参考 [babel 官网](https://babeljs.io/docs/en/babel-plugin-proposal-pipeline-operator)

### 如何使用

1. 项目目录下安装插件：

```js
npm i -D babel-plugin-pipe
```

2. `.babelrc` 文件内开启该插件。为了防止其它插件对于当前插件匹配格式的影响，请把插件置为 `plugins` 数组的第一个。

3. 在需要开启插件转换文件的顶部增加 `@pipe` 的注释。

```js
// @pipe
const { a } = test;
```

最后，该插件已经在笔者负责的几个线上环境页面上验证过了，欢迎尝鲜使用。如果你觉得好用，欢迎 star :)

参考资料：

1. [AST AST Explorer](https://astexplorer.net/)
2. [babel-handbook](https://github.com/jamiebuilds/babel-handbook)
3. [@babel/plugin-proposal-pipeline-operator](https://babeljs.io/docs/en/babel-plugin-proposal-pipeline-operator)