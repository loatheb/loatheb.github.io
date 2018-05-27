---
title: AMD CMD CommonJS ES6模块规范详解
date: '2017-4-5'
tags:
 - CommonJS
 - AMD
 - CMD
 - EcmaScript2015
---

**Javascript** 在 **ECMAScript2015**（以下简称 **ES6** ）规范之前。单独从语言本身来说，不像 **c** , **python** 等其他语言那样，带有类似 `export` ,    `import` 语法的模块加载，模块导出的功能的。所以说，我们只能单纯的通过文件合并来进行组件的加载和导出，但是不同文件的变量又会混淆在一起，有时会因为变量提升造成一系列问题。同时，在平常的逻辑中，模块的加载和导出对于业务抽象，组件复用来说都是重中之重的，这也就促成了 **js** 模块化的脚步。

<!-- more -->

## 历史遗留问题

#### 变量名冲突问题

**一方面。**假设在业务中，我们需要抽象出来一个 `ajax`方法，它的作用就是通过配置项发送一次`ajax`请求。常见的做法就是将这个方法单独写成文件，这样加载了这个文件就等于定义了这个方法。

```js
// ajax.js
var ajax = function (config) {
  ...
};
```

在其他模块中，我们假设有`a.js` 和`b.js`两个文件，分别代表着两种不同的业务，但是都用到了`ajax`这个方法。为了保证正确的依赖关系，我们必须在 `html` 文件中下一番功夫，即必须保证正确的加载顺序，才能保证在程序运行到`a.js`和`b.js`相关位置时，`ajax`函数已经被正确加载。所以，理想中的`js`加载顺序应该是这样的：

```html
<script src='ajax.js'></script>
<script src='a.js'></script>
<script src='b.js'></script>
```

将公用的依赖 `ajax.js` 放置在所有项的最前面。这样，我们才能保证在`a.js`和`b.js`内部，一定能正确运行 `ajax` 函数相关内容。

此时，整个页面的 js 文件结构如下：

```
// ajax.js
var ajax = function () {}

************
// a.js
var something = XXX;
ajax();

************
// b.js
var anotherthing = XXX;
ajax();
```

所有文件中的变量等于直接挂载在全局对象下，没有模块内的私有变量，这对于复杂场景是十分危险的，有可能变量名冲突造成无法发现的 **bug** 。

而且如果有非常多公有模块，或者非常复杂的逻辑耦合。这种通过顺序依赖一来不好维护，二来不美观简单，就显得有些捉襟见肘。

#### 后期维护问题

**另一方面。**随着业务的增加，来了第三位开发者，他并没有参与 **a.js** 和 **b.js** 业务的开发，而且不知道前面最前面有一个`ajax`方法，此时 **html** 文件结构如下：

```html
<script src='ajax.js'></script>
<script src='a.js'></script>
<script src='b.js'></script>
<script src='c.js'></script>	// 由第三位开发者新增加文件
```

此时在不知情的情况下假如他重新写了`ajax`函数来满足自己的需求，这时前面的 **a.js** 和 **b.js** 就可能导致问题，而且如果前面的依赖文件过多，变量过多，对后面的开发者来说都是一场挑战，需要时刻注意变量命名问题。

但是这样的情景是有解决方法的，那就是使用 **namespace（命名空间）** 。在开发时约定每个人写的变量加入自己的命名空间。例如对于`ajax`方法：

```js
var a = {},
    b = {};
a.ajax = function () {
  ... // a 定义的 ajax
}

b.ajax = function () {
  ... // b 定义的 ajax
}
```

这样，`a`和`b`定义的变量就永远不会重复，`b`在修改的时候仅仅修改自己的内容，永远不会因为自己的操作导致其他人代码出现问题。同样的，这样做会引入一个新的问题，如果说命名空间过长的话，在使用中我们需要记住很长的名称空间，就会出现下面这样的代码：

```js
// c.js
c.utils.vars.status	// 为了使用 c 定义的一个变量
```

这样代码无论是阅读还是 debug 都会造成一定的难度。所以在这种历史遗留的矛盾中，开发者必须小心翼翼，才能保证代码的正确性和可维护性。

