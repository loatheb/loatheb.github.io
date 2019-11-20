---
title: 100 行代码实现一个前端 JS 模块打包工具 - 1 - 模块化概览
date: '2019-11-10'
tags:
 - CommonJS
 - Bundler
 - webpack
 - fis
 - Node
---


在 WEB 开发的早期，为了团队协作和代码维护的方便，许多开发者会选择将 JavaScript 代码分开写在不同的文件里面，然后通过多个 script 标签来加载它们。

```html
<script src="./a.js"></script>
<script src="./b.js"></script>
<script src="./c.js"></script>
```

虽然每个代码块处在不同的文件中，但最终所有 JS 变量还是会处在同一个 **全局作用域** 下，这时候就需要额外注意由于作用域`变量提升`所带来的问题。

<!-- more -->

```html
<!-- index.html -->
<script>
    // a.js
    var num = 1;
    setTimeout(() => console.log(num), 1000);
</script>
<script>
    // b.js
    var num = 2;
</script>
```

在这个例子中，我们分别加载了两个 script 标签，两段 JS 都声明了 `num` 变量。第一段脚本的本意本来是希望在 1s 后打印自己声明的 `num` 变量 ** 1 **。但最终运行结果却打印了第二段脚本中的 `num` 变量的结果 ** 2 **。虽然两段代码写在不同的文件中，但是因为运行时声明变量都在全局下，最终产生了冲突。

同时，如果代码块之间有依赖关系的话，需要额外关注脚本加载的顺序。如果文件依赖顺序有改动，就需要在 html 手动变更加载标签的顺序，非常麻烦。

要解决这样的问题，我们就需要将这些脚本文件「模块化」：

1. 每个模块都要有自己的 **变量作用域**，两个模块之间的内部变量不会产生冲突。
2. 不同模块之间保留相互 **导入和导出** 的方式方法，模块间能够相互通信。模块的执行与加载遵循一定的规范，能保证彼此之间的依赖关系。

主流的编程语言都有处理模块的关键词，在这些语言中，模块与模块之间的内部变量相互不受影响。同时，也可以通过关键字进行模块定义，引入和导出等等，例如 JAVA 里的 `module` 关键词，python 中的 `import`。

但是 JavaScript 这门语言在 Ecmascript6 规范之前并没有语言层面的模块导入导出关键词及相关规范。为了解决这样的问题，不同的 JS 运行环境分别有着自己的解决方案。

## CommonJS 规范初探

Node.js 就是一个基于 V8 引擎，事件驱动 I/O 的服务端 JS 运行环境，在 2009 年刚推出时，它就实现了一套名为 **CommonJS** 的模块化规范。

在 CommonJS 规范里，每个 JS 文件就是一个 **模块(module)** ，每个模块内部可以使用 `require` 函数和 `module.exports` 对象来对模块进行导入和导出。

```js
// 一个比较简单的 CommonJS 模块
const moduleA = require("./moduleA"); // 获取相邻的相对路径 `./moduleA` 文件导出的结果
module.exports = moduleA;             // 导出当前模块内部 moduleA 的值
```

下面这三个模块稍微复杂一些，它们都是合法的 CommonJS 模块：

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

* ** index.js ** 代表的模块通过执行 `require` 函数，分别加载了相对路径为 `./moduleA` 和 `./moduleB` 的两个模块，同时输出 ** moduleB ** 模块的结果。
* ** moduleA.js ** 文件内也通过 `require` 函数加载了 ** moduleB.js ** 模块，在 1s 后也输出了加载进来的结果。
* ** moduleB.js ** 文件内部相对来说就简单的多，仅仅定义了一个时间戳，然后直接通过 `module.exports` 导出。

它们之间的 **物理关系** 和 **逻辑关系** 如下图：

![/100-lines-of-code-web-module-resolver-1/commonjs.png](/images/100-lines-of-code-web-module-resolver-1/commonjs.png)

在装有 Node.js 的机器上，我们可以直接执行 `node index.js` 查看输出的结果。我们可以发现，无论执行多少次，最终输出的两行结果均相同。

