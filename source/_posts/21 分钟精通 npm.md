---
title: 你不知道的 npm
date: '2017-3-4'
tags:
 - npm
---

​        npm 作为 node 自带的包管理工具，每天通过 npm 命令下载和使用的第三方依赖已然过亿，npmjs.com 累计存放的包也突破了 50 万，npm 命令已经和每一个 JS 开发者紧密相连。同时，npm 在前端的工程化构建占有重要地位，本文由浅入深，详解 npm 命令。

​        本文面向有一定经验的前端开发者，从 package.json 的配置入手，逐步深入阐述了笔者对 node 包管理中易错点以及思考。

<!-- more -->

### package 的组成

​        无论是前端项目还是 node 的项目，通过 npm 的命令我们可以下载一些项目依赖，同时我们也可以发布一些 package 到 npm 的官方仓库。但是，不是所有的代码都符合作为 package 的规范，究竟什么样的代码符合 npm 的规范呢？

#### 作为一个 package 的必要条件

​        对于一个 package 来说，必须在其根目录下具有 package.json 这个文件。对于这个 json 文件，必须有 name 和 version 这两个键的值，之后我们就可以通过 `npm publish` 命令来发布这个 package 到 npmjs.com 上，供第三方下载和使用。

​        同时，作为一个包管理工具，npm 会将所有安装的包放置在运行命令目录下的 node_modules 下，而通过 npm install 命令，默认可以安装 npmjs.com 上的所有 package，在项目中引用。

#### package 的种类

​        从功能上来说，package 大致可以分为 **cli 与 module **两类。

​        cli 全称 command line interface，意为直接能从终端通过命令运行的脚本。例如我们熟悉的 webpack 就是一种 cli 工具。

​        当我们使用`npm install -g webpack`全局安装 webpack 之后，就能通过诸如 `webpack index.js index.bundle.js`的命令来进行打包操作。

​        相对应的，还有一种 package 就是最常见的 module ，在项目中安装之后，可以通过不同的模块化规范的而引入到自己的文件中。

​        当我们使用`npm i --save jquery`安装过 jquery 之后，我们就能通过 ESM ( `import jquery from 'jquery`) 或者 CommonJS ( `const jquery = require('jquery')` ) 之类的模块化语法将 jquery 对象引用到项目中。

​        **对于某些 package 来说，它既是一个 cli，也是一个 module，也就是既能通过命令行运行命令，也能进行模块化加载。**

​        例如 webpack 就是一个典型的例子，`webpack index.js index.bundle.js` 和 `const webpack = require('webpack')`都是可以正常工作并且又对应的确定含义的。

​        **而在 package.json 文件中，npm 会通过配置不同的 key 来区分当前模块究竟是 cli 还是一个 module。**

​        一部分 js 文件通过特殊的配置可以直接执行，例如，我们新建一个名为 index.js 的文件如下：

```javascript
#!/usr/bin/env node
console.log("this is a js file running in terminal directly!")
```


​        与普通的 js 文件不同，这种需要执行的 js 文件会在文件的开头加入 `!/usr/bin/env node` 这一行代码，可以使得其通过 node 来执行这个脚本。当我们运行`chmod +x index.js`，更改这个文件的权限成为可执行时，就能够直接通过 `./index.js` 来执行这个文件，效果等同于 `node index.js`。

![/images/npm/index_chmod.png](/images/npm/index_chmod.png)

​        package.json 中的 bin (全称为 binary) 字段 ，目标是接受一个对象，存放的当前 package 下的可执行命令与对应执行的 node 脚本的映射。举例如下：

```javascript
"bin": {
  "cmd": "index.js"
}
```

​        例如上面的配置，运行 `npm link` 命令，就会在 node_modules/.bin/ 目录下创建一个 cmd 文件**软链接**到 index.js 。在安装这个依赖之后，直接运行 `cmd` 命令，等同于运行 `./index.js` ，等同于运行 `node index.js` ，从而执行这个 node 脚本。

​       而 package.json 中的 main 字段，代表着当这个 package 在 CommonJS 规范下作为 module 时在文件中会引入哪个文件。

```javascript
"main": "lib/webpack.js"
```

​        当我们使用 CommonJS 的 `require('webpack')` 语法来加载模块时，就会去 node_modules 目录下 webpack 文件夹中，找出 `lib/webpack.js` 导出的模块，加载进来。（模块加载有严格的顺序，相关权重可以参考另一篇讲模块化规范的文章）

