---
title: 100 行代码实现一个前端 JS 模块打包工具 - 2 - 运行时代码思考
date: '2019-11-16'
tags:
 - CommonJS
 - Bundler
 - webpack
 - fis
 - Node
---

这是 **[100 行代码实现一个前端 JS 模块打包工具]** 这个系列的第二篇文章，如果您还没有看过第一篇「模块化概览」，可以点击链接查看[100 行代码实现一个前端 JS 模块打包工具 - 2 - 运行时代码思考](/posts/100-lines-of-code-web-module-resolver-1/)。

经过上篇文章的分析，我们知道了为什么我们必须使用一个 **打包(bundle)工具** 在 WEB 端来处理我们的项目：
* 社区有非常多的优秀模块，但它们的规范不一，有 AMD，有 CommonJS，项目还得分情况处理模块化信息。
* 使用 ESModule 规范的模块，即使通过了 **babel** 编译，但最终成了 CommonJS 模块，还是无法直接应用于项目中。

所以打包工具的出现，就是为了处理这些问题。使用打包工具处理之后，无论你引入了什么模块，无论你的模块使用什么规范书写，最终都能变成一个适合于浏览器使用的 JS。

这篇文章全篇还会拿之前的三个模块举例，这三个模块也涵盖了基本的模块化的必要条件。

```js
// index.js
require("./moduleA");
var m = require("./moduleB");
console.log(m);

// moduleA.js
var m = require("./moduleB");
setTimeout(() => console.log(m), 1000);

// moduleB.js
var m = new Date().getTime();
module.exports = m;
```

它们之间的物理关系和逻辑关系如图：

![/100-lines-of-code-web-module-resolver-1/commonjs.png](/images/100-lines-of-code-web-module-resolver-1/commonjs.png)

## 打包为 AMD 规范的模块

第一种思路，当然就是把他们都想办法转化为 AMD 的模块了，既然浏览器支持 AMD 规范，那我们就想办法直接转化这些模块，浏览器就可以直接运行了。

前面我们也说过，这三个模块的 AMD 形式如下，变化比较大的就是入口模块 `index.js`。

```js
// index.js
require(['moduleA', 'moduleB'], function(moduleA, moduleB) {
    console.log(moduleB);
});

// moduleA.js
define(function(require) {
    var m = require('moduleB');
    setTimeout(() => console.log(m), 1000);
});

// moduleB.js
define(function(require, exports, module) {
    var m = new Date().getTime();
    module.exports = m;
});
```

我们可以看到， moduleA 和 moduleB 仅仅是外层包裹了在 define 接受的回调函数里面，但是入口模块得使用 require 显式声明依赖，同时加载进来的结果放到了回调函数参数中，这个看起来在静态阶段比较难处理。

事实上我们可以自定义一个入口模块，把 index.js 也作为一个子依赖进行加载，所有模块的静态处理就会变得统一一些，我们就不需要单独思考如何处理入口的部分。

```js
require(['index']);

// index.js
define(function(require, exports, module) {
    require('moduleA');
    var m = require('moduleB');
    console.log(m);
});

// moduleA.js 和 moduleB.js 照旧
```

我们总结一下「将 CommonJS 模块转化为 AMD 模块」这种方式的打包结果。
* 我们要把模块进行处理，每个 CommonJS 模块