![/100-lines-of-code-web-module-resolver-1/commonjs-result.png](/images/100-lines-of-code-web-module-resolver-1/commonjs-result.png)

虽然这个例子非常简单，但是我们却可以发现 CommonJS 完美的解决了最开始我们提出的痛点：

1. 模块之间内部即使有相同的变量名，它们运行时没有冲突。**这说明它有处理模块变量作用域的能力。**上面这个例子中三个模块中均有 `m` 变量，但是并没有冲突。
2. moduleB 通过 `module.exports` 导出了一个内部变量，而它在 moduleA 和 index 模块中能被加载。**这说明它有导入导出模块的方式，同时能够处理基本的依赖关系。**
3. 我们在不同的模块加载了 moduleB 两次，我们得到了相同的结果。**这说明它保证了模块单例。**

但是，这样的 CommonJS 模块只能在 Node.js 环境中才能运行，直接在其他环境中运行这样的代码模块就会报错。这是因为只有 node 才会在解析 JS 的过程中提供一个 `require` 方法，这样当解析器执行代码时，发现有模块调用了 `require` 函数，就会通过参数找到对应模块的物理路径，通过系统调用从硬盘读取文件内容，解析这段内容最终拿到导出结果并返回。而其他运行环境并不一定会在解析时提供这么一个 `require` 方法，也就不能直接运行这样的模块了。

从它的执行过程也能看出来 CommonJS 是一个 **同步加载模块** 的模块化规范，每当一个模块 `require` 一个子模块时，都会停止当前模块的解析直到子模块读取解析并加载。

## 适合 WEB 开发的 AMD 模块化规范

另一个为 WEB 开发者所熟知的 JS 运行环境就是浏览器了。浏览器并没有提供像 Node.js 里一样的 `require` 方法。不过，受到 CommonJS 模块化规范的启发，WEB 端还是逐渐发展起来了 AMD，SystemJS 规范等适合浏览器端运行的 JS 模块化开发规范。

AMD 全称 **Asynchronous module definition**，意为`异步的模块定义`，不同于 CommonJS 规范的同步加载，AMD 正如其名所有模块默认都是异步加载，这也是早期为了满足 web 开发的需要，因为如果在 web 端也使用同步加载，那么页面在解析脚本文件的过程中可能使页面暂停响应。

而 AMD 模块的定义与 CommonJS 稍有不同，上面这个例子的三个模块分别改成 AMD 规范就类似这样：

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
define(function(require) {
    var m = new Date().getTime();
    return m;
});
```

我们可以对比看到，AMD 规范也支持文件级别的模块，模块 ID 默认为文件名，在这个模块文件中，我们需要使用 `define` 函数来定义一个模块，在回调函数中接受定义组件内容。这个回调函数接受一个 `require` 方法，能够在组件内部加载其他模块，这里我们分别传入模块 ID，就能加载对应文件内的 AMD 模块。不同于 CommonJS 的是，这个回调函数的返回值即是模块导出结果。

差异比较大的地方在于我们的入口模块，我们定义好了 moduleA 和 moduleB 之后，入口处需要加载进来它们，于是乎就需要使用 AMD 提供的 `require` 函数，第一个参数写明入口模块的依赖列表，第二个参数作为回调参数依次会传入前面依赖的导出值，所以这里我们在 index.js 中只需要在回调函数中打印 moduleB 传入的值即可。

Node.js 里我们直接通过 `node index.js` 来查看模块输出结果，在 WEB 端我们就需要使用一个 html 文件，同时在里面加载这个入口模块。这里我们再加入一个 **index.html** 作为浏览器中的启动入口。

如果想要使用 AMD 规范，我们还需要添加一个符合 AMD 规范的加载器脚本在页面中，符合 AMD 规范实现的库很多，比较有名的就是 **require.js**。

```html
<html>
    <!-- 此处必须加载 require.js 之类的 AMD 模块化库之后才可以继续加载模块-->
    <script src="/require.js"></script>
    <!-- 只需要加载入口模块即可 -->
    <script src="/index.js"></script>