​        通过上面的介绍，简单的一句话总结就是，所有存放在 npm 仓库中的统称为 package 其中有 module ，可以被其他文件直接加载进去，还有 cli ，可以直接以命令行执行，同时有些 package 既属于 module 又属于 cli。他们是整个 npm 生态的基石。

#### dependencies, devDependencies, peerDependencies,  optionalDependencies 和 bundledDependencies

​        dependencies 代表项目运行过程中需要的依赖，一般通过 `npm --save ${module_name}` 进行安装。

​        devDependencies 代表项目构建过程中需要的依赖，一般通过 `npm --save-dev ${module_name}` 进行安装。

​        peerDependencies 代表项目运行过程中需要的依赖，但是使用该依赖宿主所使用的版本。

​        这种依赖的写法常见于大型框架的插件上。例如 babel-preset-xx，xx-loader。例如我们需要扩展 webpack 对象，并给这个 package 命名为 webpack-test，则在 webpack-test 中，首先需要引用 webpack 。

```javascript
// index.js
const webpack = require('webpack')
webpack.plugin = 'do something for webpck'
```

​        这样一来，webpack 就必须放置于 webpack-test 项目下 package.json 的 dependencies 中作为运行时的依赖。**然而对于需要使用 webpack-test 这个项目的人来说，webpack 必须作为一个 dependencies 放置于上层。**所以在 webpack-test 中，我们不需要显示的指定 webpack 的版本，只需将它放置于 peerDependencies ，就是说明了，使用调用我这个插件的人的 webpack 的版本。这样的好处就是对于宿主来说，可以有效避免处理多个相同模块版本的问题。

​        当使用 peerDependencies 指定了特殊版本而宿主并没有使用该版本时，则会报出 warning 但是并不会继续安装这个依赖在 node_modules 中。以 babel-loader 为例，它将 webpack 和 babel-core 加入了 peerDependencies ，如果没有符合条件的版本，则会在安装时报出如下的警告，但是并不会继续安装符合条件的依赖：

![/images/npm/missing_peerDependencies.png](/images/npm/missing_peerDependencies.png)

​       注意：如果使用 npm 的版本过低（npm@1 或 npm@2），默认是会安装 peerDependencies 里的依赖的，只有在 npm@3 之后，才会生成警告而不继续安装。

​        optionalDependencies 代表项目可选的依赖，意为即使这个模块没有获取到，但是整个代码和 npm 的安装依旧能够运行的一类模块。使用这种模块时，常常需要我们显示的在代码中做好判断。

```Javascript
let module;
try {
	module = require('webpack');
} catch (e) {
  	module = null;
}
```

​        bundledDependencies 与上面所有的配置都不相同，bundledDependencies 接受一个数组，里面可以配置一些依赖的名称。当通过 npm pack 命令将当前包打包成为压缩文件之后，在进行安装就会默认安装上 bundledDependencies 里面的内容。

​        当我们完成一个 package 的书写的时候，可以通过 `npm pack` 将这个包打包成为一个 `${name}-${version}.tgz` 的压缩文件，而把这些依赖压缩的方式曾经也可以作为控制版本的一种方法。如下的例子：

```json
{
    "name": "module-test",
	"version": "1.0.0"
}
```

​        `npm pack` 之后，就会在当前目录下生成 module-test-1.0.0.tgz 的文件，在开发过程中，将这个文件加入 git 的目录进行跟踪曾经也可以作为一种锁定版本的方式，将这个文件解压，就会有当前 package 的所有文件。如果将一些依赖加入 bundledDependencies （又名 bundleDependencies），则会在 `npm pack` 时，这些依赖也放入 node_modules 中。同时，发布时这些内容也会作为 package 的一部分同时发布。

#### module 在多环境下的加载

​        CommonJS 规范中，使用 require 函数进行模块的加载，它会把 package.json 中的 main 字段的值的文件导出的内容加载进来。

```json
{
  "name": "module-test",
  "version": "1.0.0",
  "main": "./lib/index.js"
}
```

​        上面这个例子中，`require('module-test')` 就会默认将 `./lib/index.js` 文件 export 出来的内容加载进来。

​        但是在 package.json 中，并不是只有 main 这么一个字段来表示加载模块的位置。当我们使用 webpack 或者 browerify 等模块化的处理工具时，它们会通过打包配置的 target ，会使用 package.json 中的不同字段。

​        当 target 为 node 时，webpack 会默认通过 **["module", "main"]** 这个顺序依次查找 package.json 的字段的值。如果没有 module 字段，则会加载 main 字段的文件的结果。

​        当 target 为 web （默认）， webworker 或者其他内容时，默认会使用 **["browser", "module", "main"] ** 的顺序判断 package.json 对应的字段。

