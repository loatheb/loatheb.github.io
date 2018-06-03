---
title: 使 node 也支持从 url 加载一个 module
date: '2018-6-3'
tags:
 - CommonJS
 - Node
---

最近两天 ry 大神的 deno 火了一把。作为 node 项目的发起人，现在又重新写了一个类似 node 的项目，引发了大家的强烈关注。在 deno 项目 readme 的开始就列举出了这个项目的优势和需要解决的问题，里面最让我瞩目的就是原生支持 ts ，同时也能也是必须从 url 加载模块，确实让人感到新意。

周末在家仔细思考了一下，即使是使用 node 原生的 CommonJS 模块规范，想要直接从 url 加载或者加载 ts 模块也并不复杂。所以周六在家准备撸个小项目，算是仿照 deno 让原生 node 也支持这些特性。

<!-- more -->

### CommonJS 的执行过程

想要让 CommonJS 支持 url 访问，必须从 CommonJS 的执行过程中入手，在中间阶段将模块注入进去。而 CommonJS 的执行过程其实总结起来很简单，大概分为以下几点：

* 处理路径依赖

处理路径依赖应该也是所有模块化加载规范的第一步，也就是根据路径找到文件的位置。路径有可能是绝对路径，有可能是相对路径，有可能省略了后缀（js、node、json），有可能省略了文件名（index) ，甚至是动态路径（runtime 拼接）等等。

首先就是遵守约定，同时按照一定的策略找到这个文件的真实位置，并且确保这个文件确确实实的躺在这个位置的「硬盘」上，模块加载工具才能进一步的向下执行。这中间的过程简单来说就是补齐上面模块化省略的东西，如果参数省略了后缀则依次按照 js json node 的顺序查找文件，如果参数是个路径则依次寻找 index 或是 package.json 中的 main 字段指向的文件，如果是动态拼接出来的路径则需要看看这里拼接出的路径是否有错误，总之这一步一定要有确定的模块路径出来。

![commonjs](/images/commonjs-module/commonjs.png)

* 加载文件

加载文件这一步就简单粗暴的多，最简单的方式就是以 utf-8 的方式直接读取硬盘上的文件，获取真正意义上纯文本的源代码。

* 拼接函数

在上一步中获取到的只是代码的文本形式源文件，文本形式的文件。在接下来的步骤中需要将它变为一个可执行的代码段。

这里插一句小 tips，如果有同学看过 webpack 打包出来的结果，可以发现有这么一个现象，每个文件的内容都处在一个函数的闭包中，而给每个这样的函数传入的参数都是相同的，都是 `__webpack_require__` 这样的东西，内部所有的 require 都被替换成了这个 webpack 的 require。

还有一个问题，在 CommonJS 模块化规范中我们或多或少在每个文件中会写 module, module.exports require 等等这样的「字眼」，因为这里的 module 和 require 讲道理并不能称为关键字，JS 中关于模块加载方面的关键字只有 ESModule 中 import 和 export 等等相关的内容，他们是真真正正的关键字。而这里 CommonJS 里面带来的 module 和 require 则完全算是自己实现的一种 hack，在日常的 CommonJS 模块书写过程中，module 对象和 require 函数完全是 node 在包解析时注入进去的（类似上面的 `__webpack_require__`）

这也就给了我们极大的想象空间，我们也完全可以将上面拿到的 module 进行包裹然后注入我们传递的每一个变量。简单的例子：

```text
// 纯文本代码 无法执行
var str = 1;
console.log(str);
```

将函数进行拼接，结果依旧是一个纯文本代码。但是已经可以给这个文件内部注入 require module 等变量，只需后续将它变为可执行文件并执行，就能把模块取出来。

```js
function(require, module, exports, __dirname, __filename) {
  // 纯文本代码
  var str = 1;
  console.log(str);
}
```

* 转化为可执行代码

拼接完成之后我们拿到的是还是纯字符串的代码，接下来就需要将这个字符串变成真正的代码，也就是将字符串变为可执行代码片段，这种操作在 JS 的历史上一直是危险的代名词...一直以来也有多种方法可以使用，`eval`、`new Function(str)` 等等。而在 node 环境中可以直接使用原生提供的 vm 模块，内部的沙盒环境支持我们手动注入一些变量，相对来说安全性还有所保证。

```js
var txt = "function(require, module, exports, __dirname, __filename) {
  module.exports = 1;
}"

var vm = require('vm');
var script = new vm.Script(txt);
var func = script.runInThisContext();
```

上面这个示例中，func 就已经是经过 vm 从字符串变为可执行代码段的结果，我们的 txt 给定的是一个函数，所以此时我们需要调用这个函数来最后完成模块的导出。

```js
var m = {
  exports: {}
};
func(null, m, m.exports);
```

这样的话，内部导出的 1 就会被外面全局对象 m 所截获，将每一个模块导出的结果缓存到全局的 m 对象上面来，不过按照 CommonJS 的规范这里应该有个 cache 的过程。而对于 require 函数来讲，注入时我们需要考虑的就是走完上面的几个步骤，require 接受一个字符串变量路径，然后依次通过路径找到文件，获取文件，拼接函数，变为可执行代码段并执行，之后仍给全局的缓存的 module 对象，这就是 require 需要做的内容。

### 面向切面思考

如果你真的能清清楚楚明白上面说的所有过程，那么我相信对于你来说将任何代码变成 CommonJS 模块都不是问题。接下来就是我做这个项目的一点思考：

* 最终形态是什么？

对于最终的形态，本质上我们是要提供一个 require 函数，它的目标就是在 runtime 能够从远端 url 加载 js 模块，能够加载 ts 模块甚至类似 babel 提供 preset 加载各种各样的模块。