</html>
```

使用 AMD 规范改造项目之后的关系如下图，在物理关系里多了两个文件，但是模块间的逻辑关系仍与之前相同。

![/100-lines-of-code-web-module-resolver-1/amd.png](/images/100-lines-of-code-web-module-resolver-1/amd.png)

启动静态服务之后我们打开浏览器中的控制台，无论我们刷新多少次页面，同 Node.js 的例子一样，输出的结果均相同。同时我们还能看到，虽然我们只加载了 index.js 也就是入口模块，但当使用到 moduleA 和 moduleB 的时候，浏览器就会发请求去获取对应模块的内容。

![/100-lines-of-code-web-module-resolver-1/amd-console.png](/images/100-lines-of-code-web-module-resolver-1/amd-console.png)

从结果上来看，AMD 与 CommonJS 一样，都完美的解决了上面说的 **变量作用域** 和 **依赖关系** 之类的问题。但是 AMD 这种默认异步，在回调函数中定义模块内容，相对来说使用起来就会麻烦一些。

同样的，AMD 的模块也不能直接运行在 node 端，因为内部的 `define` 函数，`require` 函数都必须配合在浏览器中加载 require.js 这类 AMD 库才能使用。

## 能同时被 CommonJS 规范和 AMD 规范加载的 UMD 模块

有时候我们写的模块需要同时运行在浏览器端和 Node.js 里面，这也就需要我们分别写一份 AMD 模块和 CommonJS 模块来运行在各自环境，这样如果每次模块内容有改动还得去两个地方分别进行更改，就比较麻烦。

```js
// 一个返回随机数的模块，浏览器使用的 AMD 模块
// math.js
define(function() {
    return function() {
        return Math.random();
    }
});

// 一个返回随机数的模块，Node.js 使用的 CommonJS 模块
module.exports = function() {
    return Math.random();
}
```

基于这样的问题，** UMD(Universal Module Definition)** 作为一种 **同构(isomorphic)** 的模块化解决方案出现，它能够让我们只需要在一个地方定义模块内容，并同时兼容 AMD 和 CommonJS 语法。

写一个 UMD 模块也非常简单，我们只需要判断一下这些模块化规范的特征值，判断出当前究竟在哪种模块化规范的环境下，然后把模块内容用检测出的模块化规范的语法导出即可。

```js
(function(self, factory) {
    if (typeof module === 'object' && typeof module.exports === 'object') {
        // 当前环境是 CommonJS 规范环境
        module.exports = factory();
    } else if (typeof define === 'function' && define.amd) {
        // 当前环境是 AMD 规范环境
        define(factory)
    } else {
        // 什么环境都不是，直接挂在全局对象上
        self.umdModule = factory();
    }
}(this, function() {
    return function() {
        return Math.random();
    }
}));
```

上面就是一种定义 UMD 模块的方式，我们可以看到首先他会检测当前加载模块的规范究竟是什么。如果 `module.exports` 在当前环境中为对象，那么肯定为 CommonJS，我们就能用 `module.exports` 导出模块内容。如果当前环境中有 `define` 函数并且 `define.amd` 为 `true`，那我们就可以使用 AMD 的 `define` 函数来定义一个模块。最后，即使没检测出来当前环境的模块化规范，我们也可以直接把模块内容挂载在全局对象上，这样也能加载到模块导出的结果。

## ESModule 规范

前面我们说到的 CommonJS 规范和 AMD 规范有这么几个特点：

1. 语言上层的运行环境中实现的模块化规范，模块化规范由环境自己定义。
2. 相互之间不能共用模块。例如不能在 Node.js 运行 AMD 模块，不能直接在浏览器运行 CommonJS 模块。

在 EcmaScript 2015 也就是我们常说的 ES6 之后，JS 有了语言层面的模块化导入导出关键词与语法以及与之匹配的 ESModule 规范。使用 ESModule 规范，我们可以通过 `import` 和 `export` 两个关键词来对模块进行导入与导出。

还是之前的例子，使用 ESModule 规范和新的关键词就需要这样定义：

```js
// index.js
import './moduleA';
import m from './moduleB';
console.log(m);

