---
title: 下一代状态管理工具 immer 简介及源码解析
date: '2018-2-1'
---
JS 里面的变量类型可以大致分为基本类型和引用类型。在使用过程中，引用类型经常会产生一些无法意识到的副作用，所以在现代 JS 开发过程中，有经验的开发者都会在特定位置有意识的写下断开引用的不可变数据类型。

```js
// 引用带来的副作用
var a = [{ val: 1 }]
var b = a.map(item => item.val = 2)

// 期望：b 的每一个元素的 val 值变为 2
console.log(a[0].val) // 2
```

<!--more-->

<div class="tip">
  从上述例子我们可以发现，本意是只想让 `b` 中的每一个元素的值变为 2 ，但却无意中改掉了 `a` 中每一个元素的结果，这是不符合预期的。接下来如果某个地方使用到了 `a` ，很容易发生一些我们难以预料并且难以 debug 的 bug。
</div>

在发现这样的问题之后，解决方案也很简单。一般来说当需要传递一个引用类型的变量（例如对象）进一个函数时，我们可以使用 `Object.assign` 或者 `...` 对对象进行解构，成功断掉一层的引用。

例如上面的问题我们可以改用下面的这种写法：

```js
var a = [{ val: 1 }]
var b = a.map(item => ({ ...item, val: 2 }))

console.log(a[0].val) // 1
console.log(b[0].val) // 2
```

但是这样做会有另外一个问题，无论是 `Object.assign` 还是 `...` 的解构操作，断掉的引用也只是一层，如果对象嵌套超过一层，这样做还是有一定的风险。

```js
// 深层次的对象嵌套
var a = [{
    val: 1,
    desc: { text: 'a' }
  }]
var b = a.map(item => ({ ...item, val: 2 }))

console.log(a === b)           // false
console.log(a.desc === b.desc) // true
```

`a.desc === b.desc` 表达式的结果仍为 true，这说明在程序内部 `a.desc` 和 `b.desc` 仍然指向相同的引用。如果后面的代码一不小心在一个函数内部直接通过 `b.desc` 进行赋值，就一定会改变具有相同引用的 `a.desc` 部分的结果，这当然是不符合我们的预期的。

所以在这之后，大多数情况下我们会考虑 **深拷贝** 这样的操作来完全避免上面遇到的所有问题。**深拷贝**，顾名思义就是在遍历过程中，如果遇到了可能出现引用的数据类型，就会递归的完全创建一个新的类型。

```js
// 一个简单的深拷贝函数，去掉了一些胶水部分
// 用户态输入一定是一个 Plain Object，并且所有 value 也是 Plain Object
function deepClone(obj) {
  const keys = Object.keys(obj)
  return keys.reduce((memo, current) => {
    const value = obj[current]
    if (typeof value === 'object') {
      return {
        ...memo,
        [current]: deepClone(value),
      }
    }
    return {
      ...memo,
      [current]: value,
    }
  }, {})
}
```

用上面的 deepClone 函数进行简单测试

```js
var a = {
  val: 1,
  desc: {
    text: 'a',
  },
}
var b = deepClone(a)

b.val = 2
console.log(a.val) // 1
console.log(b.val) // 2

b.desc.text = 'b'
console.log(a.desc.text) // 'a'
console.log(b.desc.text) // 'b'
```

上面的这个 `deepClone` 可以满足简单的需求，但是真正在生产工作中，我们需要考虑非常多的因素。举例来说：

* key 里面 getter，setter 以及原型链上的内容如何处理？
* value 是一个 Symbol 如何处理？
* value 是其他非 Plain Object 如何处理？
* value 内部出现了一些循环引用如何处理？

因为有太多不确定因素，所以在真正的工程实践中，还是推荐大家使用大型开源项目里面的工具函数。比较常用的为大家所熟知的就是 `lodash.cloneDeep`，无论是安全性还是效果都有所保障。

其实，这种去除引用数据类型副作用的数据的概念我们称作 immutable ，意为不可变的数据，其实理解为不可变关系更为恰当。每当我们创建一个被 `deepClone` 过的数据，新的数据进行有**副作用** (side effect) 的操作都不会影响到之前的数据，这也就是 immutable 的精髓和本质。

<div class="tip">
  这里的**副作用**不只局限于通过「点」操作对属性赋值。例如 array 里面的 push,pop,splice 等操作都是会改变原来的数据，这些操作都算是非 immutable。
</div>

然而 deepClone 这种函数虽然断绝了引用关系实现了 immutable，但是相对来说开销太大。所以在 2014 年，facebook 的 immutable-js 横空出世，即保证了数据间的 immutable ，又兼顾了性能。

## immutable-js 简介

immutable-js 使用了另一套数据结构的 API ，与我们的常见操作有些许不同，它将所有的原生对象都会转化成 immutable-js 的内部对象，并且任何操作最终都会返回一个新的 immutable 的值。

上面的例子使用 immutable-js 就需要这样改造一下：

