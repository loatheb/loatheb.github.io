---
title: 零依赖 100 行代码实现一个前端 JS 模块打包工具
date: '2019-11-10'
tags:
 - CommonJS
 - Bundler
 - webpack
 - fis
 - Node
---

在 web 开发的早期，所有浏览器的脚本代码都写在一个 `Javascript` 文件里面。后来，代码越来越多，为了维护的方便，很多人会选择将代码分开写在不同的文件里面，然后在 html 里面写多个 script 标签加载这些文件。虽然我们分开加载了这些文件，但实际上所有脚本内容还是会处在同一个 **全局作用域** 下，这时候就需要额外注意由于`变量提升`所带来的 bug。

```html
<!-- index.html -->
<html>
    <body></body>
    <script>
        var config = {
            info: "this is script1.config.info"
        };
        setTimeout(function() {
            console.log("this is called from first script tag", config);
        }, 1000);
    </script>
    <script>
        var config = "this is script2.config";
    </script>
</html>
```

上面例子中的 html 加载了两个 script 标签，两段脚本都声明了 config 变量。第一段脚本本意是想在 1 秒中后控制台打印自己定义的 `config.info` 内容，但最后却因为变量提升后第二段脚本的重新赋值，打印了第二段脚本中同名变量 config 的值。如果这两个文件由不同的人维护，那么这就是个很难以发现的 bug。

随着现代 web 应用不断庞大，我们不可能还将代码写进一个文件来规避这样的风险。那有没有方法能让写在不同文件的脚本中的变量限制在自己的文件中呢？换言之，我们需要将这些代码文件「模块化」。

<!-- more -->
我们需要模块化所解决的问题是：

1. 每个模块都要有自己的 **变量作用域**，两个模块之间的内部变量不会因为变量作用域的提升而产生冲突。
2. 不同模块之间保留相互 **导入和导出** 的方式方法，这样模块间就能相互通信。模块的执行与加载遵循一定的规范，从而保证彼此之间的依赖关系。

然而，JS 这门脚本语言在 Ecmascript5 规范之前都没有语言层面的模块导入导出关键词。为了解决这个问题，node.js 在运行时(runtime)实现了一套名为 **CommonJS** 的模块化规范。

在 CommonJS 规范里，每个 JS 文件就是一个模块(module)，每个文件独享自己的作用域，即使不同的模块声明相同的变量也不会出现问题。同时给每个模块内部可以使用 `require` 函数和 `module.exports` 对象来对模块进行导入和导出。只需要使用形如 `const moduleA = require("./moduleA");` 这样的语法，就能获取到相邻的 moduleA 文件中 `module.exports` 的结果。

在 web 端的开发者，受到 CommonJS 模块化规范的启发，逐渐发展起来了 AMD，SystemJS 规范。AMD 全称 **Asynchronous module definition**，意为`异步的模块定义`，不同于 CommonJS 规范的同步加载，AMD 正如其名所有模块默认都是异步加载，这也是早期为了满足 web 开发的需要，因为如果在 web 端使用同步加载，那么页面在解析大的脚本文件时可能使页面暂停响应。

从结果上来看，AMD 确实完美的解决了上面说的变量提升和依赖关系之间的问题，但是与 CommonJS 相比，它定义模块和引入模块实在是比较麻烦，同时，默认的异步加载也让想同步获取一部分模块变得比较难受。如果能直接在 web 端使用 CommonJS 规范，一个文件就是一个模块，那么写起来就方便的多。

抱着这样的想法，静态打包的工具链条逐渐发展起来。这其中有一类工具名为打包工具，它们的作用是提前将所有模块打包到一个文件当中。比较为大家所熟知的打包工具就是 fis 和 webpack。除了前面所说的功能，现代化的打包工具还具有更多工程化的能力，比如优化依赖之间的关系，增加文件 hash，处理 less，scss 等样式文件，通过插件可以实现各种丰富的功能。同时 webpack 也提供了 `require.ensure` 作为异步加载脚本的语法，使用 `require.ensure` 加载的模块，会自动在页面需要他时才进行加载，达到了和 AMD 一样的效果，但比使用 AMD 规范灵活的多。

今天这篇文章的目的，就是实现这么一个简单的能打包 web 模块的打包工具，能够支持同步加载(synchronous import)和异步加载(dynamic import) CommonJS 模块，算是分享一下写这个小工具的思路和想法。

