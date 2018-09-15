---
title: 使 node 也支持从 url 加载一个 module
date: '2018-6-3'
tags:
 - CommonJS
 - Node
---

最近两天 ry 的新项目 deno 火了一把。作为 node 项目的发起人，如今基于 go 重新写了一个服务端 JS 上下文（后来又改成了 rust = =），同时项目名 deno 也是 "n", "o", "d", "e" 四个字母更换了一下顺序，引发了大家的强烈关注以及联想。

与 node 相比，deno 项目在 readme 的一开始就列举出了这个项目的优势和需要解决的问题。里面最让人瞩目的就是所有模块原生支持 ts ，同时也必须从 url 来加载一个模块，这也是与现有的 node.js 里的 CommonJS 模块化最大的不同。

细细品味一下，deno 的模块化与 CommonJS 相比，更多的是一些运行时(runtime)处理的能力。比如运行时处理 ts 的过程，deno 底层的 JS 解释器依旧选择了 V8 引擎，而 V8 引擎并不支持解析 ts，所以 deno 内部也是在获取 ts 文件之后动态转化为 js 文件，而从 url 加载模块就更加动态化。这两点都是目前 node CommonJS 模块所不具备的。

现有的 CommonJS 底层实现过程也并不是静态化，但是却迟迟没有加入这些特性，需要用一些其他工具才能达到效果。正是因为受到 deno 这些特性的启发，所以我花了一天时间写了个小巧的库，从上层入手使用 CommonJS 来支持从 url 加载模块，同时写下这篇文章也简单介绍一下 CommonJS 的实现细节。

<!-- more -->

### CommonJS 的执行过程

想要让 CommonJS 支持 url 访问或者原生加载 ts 模块，必须从 CommonJS 的执行过程中入手，在中间阶段将模块注入进去。而 CommonJS 的执行过程其实总结起来很简单，大概分为以下几点：

* 处理路径依赖

处理路径依赖应该也是所有模块化加载规范的第一步，换言之就是根据路径找到文件的位置。无论是 CommonJS 的 require 还是 ESModule 的 import，无论是相对路径还是绝对路径，都必须首先在内部对这个路径进行处理，找到合适的文件地址。

<div class="tip">
模块路径有可能是绝对路径，有可能是相对路径，有可能省略了后缀(js、node、json)，有可能省略了文件名(index)，甚至是动态路径(运行时基于变量的动态拼接)等等。
</div>

首先就是遵守约定，同时按照一定的策略找到这个文件的真实位置，中间的过程就是补齐上面模块化省略的东西。一般都是根据 CommonJS 的这张流程图

![commonjs](/images/commonjs-module/commonjs.png)

* 加载文件

确认了路径并且确保了文件存在之后，加载文件这一步就简单粗暴的多。最简单的方式就是直接读取硬盘上的文件，将纯文本的模块源代码读取至内存。

* 拼接函数

在上一步中获取到的只是代码的文本形式源文件，并不具有执行能力。在接下来的步骤中需要将它变为一个可执行的代码段。

<div class="tip">
如果有同学看过 webpack 打包出来的结果，可以发现有这么一个现象，所有模块化的内容都处在一个函数的闭包中，内部所有的模块加载函数都替换成了 `__webpack_require__` 这类的 webpack 内部变量。
</div>

还有一个问题，在 CommonJS 模块化规范中我们或多或少在每个文件中会写 module, require 等等这样的「字眼」，module 和 require 并不能称为关键字，JS 中关于模块加载方面的关键字只有 ESModule 中 import 和 export 等等相关的内容。在日常的模块书写过程中，module 对象和 require 函数完全是 node 在包解析时注入进去的（类似上面的 `__webpack_require__`）

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

上面这个示例中，`func` 就已经是经过 `vm` 从字符串变为可执行代码段的结果，我们的 txt 给定的是一个函数，所以此时我们需要调用这个函数来最后完成模块的导出。

```js
var m = {
  exports: {}
};
func(null, m, m.exports);
```

这样的话，内部导出的内容就会被外面全局对象 `m` 所截获，将每一个模块导出的结果缓存到全局的 `m` 对象上面来。

而对于 require 函数来讲，注入时我们需要考虑的就是走完上面的几个步骤，require 接受一个字符串变量路径，然后依次通过路径找到文件，获取文件，拼接函数，变为可执行代码段并执行，之后仍给全局的缓存对象，这就是 「require」需要做的内容。

### 过程中的切面


* 最终形态是什么

对于最终的形态，本质上我们是要提供一个 require 函数，它的目标就是在 runtime 能够从远端 url 加载 js 模块，能够加载 ts 模块甚至类似 babel 提供 preset 加载各种各样的模块。

但是我们的 require 无法注入到 node bootstrap 阶段，所以最终结果一定得是 bootsrap 文件使用 CommonJS 模块加载，通过我们自定义的 require 加载的所有文件都能实现功能。

* 生命周期的设计

就如上面的第二部分介绍的那样，对于 require 函数我们要依次做这些事情，完全可以把每个阶段看做一个切面，任何一个阶段只关注输入和输出而不关注上个阶段是如何产出的。

经过仔细的思考，最终设置了两个核心的过程，**包裹模块内容** 和 **编译文件结果**。

包裹模块内容就是将字符串的文件结果包裹一下函数，专注于处理字符串结果，将普通文件的文本进行包裹。

编译文件结果这一步就是将代码结果编译成 node 能够直接识别的 js 而使得下一步沙盒环境进行执行，每次通过文件结果动态在内存进行编译，从而使得下一步 js 的执行。

* 同步还是异步？

这个问题其实困扰了很久。最大的问题就是里面涉及了部分异步加载的问题，按照传统前端的做法，这里一般都是使用 callback 或者 promise（async／await) 的方式，但这样就会带来一个很大的问题。

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

说实话这种方式也显得很愚蠢。不过中间我想了个方法，包裹函数时多包一层，包一个 IIFE 然后自执行一个 async 的 wrapper，不过这样的话 bootstrap 文件就必须还得手动包裹在 async 的函数中，子函数的问题解决了但是上层没有解决，不够完美。

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