```js
const { fromJS } = require('immutable')
const data = {
  val: 1,
  desc: {
    text: 'a',
  },
}

const a = fromJS(data)

const b = a.set('val', 2)
console.log(a.get('val')) // 1
console.log(b.get('val')) // 2

const pathToText = ['desc', 'text']
const c = a.setIn([...pathToText], 'c')
console.log(a.getIn([...pathToText])) // 'a'
console.log(c.getIn([...pathToText])) // 'c'
```

对于性能方面，immutable-js 也有它的优势，举个简单的例子：

```js
const { fromJS } = require('immutable')
const data = {
  content: {
    time: '2018-02-01',
    val: 'Hello World',
  },
  desc: {
    text: 'a',
  },
}

const a = fromJS(data)
const b = a.setIn(['desc', 'text'], 'b')
console.log(b.get('desc') === a.get('desc'))       // false
console.log(b.get('content') === a.get('content')) // true

const c = a.toJS()
const d = b.toJS()
console.log(c.desc === d.desc)       // false
console.log(c.content === d.content) // false
```

从上面的例子可以看出来，在 immutable-js 的数据结构中，深层次的对象在没有修改的情况下仍然能够保证严格相等，这也是 immutable-js 的另一个特点 「深层嵌套对象的结构共享」。即嵌套对象在没有改动前仍然在内部保持着之前的引用，修改后断开引用，但是却不会影响之前的结果。

经常使用 React 的同学肯定也对 immutable-js 不陌生，这也就是为什么 immutable-js 会极大提高 React 页面性能的原因之一了。

当然能够达到 immutable 效果的当然不只这几个个例，这篇文章我主要想介绍实现 immutable 的库其实是 immer。

## immer 简介

immer 的作者同时也是 mobx 的作者，一个看起来非常感性的中年大叔。mobx 又像是把 Vue 的一套东西融合进了 React，已经在社区取得了不错的反响。immer 则是他在 immutable 方面所做的另一个实践，在 2018-02-01，immer 成功发布了 **1.0.0** 版本，我差不多在一个月前开始关注这个项目，所以大清早看到作者在 twitter 上发的通告，有感而发今天写下这篇文章，算是简单介绍一下 immer 这个 immutable 框架的使用以及内部简单的实现原理。

与 immutable-js 最大的不同，immer 是使用原生数据结构的 API 而不是内置的 API，举个简单例子：

```js
const produce = require('immer')

const state = {
  done: false,
  val: 'string',
}

const newState = produce(state, (draft) => {
  draft.done = true
})

console.log(state.done)    // false
console.log(newState.done) // true
```

所有**具有副作用的逻辑**都可以放进 produce 的第二个参数的函数内部进行处理。在这个函数内部对原来的数据进行任何操作，都不会对原对象产生任何影响。

简单介绍完使用之后，下面就开始简单介绍它的内部实现。

<div class="tip">
  不过在这之前，想先通过上面的例子简单的思考一下。

  通过文章最开始的例子我们就能明白，给函数传入一个对象，直接通过「点」操作符对对象内部的一个属性进行更改是一定会改变外面的结果的。而上面的这个例子中，`draft` 参数传入进去，与 `state` 一样也有 done 这个属性，但是在通过 `draft.done` 改变值之后，原来的 `state.done` 并没有发生改变。

  其实到这里，结合之前研究 vue 源码的经验，大概就能笃定，这里一定用了 `Object.defineProperty`，对数据的结果做了一部分劫持，从而做了一些新的操作完成目的。
</div>

## immer 原理解析

真正翻开源码，诚然里面确实有 defineProperty 的身影，不过在另一个核心的文件中，用了一种新的方式，那就是 ES6 中新增的 Proxy 对象。而在日常的业务过程中，JSer 在编码习惯上并不一定经常使用 Proxy 对象，所以这里简单且快速的介绍一下它的使用。

Proxy 对象接受两个参数，第一个参数是需要操作的对象，第二个参数是设置对应拦截的属性，这里的属性同样也支持 get，set 等等，也就是劫持了对应元素的读和写，能够在其中进行一些操作，最终返回一个 Proxy 对象。

```js
const proxy = new Proxy({}, {
  get(target, key) {
    console.log('proxy get key', key)
  },
  set(target, key, value) {
    console.log('value', value)
  }
})

proxy.info     // 'proxy get key info'
proxy.info = 1 // 'value 1'
```

上面这个例子中传入的第一个参数是一个空对象，当然我们可以用其他已有内容的对象代替它。

immer 的做法就是维护一份 state 在内部，劫持所有操作，内部来判断是否有变化从而最终决定如何返回。下面这个例子就是一个构造函数，如果将它的实例传入 Proxy 对象作为第一个参数，就能够后面的处理对象中使用其中的方法：

```js
class Store {
  constructor(state) {
    this.modified = false
    this.source = state
    this.copy = null
  }
  get(key) {
    if (!this.modified) return this.source[key]
    return this.copy[key]
  }
  set(key, value) {
    if (!this.modified) this.modifing()
    return this.copy[key] = value
  }
  modifing() {
    if (this.modified) return
    this.modified = true
    this.copy = Array.isArray(this.source)
      ? this.source.slice()
      : { ...this.source }
  }
}
```