// moduleA.js
import m from './moduleB';
setTimeout(()) => console.log(m), 1000);

// moduleB.js
var m = new Date().getTime();
export default m;
```

ESModule 与 CommonJS 和 AMD 最大的区别在于，ESModule 是由 JS 解释器实现，而后两者是在宿主环境中运行时实现。ESModule 导入实际上是在语法层面新增了一个语句，而 AMD 和 CommonJS 加载模块实际上是调用了 `require` 函数。

```js
// 这是一个新的语法，我们没办法兼容，如果浏览器无法解析就会报语法错误
import moduleA from "./moduleA";

// 我们只需要新增加一个 require 函数，就可以首先保证 AMD 或 CommonJS 模块不报语法错误
function require() {}
const moduleA = require("./moduleA");
```

ESModule 规范支持通过这些方式导入导出代码，具体使用哪种情况得根据如何导出来决定：
```js
import { var1, var2 } from './moduleA';
import * as vars from './moduleB';
import m from './moduleC';

export default {
    var1: 1,
    var2: 2
}

export const var1 = 1;

const obj = {
    var1,
    var2
};
export default obj;
```

这里又一个地方需要额外指出，`import {var1} from "./moduleA"` 这里的括号并不代表获取结果是个对象，虽然与 ES6 之后的对象解构语法非常相似。

```js
// 这些用法都是错误的，这里不能使用对象默认值，对象 key 为变量这些语法
import {var1 = 1} from "./moduleA"
import {[test]: a} from "./moduleA";

// 这个才是 ESModule 导入语句种正确的重命名方式
import {var1 as customVar1} from "./moduleA";

// 这些用法都是合理的，因为 CommonJS 导出的就是个对象，我们可以用操作对象的方式来操作导出结果
const {var1 = 1} = require("./moduleA");
const {[test]: var1 = a} = require("./moduleA");