正因为这样或者那样的矛盾无法解决。这种时候，在前端圈 `'CommonJS'、'CMD'、'AMD'`规范应运而生，它们的出现，都是为了解决上面说的这种问题。从而**对复杂模块进行解耦，使模块之间关系变得简单。**

## CommonJS规范

一说到`CommonJs`，就不得不说 `Node.js`。对于前端开发者来说，`node` 一定不会陌生。`node`是对 **谷歌 V8** 引擎封装的一层 `js` 的解释器，chrome 浏览器使用的就是这种引擎来解析 js 。

`node`简单来说就是使用 js 通过来写服务端代码。在`node`的实现过程中，实现了基于`CommonJS`的模块开发。具体内容可以查看`node`官方网站的介绍[https://nodejs.org/docs/latest/api/modules.html](https://nodejs.org/docs/latest/api/modules.html)，里面详细介绍了`node`对于`module`的定义。在 **CommonJS** 中，我们通过 `require` 进行模块的引入，通过 `exports` 和 `module.exports` 进行模块的导出。

#### module的定义

在node中，每一个文件就是一个单独的module（模块）。例如：

```js
// circle.js
const PI = Math.PI;

exports.area = (r) => PI * r * r;

exports.circumference = (r) => 2 * PI * r;
```

```js
// foo.js
const circle = require('./circle.js');

circle.area(4);	// 16
```

在`foo.js`文件中，通过`require`函数加载了相同目录下的`circle.js`模块。同样的，对于`circle.js`，通过给对象`exports`添加`area`和`circumference`两个方法，进行了方法的导出，如果要新增加方法的话，只需要在`circle.js`中继续向`exports`对象增加属性或者方法即可。

```js
// circle.js
const PI = Math.PI;

exports.area = (r) => PI * r * r;

exports.circumference = (r) => 2 * PI * r;

exports.status = false;

exports.radius = (l) => l / 2;
```

对于`PI`这个变量，如果没有显式的导出的话，仅仅存在于 `circle.js` 这个模块中，对于外部来说，它是模块内部 **私有** 的，这个在后面我们会详细解释它的原理。

上面这种给exports新增加属性和方法进行模块导出具有一定的局限性，有时候我们希望直接导出一个对象：

```js
// module.js
// 我们希望直接导出的对象
var obj = {
  a: 1,
  b: 2
};
```

从而在`require()`时，接收模块的变量具有`a`和`b`两个属性。

```js
// 我们希望这样去使用模块
var module = require('./module.js');
module.a;	// 1
module.b;	// 2
```

这种时候，在`module.js`中，如何进行模块的导出呢？可能会立马这样想：

```js
// module.js
// 我们希望直接导出的对象
var obj = {
  a: 1,
  b: 2
};

exports = obj;	// 不会生效
```

实际上这样是不会生效的，具体原因我们会在后面详细讲解。

在这里我们就需要使用`module.exports`进行模块的导出，下面这种写法才是正确的：

```js
// module.js
// 我们希望直接导出的对象
var obj = {
  a: 1,
  b: 2
};

module.exports = obj;	// 生效，导出 obj
```

所以简单来说，在模块导出时。如果单纯的导出一些属性或者方法，可以通过给 `exports` 添加属性或者方法的方式进行导出、如果想直接导出一个对象作为属性，方法的集合，就得使用 `module.exports`进行模块的导出。

**在本小节的最后，我们会通过代码详细解释为什么文件中的变量是私有的，以及在上面的例子中为什么`exports`不生效而必须使用`module.exports`。**

#### module加载顺序

上面的一小段主要讲述了`exports`和`module.exports`如何进行模块的导出。这一段主要讲解一下模块如何导入，也就是`require()`函数。

首先我们要明白`node`当中的模块可以分为两类，一类是`core`模块，一类是`自定义`模块。

**core模块**可以说是官方自带模块，这些是官方封装的一些常见API，从文件操作到HTTP请求。这些模块都称为`node` `core` 模块。具体内容可以参考node官网里面对应版本的API文档[https://nodejs.org/docs/latest/api/documentation.html](https://nodejs.org/docs/latest/api/documentation.html)，里面有详细的具体API对应的方法和内容。举个例子：

```js
// 以 path 模块举例
var path = require('path');

// ...即可以使用 path 模块内部的方法，例如 resolve 和 join 方法
path.resolve();
path.join();
```

可以看到，对于core模块来说，我们可以直接采用`require(<core module name>)`进行加载。

**自定义模块**故名思义就是自己或别人书写的模块。对于他人开源的模块，`npm`是node的一个包管理软件，而在npmjs.com上有上百万开发者发布的模块，这里的模块(module)也称为包(package)，只要符合开源协议，我们都能自由使用。

对于他人的模块，我们只需要搜索到我们想要的模块，再执行`npm i <module name>`，接下来就会在执行路径下生成一个 node_modules 目录，这时候，无论我们在哪个文件中，我们只需要通过`require(<module name>)`进行引入依赖。

```js
// 从 npmjs.com 找到的模块
npm i utils --save

// 安装完成后在当前目录下生成 node_modules 目录

// index.js
var utils = require('utils');

// folders/index.js
var utils = require('utils');
```

对于当前自己写入的模块，即我们自行导出的模块，就无法采用这种方式引入了。比如通过`require()`传入一个路径或者文件，才能够加载正确的模块。

例如，我们从`npmjs.com`安装了一个`path`模块，这个属于自定义模块，与`core`模块中的`path`重名，当我们`require('path')`的时候加载的就是`core`模块的内容，而不是自己安装的`path`模块。下面这个表表明了`require`加载的顺序：

```js
当前路径为 Y ，此时一个文件中加载一个模块 require(X)
1. 如果 X 是 core module,
   a. 返回 core module
   b. STOP
2. 如果 X 以 './' 或 '/' 或 '../' 开始
   a. LOAD_AS_FILE(Y + X)	// 首先执行函数 LOAD_AS_FILE 以文件方式加载
   b. LOAD_AS_DIRECTORY(Y + X)  // 其次执行函数 LOAD_AS_DIRECTORY 以路径方式加载
3. LOAD_NODE_MODULES(X, dirname(Y)) // 最后执行 LOAD_NODE_MODULES 以 node_modules 安装路径加载
4. 返回 "not found"

LOAD_AS_FILE(X)
1. 如果 X 是一个文件, 以 JavaScript 文本的方式加载 X 。加载结束
2. 如果 X.js 是一个文件, 以 JavaScript 文本的方式加载 X.js 。加载结束
3. 如果 X.json 是一个文件, 以 JavaScript 对象的方式加载 X.json 。加载结束
4. 如果 X.node 是一个文件, 以 JavaScript 二进制的方式加载 X.node 。加载结束

LOAD_AS_DIRECTORY(X)
1. 如果存在 X/package.json ,
   a. 解析 X/package.json, 查找 "main" 键的值.
   b. let M = X + (json 'main' 键的值)
   c. LOAD_AS_FILE(M)	// 重新执行 LOAD_AS_FILE
2. 如果 X/index.js 是一个文件, 以 JavaScript 文本的方式加载 X/index.js 。加载结束
3. 如果 X/index.json 是一个文件, 以 JavaScript 对象的方式加载 X/index.json 。加载结束
4. 如果 X/index.node 是一个文件, 以 JavaScript 二进制的方式加载 X/index.node 。加载结束

LOAD_NODE_MODULES(X, START)
1. let DIRS=NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
   a. LOAD_AS_FILE(DIR/X)
   b. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
   a. if PARTS[I] = "node_modules" CONTINUE
   b. DIR = path join(PARTS[0 .. I] + "node_modules")
   c. DIRS = DIRS + DIR
   d. let I = I - 1
5. return DIRS
```

**上面是核心的全部加载顺序。对于平常使用时，我们只需要记住常见的加载顺序即可：**

1. `require()`参数**不是以 `'./'` 或 `'/'` 或 `'../'` 开始**
   1. 加载核心模块
   2. 由当前路径向上冒泡，找到最近的`node_modules`并加载该模块
   3. 模块未找到
2. `require()`参数**是以 `'./'` 或 `'/'` 或 `'../'` 开始，假设参数为`X`**
   1. 依次寻找`X`，`X.js`，`X.json`，`X.node`文件，并按照对应方式加载
   2. 如果没有找到，则以路径处理
      1. 如果该路径下有`package.json`文件，则以`package.json`里面`main`的键值作为目标。
      2. 如果没有`package.json`文件，则依次寻找`X/index.js`，`X/index.json`，`X/index.node`文件，并按照对应方式加载。
   3. 模块未找到

#### 模块缓存

前面讲到了，对于一个模块内部来说，多次加载另一个模块时会进行缓存，也就是`require`函数在加载相同的模块仅仅执行一次：

```js
// module.js
module.exports = new Date();
```

```js
// index.js
var time = require('./module');
var anotherTime = require('./module');

time === anotherTime;	// true
```

在上面这个例子中，我们可以得出，在一个文件（模块）内加载其他模块的内容，加载的内容只会执行一次，也就是添加了缓存。如果希望模块导入时多次执行，可以在模块导出时导出一个函数，加载时多次调用。

```js
// module.js
exports.getTime = () => new Date();
```

```js
// index.js
var time = require('./module');
var anotherTime = require('./module');

time.getTime() === anotherTime.getTime();	// false
```

#### 循环加载

模块加载时有时会出现循环加载的现象。

```js
// a.js
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done = %j', b.done);
exports.done = true;
console.log('a done');
```

```js
// b.js
console.log('b starting');
exports.done = false;
const a = require('./a.js');
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');
```

```js
// index.js
console.log('main starting');
const a = require('./a.js');
const b = require('./b.js');
console.log('in main, a.done=%j, b.done=%j', a.done, b.done);
```

在`index.js`	中，首先它加载了`a`模块，对于`a`模块内，中间又加载了`b`模块，结果在`b`模块内，又重新加载了`a`模块。这里就造成了一个循环加载。为了避免无限循环，所以，在`b`模块请求加载`a`时，**加载的是一个不完全的`a`模块**，之后执行完了`b`模块的内容之后，在`a`模块内部，继续向下执行，最终执行结束。

而在`index.js`内部，则加载的是全部完成状态下的`a`,`b`模块，所以输出的也是最终状态的输出。

```shell
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done=true, b.done=true
```

#### module对象

在node中，每一个文件就是一个模块，对于每个模块来说，对应就有一个module对象，上面例子中经常出现的module.exports就是调用了module对象的exports方法，module对象不是一个全局对象，但是存在于每个文件中。

* module.children

  * <Array>

    当前模块加载的其他模块列表

* module.filename

  * <String>

    当前模块完整的文件名

* module.id

  * <String>

    当前模块的标示

* module.loaded

  * <Boolean>

    当前模块是否加载完毕

* module.parent

  * <Object>

    第一个加载当前模块的模块

* module.require(id)

  * id <String>

  * return <Object> 参数 id 对应模块导出的内容

    提供了另一种加载模块的方式，与`require()`作用相同。

* module.exports

  * <Object>

    导出一个模块，可以是函数，可以是对象。

* **exports** 对象

  exports是另一种导出模块的方法，在某些情况下，它和module.exports是相同的。

  ```js
  // index.js
  exports.hello = false;
  module.exports.hello = false;
  ```

  但是如果对exports重新分配一个对象的话，它们则不相同。

  ```js
  // index.js
  exports = { hello: false }; 	// 失效
  module.exports = { hello: false };	// 有效
  ```

  从`require`函数的代码来理解：

  ```js
  function require(...) {
    var module = { exports: {} };
    // require 全局 module 对象，对 module 进行缓存
    ((module, exports) => {

      // ... 所有模块内容在此 some_fun() 代表一个模块，模块内容在一个函数内
      function some_func() {};

      exports = some_func;
  	// 对于引用类型的值来说，重新赋值切断了 exports 和 module.exports 之间的联系

      module.exports = some_func;
   	// 对于 module.exports 来说，真正将模块导出
    })(module, module.exports);
    // 最终返回 module.exports 引用的内容
    return module.exports;
  }
  ```

  到此，上面的代码解释了之前为什么模块内的变量仅属于模块私有，为什么加载同一个模块代码仅执行一次，以及`exports`和`module.exports`两者之间的区别。到此，`CommonJS`模块规范也基本结束。


上面所介绍的 `CommonJS` 规范，从某种意义上来说，仅仅适用于`node`端。对于服务端来说，性能的瓶颈一般在并发或者其他问题上，文件间加载和代码量基本可以忽略不计，所以模块加载就是读取文件并全部加载，仅仅在循环依赖时会简单处理模块。而对于客户端来说，性能的瓶颈在于文件的传输时延和渲染速率。这两者与文件依赖关系和传输文件大小息息相关。当前模块依赖其他模块时，就需要从服务端重新请求，具有一定的时延，而且加载完成后，单纯的文件导入，无论是HTTP请求还是客户端解析，都会造成巨大的性能影响。所以在客户端来说，异步和按需加载成了模块导出加载设计时最先考虑的因素。于是乎就有了 CMD 规范和 AMD规范。

## AMD规范

AMD 规范就是为了解决客户端问题而出现的，通过 AMD 规范，模块与模块间的依赖关系可以变为异步进行。

#### 定义

```js
define(id?, dependencies?, factory);
```

define函数为模块内的全局变量，在每个模块内部都能够使用。

* id

  * <String>

    可选，如果没有则默认为当前模块的文件名。参数意义是指定当前模块的名称。

    模块格式同 CommonJS 模块格式，但有些地方有些不同。

    * 模块名是由一个或多个单词以正斜杠为分隔符拼接成的字符串

    - 单词须为驼峰形式，或者"."，".."
    - 模块名不允许文件扩展名的形式，如".js"
    - 模块名可以为 "相对的" 或 "顶级的"。如果首字符为"."或".."则为"相对的"模块名
    - 顶级的模块名从根命名空间的概念模块解析
    - 相对的模块名从 "require" 书写和调用的模块解析

* dependencies

  * <Array>

    定义当前模块所依赖的模块数组。依赖的模块最终会以相同的顺序传入最后的工厂方法中。

    依赖参数是可选的，如果省略参数，则会默认应用`["require", "exports", "module"]`。

* factory

  * <Function|Object|Array|String|Boolean>

    工厂方法，为模块初始化要执行的函数或对象。如果为函数，它应该只被执行一次。如果是对象，此对象应该为模块的输出值。

    如果工厂方法返回一个值（对象，函数，或任意强制类型转换为true的值），应该为设置为模块的输出值。

同时，define函数还具有一个amd属性，它的值作为一个对象。目的是，如果当前全局已经存在define变量，但是并不是因为AMD规范而引入的话，保证之前的define变量存在而不冲突。

AMD 提倡提前加载，所有的依赖写在最前面（define函数的第二个参数）。但是另一方面，他也兼容 CommonJS的写法，只需要缺省第二个参数即可。

```js
// module.js
define(function (require, exports, module) {
  var math = require('./math');
  exports.increament = function (val) {
    return math.add(val, 1);
  }
})
```



#### 例子

创建一个名为"alpha"的模块，使用了require，exports，和名为"beta"的模块:

```js
define("alpha", ["require", "exports", "beta"], function (require, exports, beta) {
     exports.verb = function() {
         return beta.verb();
         //Or:
         return require("beta").verb();
     }
 });
```

一个返回对象的匿名模块：

```js
 define(["alpha"], function (alpha) {
     return {
       verb: function(){
         return alpha.verb() + 2;
       }
     };
 });
```

一个没有依赖性的模块可以直接定义对象：

```js
 define({
   add: function(x, y){
     return x + y;
   }
 });
```

一个使用了简单CommonJS转换的模块定义：

```js
 define(function (require, exports, module) {
   var a = require('a'),
       b = require('b');

   exports.action = function () {};
 });
```

## CMD规范

`CMD`规范全称`Common Module Definition`，具体内容可以看 **@玉伯** 整理的文档 [https://github.com/cmdjs/specification/blob/master/draft/module.md](https://github.com/cmdjs/specification/blob/master/draft/module.md)。

CMD 规范并没有任何浏览器原生支持，所以不能像node那样，直接在浏览器中使用符合 CMD 的模块，必须提前加载能够对应解析 CMD 规范模块的库。**@玉伯**所书写的 **sea.js**就是一个能够解析CMD模块的库。具体使用方法如下：

```html
<script src='./sea.js'></script>	// 因为没有浏览器端原生支持 CMD ，必须使用第三方解析 CMD 模块
<script src='./module.js'></script> // 符合 CMD 的模块
```

在 CMD 规范中，每一个文件也当作一个模块看待。这时候对于每个模块（文件）来说，全局就有`define`变量，这也是加载模块和导出模块的关键。

define是一个函数，存在于每一个模块中，对于define函数：

* 仅仅接受一个参数，可以是函数或者其他有效的类型
* 如果参数是一个函数，则函数的三个参数依次是：`require`,`exports`和`module`
* 如果参数不是一个函数，则该模块的exports则导出对应对象

对于define函数，参数为函数时调用方式如下：

```js
define(function(require, exports, module) {

 //  模块加载和导出都在此进行

});
```

对于函数三个参数具体信息：

* require

  * <Function>

    require参数是一个函数，接收一个module对应的id，module id 默认为文件路径，支持相对路径。然后返回对应模块导出的内容，如果未能找到对应模块，则返回`null`。

* require.async

  * <Function>

    require.async接收一个module的列表，第二个参数接收一个可选的回调函数。回调函数接收前面所有加载的module所返回的内容，而且顺序与前面加载模块时书写的顺序相同。

* exports

  * <Object>

    exports 是一个对象，简单来说是`module.exports`的引用，可以将模块需要导出的方法直接挂载在该对象上。

* module

  * module.uri

    * <String>

      返回模块的完整路径

  * module.dependencies

    * <Array>

      返回当前模块导入的其他模块列表

  * module.exports

    * <Object>

      模块导出内容，`require`加载时加载的就是此处的内容

#### 模块标识

1. 模块标识必须是一个有效的字符串
2. 模块标识不需要一个后缀，例如`.js`
3. 模块表示应该采用'-'的字符串，例如`foo-bar`
4. 模块标识可以采用相对路径，如`./foo`或者`../bar`

CMD 提倡按需加载，只需要在具体需要使用到的地方加载即可：

```js
// module.js
define(function (require, exports, module) {
  var a = require('./a');
  console.log(a.log());
  var b = require('./b');	// 此时加载 b 模块
  console.log(b.log());
  if (a.test) {
    var c = require('./c');		// 此时加载 c 模块
  }
});
```

#### 示例

```js
// module.js
define(function (require, exports, module) {
  exports.log = function () {
    console.log('this is module');
  };
});
```

```js
// index.js
define(function (require, exports, module) {
  var module = require('./module');
  module.log();	// this is module
});
```

## ES2015 模块详解

ES2015规范作为js的官方规范，提出了新的模块加载方式。在CommonJS规范中，模块的加载是运行时进行的。比如说：

```js
// index.js
// CommonJS 规范加载
var path = require('path');	// 运行时进行 path 模块加载
```

运行时加载的模块，就不能在编译时静态优化。ES2015的模块加载与导出都是静态进行的。这样在编译时，对应代码已经加载完毕，能够立刻进行优化。

#### 模块导出

ES6中的模块导出统一使用export语法，下面几种方法都是可以将模块进行导出的。

```js
// module.js
export var a = 1;

var b = 1;
export {b};

var c = 1;
export default c;
```

上面的三种方式都能够对应导出模块中的内容。同样的，我们还可以对模块进行改写。同时，使用`as`关键字可以对导出的模块进行重命名：

```js
// module.js
var func = function () {
  console.log('test');
}
export {func as foo};
```

#### 模块加载

模块加载使用的是import关键字，它能够将需要导入的模块进行导入。使用代码如下：

```js
// 导出
export default function () {
  console.log('info');
}
```