但是我们的 require 无法注入（至少目前我认为）到 node bootstrap 阶段，所以最终结果一定得是 bootsrap 文件使用 CommonJS 模块加载，通过我们自定义的 require 加载的所有文件都能实现功能。

* 生命周期的设计

就如上面的第二部分介绍的那样，对于 require 函数我们要依次做这些事情，完全可以把每个阶段看做一个切面，符合可扩展原则，任何一个阶段只关注输入和输出而不关注上个阶段是如何产出的。

经过仔细的思考，最终设置了两个核心的切面，**包裹模块内容**和**编译文件结果**。这么说可能有点形而上学的味道，用更通俗的话将就是：

包裹模块内容就是将字符串的文件结果包裹一下函数，这一步完全不需要关注前一步是怎么将这个字符串吐给我的，无论前一步是通过 url 获取下来的还是通过相对路径绝对路径从硬盘上面拿的，这一步我的输入一定要在内存中获取这个模块的所有内容。日后如果进行扩展，假如要支持通过内存甚至 API 转化模块，我也只需要给这个切面给正常的字符串就行。

编译文件结果这一步就是将代码结果编译成 node 能够直接识别的 js ，无论给什么内容 ts 也好，还是原生 js 也罢，给我的内容我通过一些标识符可以动态加载对应的编译工具在内存中进行编译，将编译好的结果吐给下一步，而下一步则不需要关注这些代码是如何来的，只需要清楚明白这里传入的参数已经是完全可以放进 vm 执行的代码段就好。

这样的两个生命周期确定之后，从 url 加载模块和执行 ts 模块就分散到了两个生命周期中，第一个生命周期主要工作就是拿到内容，从 url 加载就在这里面，主要是获取。第二个生命周期主打编译，无论是 js ，ts 还是什么内容经过这一步一定能被原生 node 执行。

日后如果需要增加通过 API 返回的结果动态编译模块，只需要在第一个生命周期增加代码，返回增加结果。

日后如果需要增加 runtime 执行 coffee 代码，只需要在第二阶段利用 CommonJS 动态加载的特性加载 coffee 的编译工具。

这样做的好处还有一个就是不正交，每一步完全做自己的事情。这也就使得从远端 url 加载一个 ts 文件执行变得可能。

* 同步还是异步？

这个问题其实困扰了我几个小时，代码也因为这个改了两版。最大的问题就是里面涉及了通过 url 加载一个文件，按照传统前端的做法，这里一般都是使用 callback 或者 promise（async／await) 的方式，但这样就会带来一个很大的问题。

如果是 callback 的方式，那么意味着最终我的 require 可能得这样调用：

```js
var r = require("nedo");
var moduleA = r("./moduleA");
var moduleB = r("./moduleB");

function log(module) {
  // 所有执行过程作为 callback
  // 这里拿到 module 的结果
  console.log(module);
}

moduleA(log); // 传入 callback，moduleA 加载结束执行回调
moduleB(log); // 传入 callback，moduleB 加载结束执行回调
```

这样就显得很愚蠢，即使改成 AMD 那样的 callback 调用也感觉是在开历史的倒车。

如果是 promise（async/await) 这样的异步方式，那么意味着最终我的 require 可能得这样调用：

```js
var r = require("nedo");
var moduleA = r("./moduleA");

moduleA.then(module => {
  // 这里拿到 module 结果
});

(async function() {
  var moduleB = await r("./moduleB");
  // 这里拿到 module 的结果
})();
```

说实话这种方式也很愚蠢。不过中间我想了个方法，包裹函数时多包一层，包一个 IIFE 然后自执行一个 async 的 wrapper，不过这样的话 bootstrap 文件就必须还得手动包裹在 async 的函数中，子函数的问题解决了但是上层没有解决，不够完美。

其实后来仔细的思考了一下，造成这样的问题的原因究其根本是因为 request 是 async 的，这就导致了后续的代码必须以 async 的方式出现。如果我们想要从硬盘读取一个文件，那么我们可以使用 promise 包裹的 fs.readFile，当然我们也可以使用 fs.readFileSync 。前者的方法会让后续的所有调用都变成异步，而后者的代码还是同步，虽然性能很差但是完全符合直觉。

所以就必须找到一个 sync 的 request 的形式，才能让最终调用变的完美，最终的想法结果应该如下：

```js
var r = require("nedo");
var moduleA = r("./moduleA");
// moduleA 结果

var moduleB = r("https://baidu.com");
// moduleB 结果，同步阻塞
```

思考了半天不知道 sync 的 request 应该怎么写，后来只得求助万能的 npmjs，结果真的发现了一个 `sync-request` 的包，仔细研究了一下代码发现核心是借助了 `sync-rpc` 这个包，虽然这个包 github 只有 5 个 star，下载量也不大。但是感觉却是非常的厉害，能够将任何异步的代码转化为同步调用的形式，战略性 star，日后可能大有所为...

![sync-rpc](/images/commonjs-module/sync-rpc.png)

* runtime 编译

解决了 request async 的问题之后其他问题都变的非常简单，ts 使用 babel + ts preset 在内存中完成了编译，如果想要增加任何文件的支持，只需要在 lib/compile 下加入对应的文件后缀即可，在内存中只要能够完成编译就能够最终保证代码结果。

* top level await

在之前的过程中我们只是包了一层注入参数的函数进去，当然也可以上层包裹一层 async 函数，这样就可以在使用 nedo require 的包内部直接使用顶层 await，不需要再使用 async 进行包裹

### 最终结果

最后经过几个小时的不懈努力，最终能够将 hello world 跑起来了，代码还处于 pre-pre-pre-prototype 的阶段。仓库地址 [nedo](https://github.com/loatheb/nedo) ，希望大家多帮忙 review，提供更多建设性的意见...