// 这种用法是错误的，因为对象不能这么使用
const {var1 as customVar1} = require("./moduleA");
```

用一张图来表示各种模块规范语法和它们所处环境之间的关系：

![/100-lines-of-code-web-module-resolver-1/env.png](/images/100-lines-of-code-web-module-resolver-1/env.png)

每个 JS 的运行环境都有一个解析器，否则这个环境也不会认识 JS 语法。它的作用就是用 ECMAScript 的规范去解释 JS 语法，也就是处理和执行语言本身的内容，例如按照逻辑正确执行 `var a = "123";`，`function func() {console.log("hahaha");}` 之类的内容。

在解析器的上层，每个运行环境都会在解释器的基础上封装一些环境相关的 API。例如 Node.js 中的 `global` 对象、`process` 对象，浏览器中的 `window` 对象，`document` 对象等等。这些运行环境的 API 受到各自规范的影响，例如浏览器端的 W3C 规范，它们规定了 `window` 对象和 `document` 对象上的 API 内容，以使得我们能让 `document.getElementById` 这样的 API 在所有浏览器上运行正常。

<div class="tip">
<p>事实上，类似于 `setTimeout` 和 `console` 这样的 API，大部分也不是 JS Core 层面的，只不过是所有运行环境实现了相似的结果。</p>

<p>`setTimeout` 在 ES7 规范之后才进入 JS Core 层面，在这之前都是浏览器和 Node.js 等环境进行实现。</p>

<p>`console` 类似 `promise`，有自己的规范，但实际上也是环境自己进行实现的，这也就是为什么 Node.js 的 `console.log` 是异步的而浏览器是同步的一个原因。同时，早期的 Node.js 版本是可以使用 `sys.puts` 来代替 `console.log` 来输出至 stdout 的。</p>
</div>

ESModule 就属于 JS Core 层面的规范，而 AMD，CommonJS 是运行环境的规范。所以，想要使运行环境支持 ESModule 其实是比较简单的，只需要升级自己环境中的 JS Core 解释引擎到足够的版本，引擎层面就能认识这种语法，从而不认为这是个 **语法错误(syntax error)** ，运行环境中只需要做一些兼容工作即可。

Node.js 在 V12 版本之后才可以使用 ESModule 规范的模块，在 V12 没进入 LTS 之前，我们需要加上 `--experimental-modules` 的 flag 才能使用这样的特性，也就是通过 `node --experimental-modules index.js` 来执行。浏览器端 Chrome 61 之后的版本可以开启支持 ESModule 的选项，只需要通过 `<script type="module"></script>` 这样的标签加载即可。

这也就是说，如果想在 Node.js 环境中使用 ESModule，就需要升级 Node.js 到高版本，这相对来说比较容易，毕竟服务端 Node.js 版本控制在开发人员自己手中。但浏览器端具有分布式的特点，是否能使用这种高版本特性取决于用户访问时的版本，而且这种解释器语法层面的内容无法像 AMD 那样在运行时进行兼容，所以想要直接使用就会比较麻烦。

## 后模块化时代

通过前面的分析我们可以看出来，使用 ESModule 的模块明显更符合 JS 开发的历史进程，因为任何一个支持 JS 的环境，随着对应解释器的升级，最终一定会支持 ESModule 的标准。但是，WEB 端受制于用户使用的浏览器版本，我们并不能随心所欲的随时使用 JS 的最新特性。为了能让我们的新代码也运行在用户的老浏览器中，社区涌现出了越来越多的工具，它们能静态将高版本规范的代码编译为低版本规范的代码，最为大家所熟知的就是 `babel`。

它把 JS Core 中高版本规范的语法，也能按照相同语义在静态阶段转化为低版本规范的语法，这样即使是早期的浏览器，它们内置的 JS 解释器也能看懂。

![/100-lines-of-code-web-module-resolver-1/babel.png](/images/100-lines-of-code-web-module-resolver-1/babel.png)

然后，不幸的是，对于模块化相关的 `import` 和 `export` 关键字，`babel` 最终会将它编译为包含 `require` 和 `exports` 的 CommonJS 规范。[点击连接在线查看编译结果](https://babeljs.io/repl#?babili=false&browsers=&build=&builtIns=false&spec=false&loose=false&code_lz=JYWwDg9gTgLgBAIgHQHoCGA7CMAWBTKAWQgBMBXAGzwQG4AoUSWONOAMyghEVRFMrwBBWnTwAPJvBJ42aSvDQ0gA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=es2015%2Ces2016%2Ces2017%2Cstage-0%2Cstage-1%2Cstage-2%2Cstage-3%2Ces2015-loose&prettier=false&targets=&version=7.7.3&externalPlugins=)

![/100-lines-of-code-web-module-resolver-1/babel-esmodule.png](/images/100-lines-of-code-web-module-resolver-1/babel-esmodule.png)

这就造成了另一个问题，这样带有模块化关键词的模块，编译之后还是没办法直接运行在浏览器中，因为浏览器端并不能运行 CommonJS 的模块。为了能在 WEB 端直接使用 CommonJS 规范的模块，除了编译之外，我们还需要一个步骤叫做**打包(bundle)**。

打包工具的作用，就是将模块化内部实现的细节抹平，无论是 AMD 还是 CommonJS 模块化规范的模块，经过打包处理之后能变成能直接运行在 WEB 或 Node.js 的内容。

社区有非常多优秀的打包工具，但我写这个系列文章的目的，就是自己实现这么一个简单的能打包模块的工具，跟读者分享一下主要思路和设计。这个小工具的主要目标是要实现：
* 能在 WEB 端使用 CommonJS 模块
* 能同时支持 **同步加载(synchronous import)** 和 **异步加载(dynamic import)**

这是 **[100 行代码实现一个前端 JS 模块打包工具]** 这个系列的第一篇文章，主要先阐明模块化的发展、模块化规范的区别以及为什么我们需要打包工具。

下一篇文章开始进入正题，主要介绍打包工具运行时代码相关的思考。

同时本系列介绍的所有代码都开源在自己写的 github 项目[100-lines-of-code-challenge-js](https://github.com/loatheb/100-lines-of-code-challenge-js)当中。