上面这个 Store 构造函数相比源代码省略了很多判断的部分。实例上面有 `modified`，`source`，`copy` 三个属性，有 `get`，`set`，`modifing` 三个方法。`modified` 作为内置的 flag，判断如何进行设置和返回。

里面最关键的就应该是 `modifing` 这个函数，如果触发了 setter 并且之前没有改动过的话，就会手动将 `modified` 这个 flag 设置为 `true`，并且手动通过原生的 API 实现一层 immutable。

对于 Proxy 的第二个参数，在简版的实现中，我们只是简单做一层转发，任何对元素的读取和写入都转发到 store 实例内部方法去处理。

```js
const PROXY_FLAG = '@@SYMBOL_PROXY_FLAG'
const handler = {
  get(target, key) {
    if (key === PROXY_FLAG) return target
    return target.get(key)
  },
  set(target, key, value) {
    return target.set(key, value)
  },
}
```

<div class="tip">这里在 getter 里面加一个 flag 的目的就在于将来从 proxy 对象中获取 store 实例更加方便。</div>

最终我们能够完成这个 produce 函数，创建 store 实例后创建 proxy 实例。然后将创建的 proxy 实例传入第二个函数中去。这样无论在内部做怎样有副作用的事情，最终都会在 store 实例内部将它解决。最终得到了修改之后的 proxy 对象，而 proxy 对象内部已经维护了两份 state ，通过判断 modified 的值来确定究竟返回哪一份。

```js
function produce(state, producer) {
  const store = new Store(state)
  const proxy = new Proxy(store, handler)

  producer(proxy)

  const newState = proxy[PROXY_FLAG]
  if (newState.modified) return newState.copy
  return newState.source
}
```

这样，一个分割成 Store 构造函数，handler 处理对象和 produce 处理 state 这三个模块的最简版就完成了，将它们组合起来就是一个最最最 tiny 版的 immer ，里面去除了很多不必要的校验和冗余的变量。但真正的 immer 内部也有其他的功能，例如上面提到的**深层嵌套对象的结构化共享**等等。

如果真的对这部分逻辑感兴趣的话，可以直接读作者的源码 [immer](https://github.com/mweststrate/immer) 或者看我自己写的继续深入了一层的源码 [tiny-immer](https://github.com/loatheb/tiny-all-over-the-world/blob/master/immutable/immer/index.js) 。写这篇文章的目的并不是一开始就让读者深入进去，而是在以后小型项目有了另一个 immutable 的选择。

## 性能

性能方面，就用 immer 官方 README 里面的介绍来说明情况。

这是一个关于 immer 性能的简单测试。这个测试使用了 100000 个组件元素，并且更新其中的 10000 个。freeze 表示状态树在生成之后已被冻结。这是一个最佳的开发实践，因为它可以防止开发人员意外修改状态树。

![performance](/images/immer/performance.png)

通过上图的观察，基本可以得出：

* 从 immer 的角度来看，这个性能环境比其他框架和库要恶劣的多，因为它必须代理的根节点相对于其余的数据集来说大得多

* 从 mutate 和 deepclone 来看，mutate 基准确定了数据更改费用的基线，没有不可变性（或深度克隆情况下的结构共享）

* 使用 Proxy 的 immer 大概是手写 reducer 的两倍，当然这在实践中可以忽略不计

* immer 大致和 immutable-js 一样快。但是，immutable-js 最后经常需要 toJS 操作，这里的性能的开销是很大的。例如将不可变的 JS 对象转换回普通的对象，将它们传递给组件中，或着通过网络传输等等（还有将从例如服务器接收到的数据转换为 immutable-js 内置对象的前期成本）

* immer 的 ES5 实现速度明显较慢。对于大多数的 reducer 来说，这并不重要，因为处理大量数据的 reducer 可以完全不（或者仅部分）使用 immer 的 produce 函数。幸运的是，immer 完全支持这种选择性加入的情况

* 在 freeze 的版本中，只有 mutate，deepclone 和原生 reducer 才能够递归地冻结全状态树，而其他测试用例只冻结树的修改部分

## 写在后面

其实纵观 immer 的实现，核心的原理就是放在了对对象读写的劫持，从表现形式上立刻就能让人想到 vue ，mobx 从核心原理上来说也是对对象的读写劫持，最近有另一篇非常火的文章 -- 如何让 `(a == 1 && a == 2 && a == 3)` 为 true，也相信不少的小伙伴读过，除了那个肉眼不可见字符的答案，其他答案也算是对对象的读写劫持从而达到目标。

所以说在 JS 中，很多知识相辅相成，理论上有多少种方式能让 (a == 1 && a == 2 && a == 3) 为 true，理论上就会有多少种 MVVM 的组成方式，甚至就有多少种方法能够实现这样的 immutable。所以每一个小小的知识点，未来都可能影响前端的发展。

最近我个人也在整理自己感兴趣的开源框架的最小实现，项目代码现在都放在 [tiny-all-the-world](https://github.com/loatheb/tiny-all-over-the-world) 。它的目的就在于把前端常用框架类库实现一个没有第三方依赖的简易版，希望大家多多帮忙 review，也希望大家也能够从其中有所学习。
