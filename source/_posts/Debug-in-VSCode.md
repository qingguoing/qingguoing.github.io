---
title: Debug in VSCode
date: 2017-10-30 15:43:29
tags: ['vscode', 'tools']
category: Tools
---

vscode 中的 debugger 功能介绍，主要是前端开发日常中的 debug 需求和调试 Node 中尚未完全支持的 EcmaScript 语法。

<!--more-->

### 背景

最近想用 vscode 中的 debug 代码工具跟踪 ES6/ES7 的代码，发现随着 vscode 和 node 的版本不断迭代，社区中已有的 debug 介绍和配置已经不能满足当前使用，私下就随便折腾了下，折腾经验记录写下。折腾环境：

```json
os: macOS 10.12.6
vscode: 1.17.2
node: v8.7.0
```

### 基本的 Debugger

vscode 默认会采用内置的 node debug 环境，走当前项目下 package.json 文件里的 main 字段对应的 js 文件。然后点击左侧的 Debug 按钮，再点击头部的运行按钮，选择 Node.js 环境即可。文件内部即支持手写 debugger 关键字，也通过在具体的行号左边区域点击左键直接增加一个红点或者点击右键增加一个满足特定条件才会执行到 debug 的方式来增加断点。而通过鼠标点击增加的断点都可以在 vscode debug 左侧区域的 BREAKPOINTS 面板查看到。

点击运行按钮或者 F5 按键进入运行调试过程，顶部会出现类似几个 chrome devtools 的 debug 按钮，左侧区域可以查看到 VARIABLES 和 CALL STACK 等面板。

![vscode debugger](https://ws1.sinaimg.cn/large/cab372d4ly1fl42mlyrovj20lo04xwet.jpg)

另外，对于除 js 以外的其它语言的 debug 调试，需要安装对应的 debug 插件，这个在 extensions 的搜索栏搜索 debug，然后选择自己需要的安装即可。

### Node.js Debugger
除了基本的 debugger，vscode 还支持配置多种形式的调试，Debug 面板顶部存在一个设置按钮，点击即可进入设置文件 luanch.json。每次点击绿色的 Start debugging 按钮之前，可在右侧选择已经设置好的其他形式。设置文件的格式如下，直接在 configurations 字段里新增对应的配置对象即可。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch Program",
            "program": "${workspaceFolder}/app.js"
        }
    ],
    "compounds": []
}
```

node 类型 Debugger 支持 launch 和 attach 两种调试形式。下面简单介绍下各自相应的配置。

#### launch

launch 形式的调试是通过 vscode 直接运行并调试文件。上文中基本的 Debugger 默认走的也是 launch 形式。下面是直接调试当前工作目录下的 app.js 的配置，其中 ${workspaceFolder} 是当前包含 package.json 的工作目录。

```json
{
    "type": "node",
    "request": "launch",
    "name": "Launch Program",
    "program": "${workspaceFolder}/app.js"
},
```

另外一种 launch 形式的调试模式是通过 npm run debug 直接走 package.json 文件里配置的 debug 脚本。

```json
{
    "type": "node",
    "request": "launch",
    "name": "Launch via NPM",
    "runtimeExecutable": "npm",
    "runtimeArgs": [
        "run-script",
        "debug"
    ],
    "port": 9229
}
```

我们的 debug 脚本形式为，两边指定一致的调试端口。

```
"debug": "node --inspect-brk=9229 web.js"
```

#### attach

attach 形式的调试原理是通过 ip 地址、端口和协议来调试一个手动跑起的 node 文件。首先我们需要再 .vscode/launch.json 加上如下配置：

```json
{
    "type": "node",
    "request": "attach",
    "name": "Attach",
    "port": 9229,
    "protocol": "inspector"
}
```

然后点击运行按钮 debug 的运行按钮，最后在 10s 内启动一个简单的 node 脚本，前提是我们已经在 test.js 代码每一行的左侧加了断点。

```json
node --inspect test.js
// test.js 的内容
console.log(111);
console.log(222);
console.log(333);
```

如果此时发现不能一步步跟着断点调试，需要通过如下方式运行 test.js。--inspect-brk 会停留在即将运行 node 文件的第一行代码处。

```json
node --inspect-brk test.js
```

通过上面的介绍，发现可以通过这种方式来一步步调试 node 里面尚未支持的 ES6/ES7 语法。

### ES6/ES7 debug

下面是 mobx 官方文档的一个简单 demo，在对应的装饰器语句加上断点，甚至可以在 node_modules 中 mobx 对应的文件内加上断点。

```js
import { observable, computed } from "mobx";
class OrderLine {
    @observable price = 0;
    @observable amount = 1;
    @computed get total() {
        return this.price * this.amount;
    }
}
const line = new OrderLine();
console.log("price" in line); // true
console.log(line.hasOwnProperty("price")); // false，price 属性是定义在类上的，尽管值会被存储在每个实例上。
line.amount = 43;
console.log(line.hasOwnProperty("price")); // true, 现在所有的属性都定义在实例上了。
```

vscode 的 launch.json 里面的配置我们采用的还是上面中 attach 的配置。接下来需要引入 babel-cli，并且配置对应的 .babelrc 文件，以便可以直接运行当前 node 版本尚未支持的装饰器语法。最后通过如下命令运行上面的 demo 即可。

```js
babel-node --inspect src/index.js
```

### 其它插件

vscode 除了上面内置的 node debugger 插件，还有很多其他类型的调试插件，下面简单说下其中的 debugger for chrome 和 react-native tools 两个插件。

每种类型的调试的 request 都分为 launch 和 attach 两种方式。对于两种插件的使用，详情可参考上面文档链接查看。而针对浏览器端的 js 调试，还是建议直接使用 Chrome dev tools 自带的 source 调试工具，毕竟这个不需要设置相应的配置文件，同时大家也最熟悉。对于 server 端的调试，推荐采用上文中 Node.js Debugger 部分介绍的方式。

参考链接：

1. https://code.visualstudio.com/docs/nodejs/nodejs-debugging