​        举个例子，有如下目录结构：

```
- module
  - index.browser.js
  - index.node.js
  - package.json
- index.js
```

​        其中的 index.browser.js 和 index.node.js 内容分别为：

```javascript
// module/index.browser.js
module.exports = () => {
  console.log('this is `index.browser.js` saying!')
}

// module/index.node.js
module.exports = () => {
  console.log('this is `index.node.js` saying!')
}
```

​        index.js 作为我们加载模块的入口文件，采用如下的方式加载模块：

```javascript
// index.js
require('./module')()
```

​        对于 module 目录来说，其中的 package.json 有如下内容：

```json
{
  "name": "module",
  "main": "index.node.js",
  "browser": "index.browser.js",
  ....
}
```

​        当我们使用 browserify 或者 webpack 的默认 target （默认 target：web）进行打包时：

```shell
browserify index.js -o index.browserify.js
node index.browserify.js // this is `index.browser.js` saying!

webpack index.js index.webpack.js
node index.webpack.js // this is `index.browser.js` saying!
```

​        如果我们使用 webpack 将 target 显示的表示成为 node 时：

```shell
webpack --target node index.js index.webpack.node.js
node index.webpack.node.js // this is `index.node.js` saying!
```

​        而当我们删除 module/package.json 中的 browser 字段时，重新执行上面所有的打包命令：

```shell
browserify index.js -o index.browserify.js
node index.browserify.js // this is `index.node.js` saying!

webpack index.js index.webpack.js
node index.webpack.js // this is `index.node.js` saying!

webpack --target node index.js index.webpack.node.js
node index.webpack.node.js // this is `index.node.js` saying!
```