同时本文介绍的所有代码都开源在自己写的 github 项目[100-lines-of-code-challenge-js](https://github.com/loatheb/100-lines-of-code-challenge-js)当中。

## 运行时代码的思考

在开始写打包器工具之前，我们首先得思考一下，我们打包出来的代码样子究竟应该是怎样的？究竟 CommonJS 代码应该以怎样的形态出现在打包结果中？

本小节主要是对**运行时文件形态**进行思考，运行时文件也就是我们的打包结果，为了保证它的兼容性，打包结果这种 **运行时代码** 均以 ES5 规范思考和书写。

这里我们先列举出几个 CommonJS 的模块，本小节就用这几个模块来做例子。我们先自己手写一下打包结果，看看为了得到这个打包结果，我们的打包器究竟要实现哪些东西。

假设我们有下面三个 CommonJS 模块，其中 index.js 作为我们的入口文件，moduleA 和 moduleB 作为两个被依赖到的模块。

```js
// index.js
require("./moduleA");
var m = require("./moduleB");
console.log("this is called from index.js", m);
```

```js
// moduleA.js
var m = require("./moduleB");
setTimeout(
    function() {
        console.log("this is called from moduleA.js", m);
    },
    1000
);
```

```js
// moduleB.js
var m = new Date().getTime();
module.exports = m;
```

我们可以看到，在物理层面上，它们都在一个文件夹下，但在逻辑依赖中，**index.js** 分别依赖 **moduleA.js** 和 **moduleB.js**，而 **moduleA.js** 继续依赖 **moduleB.js**，呈现一个`树状结构`。

![/100-lines-of-code-web-module-resolver/commonjs.png](/images/100-lines-of-code-web-module-resolver/commonjs.png)

对于这个示例来说，我们打包器的目的，就是实现以 **index.js** 作为入口文件时，生成一个 **index.bundle.js** 文件，使得浏览器直接加载这个打包结果时，能保证不报错且能输出两行 log，且最后的时间戳相同。

这几个模块虽然简单，但是却能解决之前我们提出的几个痛点：

1. 模块之间内部有相同的变量名，它们不应该有冲突：有了处理模块变量作用域的能力。
2. moduleB 通过 `module.exports` 导出了一个内部变量，而它在 moduleA 和 index 模块中被加载：有了导入导出之间的方式，同时能够处理基本的依赖关系。
3. 我们在不同的文件加载了 moduleB 两次，我们应该得到相同的结果：能够保证模块单例。

### 处理作用域

变量只所以能将作用域控制在自己当前的模块中，肯定是因为模块被打包工具处理时给当前模块加上了一个作用域限制。

而 JavaScript 这门语言中只有全局作用域和函数作用域两种作用域类型。这也就是说，我们可以分别使用这两种方式来限制模块内部的变量。

** 使用全局作用域限制模块变量作用域：** 所有模块的内容都被平行的放置在一起，那么我们就要保证最终运行的每个模块下的变量维一。我们可以在处理时给每个变量增加一个命名空间(namespace)。有了这个唯一的前缀(例如模块的绝对路径)，即使模块间有冲突的变量，也会在编译的作用下使得运行时不受影响。

** 使用函数作用域限制模块变量作用域：** 我们可以将每个模块的代码用函数包裹起来。这样模块内部变量运行时就被限制在这个函数中。这样，只需要我们执行这个函数，就有机会拿到模块内部导出的值。

![/100-lines-of-code-web-module-resolver/variable-namespace.png](/images/100-lines-of-code-web-module-resolver/variable-namespace.png)


在解决变量作用域问题的时候，两种方案都是能够解决问题的。我们可以分析两者的优劣：

- 使用全局作用域
  - 优势：打包阶段已经处理好作用域的关系，运行时没有增加任何代码。
  - 劣势：打包工具代码会比较复杂，需要分析依赖改变变量名称，即使是简单场景，AST 也是必不可少了。

- 使用函数作用域
  - 优势：运行时阶段处理，所以打包阶段工作简单，仅仅需要将原来模块代码包裹一下即可。
  - 劣势：运行时需要额外执行函数，性能上会稍微受到影响。

经过对比我们能够发现，使用函数作用域的方式，虽然运行时增加了函数的执行，但这性能差异基本可以忽略不计，同时能极大的减少打包工具的工作量，减少复杂度。所以这里我们在处理模块变量作用域时，就可以选择这种「包裹一层函数」的 **函数作用域** 方案进行处理。

事实上，主流的运行时模块化处理方案，限制作用域的手段都是使用函数作用域来限制，AMD 将模块内容放置在回掉函数中，CommonJS 也如此，每个文件在加载之后，也会被包裹一个函数后放置于 V8 的 vm 当中解析，取出导出的结果。

### 处理导入与导出

同作用域处理相似，处理 `require` 和 `module.exports` 这种导入导出语法，我们也可以有 **静态打包阶段处理** 和 **运行时处理** 两种方案。

**静态打包阶段处理：**既然我们已经确定把每个模块代码包裹在一个函数中，那么我们只需要将 `module.exports` 编译为 `return`，将 `require("./moduleA");` 编译为 `moduleA()` ，这样我们引入模块就执行了我们包裹的函数，就通过 `return` 拿到了原来导出的内容。

**运行阶段处理：**我们可以提供一个 `require` 函数和 `module.exports` 对象，在运行时内部处理模块之间的关系。代码示例如下图所示。

![/100-lines-of-code-web-module-resolver/require.png](/images/100-lines-of-code-web-module-resolver/require.png)

总结一下两种方案的优劣：

* 在静态打包阶段处理
    * 优势：运行时代码没有增减，性能相对来说没有影响
    * 劣势：静态打包阶段需要替换模块代码，打包工具执行的流程相对复杂。
* 在运行阶段处理
    * 优势：静态打包阶段处理简单，但需要注意运行时要能正确找到依赖位置，有部分工作量。
    * 劣势：运行时需要增加 `require` 函数，`module.exports` 对象，需要在运行时对应起来 `require` 函数参数和模块位置。

出于工作量和可维护性的考虑，这里还是推荐使用运行阶段处理，虽然静态打包阶段处理看起来非常完美，但是任何一个简单场景都需要使用 AST 对模块进行分析才能替换，否则极容易出现问题。使用运行时方案，就是通过牺牲部分运行时的性能，来保证这个静态打包工具足够简单且有效。

到了这里，这两段代码已经可以在浏览器端正常运行起来了，并且没有我们之前说的 **变量作用域** 的问题，模块之间也有了 **依赖关系**。

### 运行时的模块对应关系

既然选择了在运行时加入 `require` 函数，那么这个 `require` 函数就必须满足这几点：
* 能通过参数找到对应的模块内容，从而才能执行模块并加载进来。
* 这个寻找过程必须通用，而不能像上面示例那样通过 if else 的硬编码进行判断，想要生成这样的代码也不容易！！

**静态打包阶段处理：**我们可以在静态打包阶段，分析到模块之间的依赖关系，同时给模块设置 id。最终输出时直接替换 `require` 函数的参数为对应模块的 id，我们就能直接取到模块内容。

**运行时处理：**运行时的方案还是得分析，但是却不需要更改原来模块的内容，而是选择把这个模块映射的 id 关系输出到最终结果中，通过这个映射关系，运行时判断加载的模块内容。

<div class="tip">
在这里之后的例子中，我们可以把 `require`，`module`和 `module.exports` 以参数的形式传入外层包的函数内部。这样的好处就是模块内部不仅可以使用 `module.exports = "xx"` 也可以使用 `exports.something = "xx"` 这样通过对象的引用的方式来导出内容。
</div>

![/100-lines-of-code-web-module-resolver/params.png](/images/100-lines-of-code-web-module-resolver/params.png)

这里有一点要注意的是，运行时的这种方案，在执行 `require` 时，还需要知道自己是被哪个模块加载的，所以通过闭包给它传入了第二个参数，作为当前模块的父模块 id。

与之前的情景几乎一模一样，静态打包阶段的处理拥有更好的运行时性能，但是静态处理的复杂度相对来说较高，而牺牲部分性能的运行时处理，却能简单许多。

### 增加 require 缓存

最后，我们只需要给 `require` 再加上一个内存的 cache，缓存住每次导出的结果，下次再加载相同的模块 id 就先从缓存中取出。

```js
var cache = {};
var require = function require(id, parentModuleId) {
    var currentModuleId = parentModuleId !== undefined ? moduleDepMapList[parentModuleId][id] : id;
    // 先从缓存中看看有没有当前 id 的模块结果，currentModuleId 是模块列表的数组下标
    if (cache.hasOwnProperty(currentModuleId)) return cache[currentModuleId];
    var module = {exports: {}};
    var func = moduleList[currentModuleId];
    func(
        (function(parentModuleId) {
            return function(id) {
                return require(id, parentModuleId);
            }
        })(id),
        module,
        module.exports
    );
    // 执行完一次之后更新缓存
    cache[currentModuleId] = module.exports;
    return cache[currentModuleId];
};
```

### 提取公共部分代码

整理一下最终的代码，同时将这段代码块限制在函数作用域内部，这样我们定义的变量就不会与其他全局变量冲突。

```js
(function () {
    var cache = {};
    var moduleList = [
        /***************************************************************/
        function(require, module, exports) {
            require("./moduleA");
            var m = require("./moduleB");
            console.log("this is called from index.js", m);
        },
        function(require, module, exports) {
            var m = require("./moduleB");
            setTimeout(
                function() {
                    console.log("this is called from moduleA.js", m);
                },
                1000
            );
        },
        function(require, module, exports) {
            var m = new Date().getTime();
            module.exports = m;
        }
        /***************************************************************/
    ];

    var moduleDepMapList = [
        /*********************************/
        {"./moduleB": 2, "./moduleA": 1},
        {"./moduleB": 2}, 
        {}
        /*********************************/
    ];

    var require = function require(id, parentModuleId) {
        var currentModuleId = parentModuleId !== undefined ? moduleDepMapList[parentModuleId][id] : id;
        if (cache.hasOwnProperty(currentModuleId)) return cache[currentModuleId];
        var module = {exports: {}};
        var func = moduleList[currentModuleId];
        func(
            (function(parentModuleId) {
                return function(id) {
                    return require(id, parentModuleId);
                }
            })(id),
            module,
            module.exports
        );
        cache[currentModuleId] = module.exports;
        return cache[currentModuleId];
    };

    require("0"); // 这里加载哪个模块取决于入口模块最终在 moduleList 的位置
})();

```

这段代码有这样的特点：
* 所有的模块源代码包裹了一层匿名函数之后都有序的排列在 `moduleList` 这个数组中
* `moduleDepMapList` 包含每个模块内部参数所映射的子模块 id

而对于不同场景的打包结果来看，只有`moduleList` 和 `moduleDepMapList`这两个变量的结果不同(上面注释圈起来的部分)，而其他部分则完全一样，那我们可以把这部分 **动态内容** 打上标记，提取公用的 **样板代码(boilerplate)** ，通过替换的方式输出不同结果。

```js
/* bundle.boilerplate */
(function () {
    var cache = {};
    var moduleList = [
        /* module-list-template */
    ];
    var moduleDepMapList = [
        /* module-dep-map-list-template */
    ];

    var require = function require(id, parentModuleId) {
        var currentModuleId = parentModuleId !== undefined ? moduleDepMapList[parentModuleId][id] : id;
        if (cache.hasOwnProperty(currentModuleId)) return cache[currentModuleId];
        var module = {exports: {}};
        var func = moduleList[currentModuleId];
        func(
            (function(parentModuleId) {
                return function(id) {
                    return require(id, parentModuleId);
                }
            })(id),
            module,
            module.exports
        );
        cache[currentModuleId] = module.exports;
        return cache[currentModuleId];
    };

    // 这里的参数取决于最后入口模块在 moduleList 的位置
    require("0");
})();
```

上面就是最后提取出的样板代码，我们给它命名为 `bundle.boilerplate`，同时把动态的部分通过 `/* module-list-template */` 和 `/* module-dep-map-list-template */` 这两个 flag 进行标记。后续我们就可以在 **静态打包的输出阶段** 执行类似这样的代码。

```js
// 打包器脚本分析之后得到这两个变量内容
const moduleList = [];
const moduleDepMapList = [{}];

fs.writeFileSync(
    'index.bundle.js',
    fs.readFileSync(path.resolve(__dirname, 'bundle.boilerplate'), 'utf-8')
        .replace('/* module-list-template */', moduleList.join(','))
        .replace(
            '/* module-dep-map-list-template */', 
            moduleDepMapList.map(item => JSON.stringify(item)).join(',')
        ),
    'utf-8'
);
```

其中 `moduleList` 代表所有被包裹了一层匿名函数的模块源码，`moduleDepMapList` 代表当前模块依赖子模块的 id 映射信息。`moduleList` 和 `moduleDepMapList` 相同的数组下标代表相同的模块，这个数组下标也是模块的 id。

在这一小节的最后，我们总结一下 **生成运行时代码的关键步骤** 和 **打包器脚本所要做的工作**：

1. 将所有模块源码外面包裹上一层匿名函数，来保证模块内部的变量拥有自己的作用域。
2. 在运行时增加 `require` 函数和 `module.exports` 对象，使得我们组合的模块化代码能够相互导入导出。
3. 静态分析时需要得到每个模块的 `require` 参数和对应模块 id 之间的关系，在运行时通过这个关系寻找模块 id。
4. 最终我们通过替换样板代码中 `/* module-list-template */` 和 `/* module-dep-map-list-template */` 两个 flag 的形式输出打包结果。

## 编写编译脚本

上一节我们研究清楚了最终运行时的代码内容，确定了打包器需要得到模块间的依赖关系的形式。这一节主要是介绍通过何种的思路分析结构和完成打包工具脚本的写法，当然打包器脚本的执行环境取决于我们的 node 版本，写起来就没有运行时代码那么多条条框框了，可以使用任何我们想使用的语法。

### 处理传入打包工具的配置项

首先我们应该先写与模块分析无关的一些代码，先写一下处理配置的部分：

```js
const path = require('path');
const fs = require('fs');

// 获取当前项目的根目录
const root = path.dirname(require.main.paths[1]);

// 读取当前项目根目录下的 packer.config.js 或 packer.config.json 配置
const config = require(path.resolve(root, 'packer.config'));

const defaultConfig = {
    base: root,
    entry: 'index',
    output: 'index.bundle.js'
};
const bundleConfig = Object.assign({}, defaultConfig, config);

const entryAbsolutePath = path.resolve(root, bundleConfig.base, bundleConfig.entry);
// handleFunc 是我们主要的处理函数，我们假设它返回我们需要的两块内容
const [moduleList, moduleDepMapList] = handleFunc(entryAbsolutePath);

fs.writeFileSync(
    path.resolve(root, bundleConfig.base, bundleConfig.output),
    fs.readFileSync(path.resolve(__dirname, 'bundle.boilerplate'), 'utf-8')
        .replace('/* module-list-template */', moduleList.join(',')),
        .replace(
            '/* module-dep-map-list-template */', 
            moduleDepMapList.map(depMap => JSON.stringify(depMap)).join(',')
        ),
    'utf-8'
);

// 主要的处理函数，此时返回处理好的模块结果
function handleFunc(fullPath) {
    const moduleList = [
        `function() {
            console.log("hello world");
        }`
    ];
    const moduleDepMapList = [{}];
    return [moduleList, moduleDepMapList];
}
```

上面的脚本代码中，主要做了这么几件事情：

1. 读取运行脚本项目根目录下的 **packer.config.js** 配置。
2. 为传入的配置项加上一些默认的参数，支持 `base`，`entry` 和 `output` 配置，同时通过 `base` 和 `entry`/`output` 计算获取最终入口和出口的文件位置。
3. 将入口文件绝对路径传入 `handleFunc`，同时执行它，通过这个方法能分析所有模块，并返回**模块列表**和**依赖关系**的分析结果。
4. 然后读取我们之前写好的样板代码，然后替换其中预留的位置，同时将结果写入配置项指定的结果文件位置。


这里面 `base` 代表 `entry` 和 `output` 的路径前缀，如果配置了 `base`，则会默认会去 `base` + `entry` 的路径寻找入口，同时输出时也会输出到 `base` + `output` 位置去，例如下面两个配置就是等价的。

```js
    // packer.config.js
    module.exports = {
        base: "./lib",
        entry: "main.js",
        output: "index.bundle.js"
    }
```

```js
    // packer.config.json
    {
        "entry": "./lib/main.js",
        "output": "./lib/index.bundle.js"
    }
```

这里我们只支持单个对象的配置方式，而只需要将这些处理逻辑包一层函数，加个简单的判断递归后就能让它支持 **数组配置**。从仅支持 `{"entry": "index1.js"}` 变为支持 `[{"entry": "index1.js"}, {"entry": "index2.js"}]` 这样的数组配置。

这里我们将刚才所有的处理逻辑包裹在 `main` 函数中，然后入口处通过 `if (Array.isArray(config)) return config.forEach(main);` 判断来执行递归，迭代器迭代过程中把数组中的每个元素重新传进 `main` 函数中去。

```js
// ...
main(config);

function main(config) {
    // 如果是数组，则递归执行 main 函数
    if (Array.isArray(config)) return config.forEach(main);

    const defaultConfig = {
        base: root,
        entry: 'index',
        output: 'index.bundle.js'
    };
    const bundleConfig = Object.assign({}, defaultConfig, config);

    const entryAbsolutePath = path.resolve(root, bundleConfig.base, bundleConfig.entry);
    const [moduleList, moduleDepMapList] = handleFunc(entryAbsolutePath);

    fs.writeFileSync(
        path.resolve(root, bundleConfig.base, bundleConfig.output),
        fs.readFileSync(path.resolve(__dirname, 'bundle.boilerplate'), 'utf-8')
            .replace('/* module-list-template */', moduleList.join(',')),
            .replace(
                '/* module-dep-map-list-template */',
                moduleDepMapList.map(depMap => JSON.stringify(depMap)).join(',')
            ),
        'utf-8'
    );
}
```

我给这种代码起名为 `arrayify`，类似于 `promisify`，它能很轻易的通过递归使一个函数接受数组。

```js
function arrayify(func) {
    return function(arg) {
        if (Array.isArray(arg)) return arg.map(item => func(item));
        return func(arg);
    }
}
```

简单找个例子类似这样：

```js
function addOne(num) {
    return num + 1;
}

const addOneArraify = arrayify(addOne);
addOne(10); // 11
addOneArrayify(100); // 101
addOneArrayify([10, 100]); // [11, 101]
```

### 补充默认文件后缀并获取当前分析模块内容

接下来的部分我们都会用来思考 `handleFunc` 的实现，也就是整个打包工具最关键的分析模块部分代码的实现。

前面说到了，这个函数接受了一个模块的绝对路径，然后需要 **分析出这个路径模块下的所有子模块的依赖关系** 。但是这里需要注意，因为路径可能会省略文件的后缀，所以我们这里首先需要给传入的参数增加一下后缀，然后再读取这个路径处文件的内容。

```js
// 对路径参数补充后缀，如果最先匹配到哪个后缀则加载哪个后缀的文件内容
const getFilePath = modulePath => [modulePath, `${modulePath}.js`].find(fs.existsSync);

function handleFunc(fullPath) {
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');
}
```

有了这一步，即使我们传进来的是 `fullPath` 是 `/project/moduleA`，最终也能找到物理路径是 `/project/moduleA.js` 的文件。

### 分析所有子模块路径

上一小节我们获取到了模块的内容，接下来我们要获取当前模块依赖的所有子模块列表，也就是在当前模块中通过 `require` 函数所加载的模块列表，这样就得知当前模块的依赖。

上一节我们分析过了，我们设计的运行时代码，最终输出的内容 **并不需要** 改变原来模块里面的内容，**只需要得到模块间的依赖关系即可**。所以这里我们就可以不使用 AST 抽象语法树解析代码，而是直接使用正则表达式匹配当前模块内容中所有 `require` 函数的参数，就能知道当前模块究竟依赖了哪些子模块。

列举一下我们的这个正则需要面对的几种情况：

```js
const case1 = "require('./moduleA')";
const case2 = "require('./module/a')";
const case3 = "require('b')";

const case4 = "var a = `require('./moduleA')`";
const case5 = "var str = 'str'; var a = `require('./moduleA')`";
const case6 = "/* require('./moduleA') */";
const case7 = "// require('./moduleA')";
```

上面的七个示例中，我们的正则表达式需要将 `case1`，`case2`，`case3` 字符串里面 `require` 函数的参数匹配出来。
而 `case4`，`case5` 的 `require` 是在一个字符串中，`case6` 和 `case7` 的 `require` 在注释中，这四个示例的内容我们并不需要。

```js
const matcherReg = /"(?:[^\\"\r\n\f]|\\[\s\S])*"|'(?:[^\\'\n\r\f]|\\[\s\S])*'|(\/\/[^\r\n\f]+|\/\*[\s\S]*?(?:\*\/|$))|\brequire\s*\(\s*("(?:[^\\"\r\n\f]|\\[\s\S])*"|'(?:[^\\'\n\r\f]|\\[\s\S])*')\s*\)/g;
```

设置了正则表达式的 global 模式之后，我们只要一直执行 matcherReg.exec，正则内部的 lastIndex 就会一直向后匹配。所以我们写一个循环一直向后匹配。

```js

function main(config) {
    //...前面是处理 config 配置部分，具体代码见上文
    const [moduleList, moduleDepMapList] = handleFunc(
        resolve(root, bundleConfig.base, bundleConfig.entry)
    );
    //...后面是输出最终结果部分，具体代码见上文
}

// 对路径参数补充后缀，如果最先匹配到哪个后缀则加载哪个后缀
const getFilePath = modulePath => [modulePath, `${modulePath}.js`].find(fs.existsSync);

function handleFunc(fullPath) {
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');
    let match = null;
    while ((match = modulePathMatcher.exec(moduleText)) !== null) {
        const [, subModulePath] = match;
    }
}

```
只要能进入这个 while 循环，则说明匹配到了合法的 require 函数语句，我们就可以在内部取出这个 require 函数的参数，也就是当前模块需要加载的一个子模块。如果这个正则匹配不到内容了，match 被赋值为 null 的同时 while 循环也为 false，那就会跳出循环。

### 处理模块依赖关系

模块之间的依赖关系，同浏览器的 DOM 结构一样，是一个典型的树形数据结构。我们在分析时需要遍历这个树上的每一个节点，在每个节点内部找出他的子节点，然后再前往他的子节点，找出这个子节点的依赖，循环往复，当所有节点都处理完成之后，我们就拿到了模块之间的依赖关系。

树的遍历有几个经典算法，图形学中的 BFS 和 DFS 最为经典。

* BFS 全称广度优先算法(Breadth-First-Search)，它是从根节点开始，优先沿着树的`宽度`遍历树的节点。 

* DFS 全称深度优先算法(Depth-First-Search)，它也是从根节点开始，优先沿着树的`深度`遍历树的节点。

![/100-lines-of-code-web-module-resolver/iterator.png](/images/100-lines-of-code-web-module-resolver/iterator.png)

这里我们选择使用 DFS 算法，因为 DFS 比较好写递归，相比较而言更容易理解。所以我们先把之前 `handleFunc` 先改一下名字，改成 `deepTravel` 比较符合我们使用的算法。

虽然我是 immutable 的忠实粉丝，但是我不建议在递归中用返回值的形式来进行参数传递，因为这样有时候必须使用尾递归，其实是增加了递归函数编写和理解的难度。递归函数里面，还是利用引用传值的方式进行传递参数比较方便，也就是我们需要改变一下函数的返回值。(这一段都是个人想法)

主要改动的地方，就是将本来返回的 moduleList 和 moduleDepMapList 内容，变成了递归函数传递下去，内部通过引用(array.push)进行全局使用。

```js
const moduleList = [];
const moduleDepMapList = [];
const entryAbsolutePath = path.resolve(root, bundleConfig.base, bundleConfig.entry);
deepTravel(entryAbsolutePath, moduleList, moduleDepMapList);

function deepTravel(fullPath, moduleList, moduleDepMapList) {
    // 假设我在这里执行 moduleList.push("something")，因为引用的特性，外面的内容也会变，递归函数这样用其实很方便
    // 相比于 const [moduleList, moduleDepMapList] = deepTravel(entryAbsolutePath); 的设计，使用参数传递不需要过分设计和考虑尾递归的情况
}
```

既然是 DFS，那么我们第一次遇到的子孙节点，就需要递归下去了。我们在函数开始的地方和递归开始的地方分别 log 一下，看看结果输出是什么。

```js
// 注意这里我们不再返回 moduleList 和 moduleDepMapList 了，而是直接用引用改变外面的值
function deepTravel(fullPath, moduleList, moduleDepMapList) {
    console.log("current process ", fullPath);
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');

    let match = null;
    while ((match = modulePathMatcher.exec(moduleText)) !== null) {
        const [, subModulePath] = match;
        const childModuleAbsolutePath = path.resolve(path.dirname(fullPath), subModulePath);
        console.log("will from " + fullPath " go into child " + childModuleAbsolutePath);
        deepTravel(childModuleAbsolutePath, moduleList, moduleDepMapList);
    }
    console.log("will leave ")
}
```
我们跑一下这段代码，就会发现屏幕上出现这样的 log

```
current process /project/index.js
will from /project/index.js go into child /project/moduleA
current process /project/moduleA
will from /project/moduleA.js go into child /project/moduleB
current process /project/indexB
will from /project/index.js go into child /project/moduleB
current process /project/indexB.js
```

对比一下上面那张 DFS 的路线图，你就会发现这个路线和图里面的箭头完全一致。

### 使用简单情景完善递归出口

### 多叉树的后续遍历

树型数据结构中最典型的就是二叉树，二叉树是一种每个节点只有零个，一个或两个子孙的节点所构成的树。通常节点的子孙被称作「左子树」（left subtree）和「右子树」（right subtree）。

在二叉树的**深度优先遍历**算法当中，又有几种细分，分别是前序遍历 (pre-order-traversal)，中序遍历(in-order-traversal)和后续遍历(post-order-traversal)。

如果树的每个节点可以有两个以上的子孙，那么这个树就称为 n 阶的多叉树，又名 n 叉树。可以看出来，模块之间的依赖关系的数据结构，是一种标准的 n 叉树结构。

这里我们先使用递归的 DFS 算法来实现。既然是递归，那么最终的结果就可以通过引用进行层级之间的传递，虽然我是 immutable 的忠实 fans，但在递归函数里面，我不得不承认，使用引用能稍微缓解一点复杂递归函数的难度，能让递归函数变得更容易理解，同时减少一部分递归代码。

我们首先实现 DFS 算法的递归版本，我们将之前的 handleFunc 改名为 deepTravel，然后将他返回的值改成函数的参数传递下去，这样在内部我们直接通过 push 方法，直接通过引用的形式就可以该表最外层这两个变量的值。

```js
function main(config) {
    // ...

    const moduleList = [];
    const moduleDepMapList = [];

    // 把 handleFunc 名称改成了 deepTrevel，第一个参数是入口文件的绝对路径，第二个是 moduleList，第三个是依赖的映射
    deepTravel(
        resolve(root, bundleConfig.base, bundleConfig.entry), 
        moduleList,
        moduleDepMapList
    );

    // ...
}

const getFilePath = modulePath => [modulePath, `${modulePath}.js`].find(fs.existsSync);

// 核心的递归遍历 DFS 代码
function deepTravel(fullPath, moduleList, moduleDepMapList) {
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');
    let match = null;
    while ((match = modulePathMatcher.exec(moduleText)) !== null) {
        const [, subModulePath] = match;
    }
}
```

既然是 DFS 算法，那么当获取到第一个子组件地址时，我们就应该直接开启递归进行深度分析。

```js
function deepTravel(fullPath, moduleList, moduleDepMapList) {
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');
    let match = null;
    while ((match = modulePathMatcher.exec(moduleText)) !== null) {
        const [, subModulePath] = match;

        // 计算匹配到的子模块绝对路径
        const childModuleFullPath = path.resolve(path.dirname(getFilePath(fullPath)), subModulePath);

        // 直接开启递归，然后去分析子组件模块，这样的话就会优先一直向下
        deepTravel(childModuleFullPath, moduleList, moduleDepMapList);
    }
}
```

我们在遇到第一个子模块之后，计算出这个子模块的绝对路径，然后立刻开始递归，同时将 moduleList 和 moduleDepMapList 这两个「全局变量」传递下去。

### 从最简单的情况分析起

找递归出口最快也是最准确的方式，就是使用`最简单情景具体的值`或是用`特殊场景具体的值`逐步带入，一点一点的完善出口。

对于我们代码的例子来说，最简单的场景就是入口模块没有任何依赖，不用分析子模块直接结束递归的方式。这种情景下，模块代码只需要包裹在一个函数内部，并且 push 进 moduleList 即可。

处理子模块的代码块，把我们的函数分成了三块，分别是 `处理子模块前的区域`，`处理子模块区域`和`处理完成子模块区域`三部分。

```js
// 新增了一个函数，作用是将字符串包裹在一个函数当中，并返回新的字符串
function wrapFun(content) {
    const funcWrapper = ['function (require, module, exports) {', '}'];
    return `${funcWrapper[0]}\n${content}\n${funcWrapper[1]}`;
}

function deepTravel(fullPath, moduleList, moduleDepMapList) {
    const moduleText = fs.readFileSync(getFilePath(fullPath), 'utf-8');

    moduleList.push(wrapFunc(moduleText)); /* ---- 1. 在处理子模块前处理当前模块内容 ---- */

    /* ---- 分析子模块部分 ---- */
    let match = null;
    while ((match = modulePathMatcher.exec(moduleText)) !== null) {
        const [, subModulePath] = match;
        const childModuleFullPath = path.resolve(path.dirname(getFilePath(fullPath)), subModulePath);
        deepTravel(childModuleFullPath, moduleList, moduleDepMapList);
    }
    /* ---- 分析子模块部分 ---- */

    moduleList.push(wrapFunc(moduleText)); /* ---- 2. 在处理子模块后处理当前模块内容 ---- */
}
```

上面的代码我们标记了几处注释，从函数开始到分析子模块注释区域为`在处理子模块前处理当前模块内容`，从 while 函数结束到 depTreval 函数结束为 `在处理子模块后处理当前模块内容`。对于这种最简单情况来说，在这两个地方包裹函数并 push 到 moduleList 似乎都可以，那么我们究竟应该在哪个阶段进行这个操作呢？

看起来，前序遍历和后续遍历对最简单的场景都满足要求。那么接下来，我们就要分析稍复杂一步的场景，来决定具体使用哪种遍历方式。比入口模块没有依赖子模块的更复杂一点的，就是入口模块依赖了一个子模块，这也就意味着我们接下来的分析要进入 while 循环了，也意味着 moduleDepMapList 里面需要有结果了。