（以上部分代码存放在 [github](https://github.com/loatheb/blog-code/tree/master/1.npm) ）

​        所以说，我们可以通过简单的配置不同的字段让 bundle 时获取不同的文件。

```javascript
{
    "main": "./lib/index.js",
    "browser": "./lib/index.browser.js",
    "module": "./lib/index.node.js"
}
```

#### npm 与 npx

​        在 npm@5 版本的今天，随着 npm@5.2 版本的更新，同时还会附带安装一个 npx 命令。npx 命令的产生是为了方便的使用一些存放在 npm 仓库中的 cli 命令。npx 最大的特定恐怕就是它的 cli 友好型，传统的 npm 命令，当我们需要运行一些 cli 命令时，一般都必须加入 devDependencies 中去，然后通过配置 scripts 才能够进行执行。

​        比如，mocha 或者 webpack 这种构建过程中需要的依赖，我们使用 npx 命令就可以不用再将它们写进 devDependiences 中了，通过 npx mocha 或者 npx webpack 命令就能直接运行，如果 webpack 这个 package 没有安装，则会从 npm 仓库自动获取最新的依赖进行执行，如果 webpack 这个 package 已经被安装了，则会像像 scripts 里面配置那样，直接运行 package 中的脚本。

​        通过 npx 还能直接运行 create-react-app 或者 vue-cli 这种脚本，并且自动运行最新的版本，再也不需要手动全局安装。

![/images/npm/npx.png](/images/npm/npx.png)

​        可以看到，使用 npx 命令使用 create-react-app 的 cli 进行了操作，但是并没有在全局留下痕迹，对于使用一些脚手架来说是非常方便的。

### 现代的包管理

​        npm 一直作为 node 默认的包管理器。不过后来的某一天 npm 的一位作者跑去了重新开发了 yarn 。yarn 一经推出，立马受到了广泛的关注，关于 yarn 和 npm 孰优孰劣的讨论至今都没有停止过。而且经常有开发者会问起酒精 npm 与 yarn 又有哪些异同？在一开始，大概可以总结为以下几点：

1. yarn 具有离线缓存
2. yarn 的安装速度更快
3. yarn 自动安装 yarn.lock 锁定版本

​        其中最重要的，恐怕就是最后一点的锁定版本，通过 yarn 命令安装的依赖，会自动在项目的目录下生成 yarn.lock 这个文件，里面存放的是我们安装的所有模块的下载路径和对应的哈希，只要有 yarn.lock 在，即使出现了多个可以匹配的版本，使用 yarn 安装的依赖都与之前是相同的。这很好的弥补了 package.json 对依赖版本的模糊匹配造成的不足。

​        但在曾经的 npm 命令中，有一个 `npm shrikwrap` 命令，运行这个命令，会在根目录下生成一个 npm-shrikwrap.json 的文件。它的出现，也是为了锁定 package 的版本，同时存放了版本的信息校验，npm-shrikwrap.json 的出现，是早期 npm 没有自动锁定版本的时候出现的。

​        然而，在时代发展的今天，npm 也陪我们走到了 5.0 的版本，如今通过 `npm install` 安装的依赖，也会自动增加 npm-lock.json 这个文件。而出现 npm-lock.json 之后，两者是完全兼容的，甚至是可以直接将 npm-shrikwrap.json 的文件名直接改为 npm-lock.json ，这是一种完全的向后兼容的做法。

​        仔细对比一下 npm-lock.json 和 yarn.lock 这两个文件究竟存取的内容的区别：

​        yarn.lock 不是一个通用的格式，只能被 yarn 进行解析，文件结构依赖缩进，对于每一个 package ，分别存放了 版本、下载路径（锁定了哈希）以及这个模块的每一个依赖及版本号。

​        npm-lock.json 就是一个通用的 json 格式了，每一个依赖下面同样存放着版本、下载的路径（锁定了哈希）以及它所依赖的每一个模块的名称和版本号，值得一提的是，每一个依赖下面同时存放了模块内容的 sha-1 消息摘要，来保证模块下载结果的完整性。

#### 锁定版本带来的问题

​        package 锁定版本有时候也会带来一些小的问题。如果我们即将发布的 package 中包含了 yarn.lock 文件，那么当前的所有依赖都会直接读取被锁定的特定版本。在下面的例子中，a 的 package.json 中使用了宽泛匹配的 c 的版本为 ~0.1.0 ，最终内部锁定了 c 的版本为 0.1.0

```javascript
- a
  - c@0.1.0
```

​        如果这时候我们新使用了一个 b 模块，内部同样锁定了版本，但仍使用宽泛匹配的 ～0.2.0 ，但是由于 a 中的 c 被锁定了版本，这个时候 b 还是会安装锁定的 0.2.0。最终会使目录树变成：

```javascript
- a
- c@0.1.0
- b
  - c@0.2.0
```

​        使得两个宽泛匹配大版本的 c 的版本的冲突没有被解决，如果项目使用 yarn 安装的话，又会将两个版本的 c 重新写入 yarn.lock 造成版本冗余。

​        所以大多数的 package 会从上层解决这件事情。类似于 .gitignore，npm 也有 .npmignore ，它的作用就是在发布 package 时忽略一些文件，而大多数著名的 package 如 babel，react 等都会将 yarn.lock 加入 .npmignore ，目的就是在 package 安装的时候处理这一层 dependiences 的版本匹配问题。

#### npm 和类似 npm 的工具

因为 GFW 的原因，很多国内的开发者会设置 npm 的地址为国内镜像，例如使用淘宝官方的镜像。

```
npm install webpack --registry https://registry.npm.taobao.org
```

还可以修改 ~/.npmrc 文件进行设置：

```
// .npmrc
registry = https://registry.npm.taobao.org
```

> 在安装很多 package 例如 electron、node-sass 时，虽然设置 registry 为淘宝镜像，但是还是会失败或者安装不完整，因为这类 package 中会有 post install，虽然 package 的主体切换到了淘宝镜像进行下载，但是安装结束后会从第三方下载一些文件，例如 node-sass 就会继续下载一些依赖[node-sass 源码地址](https://github.com/sass/node-sass/blob/bb45385ae8bfe2c66ebb907ed9cc11e4772db8c3/lib/extensions.js#L241)，这种时候，就需要将对应依赖的 post install 地址也指向国内镜像。

​        如果你之前听说过淘宝镜像的话，那一定也听过 cnpm。cnpm 命令基本等同于 npm 命令，除了不能使用 `cnpm publish` 发布 package 以外，它们没有任何区别。

​        姑且可以把 npm，yarn，cnpm 算作一类命令，它们都是为了安装发布到 npmjs.com 上的 package 的一类命令，当然还有解决其他问题的相似命令。

**pnpm**: [https://github.com/pnpm/pnpm](https://github.com/pnpm/pnpm) 一个号称比 npm 和 yarn 都快的安装方式，因为相同版本的依赖只在硬盘上安装一次，同时可以解决多个项目多个 node_modules 本地内容冗余的问题。

**ied**：[https://github.com/alexanderGugel/ied](https://github.com/alexanderGugel/ied) 另一个 node 的包管理工具，同样号称比 npm 更快，并行安装依赖。

**ndm**: [https://github.com/720kb/ndm](https://github.com/720kb/ndm) 一个客户端可视化管理 node_modules 的工具，基于 electron。方便管理模块版本等等的问题。

#### 包管理中的 hook 函数

​        hook 函数在很多框架中都有提到，一般翻译为钩子函数。钩子函数的作用是在配置后，能够框架的特殊生命周期之前截获事件，并能触发特定的配置。

​        通过 npm 或者 yarn 安装的 package 也能通过配置，在安装过程中不同的生命周期触发不同的事件。

​        通过在 script 中配置特定的名称，可以在相应的生命周期触发这个脚本。例如，在 package.json 中我们配置 script 字段中一个属性为 prepublish ：

```javascript
{
	...
    "scripts": {
        "preinstall": "echo 'this is preinstall'"
    }
}
```

​        preinstall 就是一个特定的名称，当通过 npm install 安装这个 package 时，就会在安装之前触发这个脚本，就会在 shell 中输出一行 'this is preinstall'，在 npm 和 yarn 中有多个名称，一般以 pre 或者 post 开头，分别代表不同生命周期触发的脚本，下面列举了所有目前的钩子的名称：

1. prepublish: 在软件包打包和发布之前运行，以及在没有任何参数的本地 npm install 时运行
2. prepare：在打包和发布包之前运行，并且在没有任何参数的本地 npm install 时运行。触发时机在 prepublishOnly 之前，但是在 prepublish 之后。
3. prepublishOnly: 在准备和打包之前运行，仅在 npm 发布过程中触发。
4. prepack：在打包成 tar 文件之前运行（npm pack，npm publish 和通过 git 安装依赖时触发）
5. postpack：在打包成 tar 文件之后运行。
6. publish，postpublish：在包发布后运行。
7. preinstall：在安装软件包之前运行。
8. install，postinstall：在安装软件包后运行。
9. preuninstall，uninstall：卸载包之前运行。
10. postuninstall：卸载包后运行。
11. preversion：在处理包版本之前运行。
12. version：在处理包版本之后运行，但在提交前运行。
13. postversion：在打包包版本之后运行，在提交前运行。
14. pretest，test，posttest：由 npm test 命令运行。
15. prestop，stop，poststop： 由 npm stop 命令运行。
16. prestart，start，poststart：由 npm start 命令运行。
17. prerestart，restart，postrestart：由 npm restart 命令运行。注意：如果没有提供重新启动脚本，npm restart将运行停止并启动脚本。
18. preshrinkwrap，shrinkwrap，postshrinkwrap：由 npm shrinkwrap 命令运行。

#### 项目和 package 中的 dependencies 和 devDependencies

​        大多数有经验的开发者，都能熟练地区分 dependencies 和 devDependencies 的不同。构建过程中使用的依赖例如 webpack 或者 babel 相关的内容都需要放进 devDependencies，生产环境的依赖例如 jQuery 或者 react 都必须放进 dependencies。

​        对于模块来说，这种做法是必要的，因为当通过下载模块进项目时，也仅仅会安装相对应的 dependencies。但是这种区分对于前端项目其实不是必要的。 package.json 这种设计的初衷相信也不是为了适配前端这种原生不支持模块化的领域。

​        因为浏览器端原生不支持任何一种模块化的方案，所以在前端的开发过程中我们必须使用一些第三方的工具来使用模块化。所有的前端逻辑最终都需要运行模块化工具打包成为 bundle 。而真正的 bundle 文件才是运行在浏览器端的代码。在成为 bundle 的过程中，jQuery 和 babel 的作用是相同的，都是为了生成最终的 bundle 而处于构建环节中间的一员。唯一的区别在于 jQuery 最终存在于 bundle 中，而 babel 不会存在于 bundle 中。

​        真正需要严格区分 dependencies 和 devDependencies 的应该是 node 相关的项目。当我们 clone 一个新项目，只需要使用 `npm install --production`  仅仅安装生产依赖就能够跑起来，而不需要在安装 devDependencies 里面其他的依赖。这都得益于 node 原生支持 CommonJS 的模块化规范，对于前端来说，永远不可能直接通过  `npm install --production`  来运行项目，因为所有的依赖并不是直接从 node_modules 里面读取，而是早已经被 resolve 进了 bundle 中，这也就是前端和 node 最大的不同。

​        同时，虽然前端的 bundle 的产生相对麻烦，但是对于 debug 有一定的好处。除了一些 runtime 的 error 比如 “xxx is not defined” 或者 “xxx is not a function”。其他的所有 throw 出来的 error 都是会硬编码进 bundle 中的，一定程度上缓解了调试的难度。而在 node 项目中遇到的一些 throw 的 error ，有些情况下就得通过阅读源码进行解决，增加了调试成本。