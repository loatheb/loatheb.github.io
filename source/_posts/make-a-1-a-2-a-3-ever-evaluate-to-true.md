---
title: 如何让 (a == 1 && a == 2 && a == 3) 返回 true
date: '2018-1-18'
---
前两天在网上看到了一道很有趣的题目，题目大意为：**JS 环境下，如何让 `a == 1 && a == 2 && a == 3` 这个表达式返回 `true` ？**。

这道题目乍看之下似乎不太可能，因为在正常情况下，一个变量的值如果没有手动修改，在一个表达式中是不会变化的。当时我也冥思苦想很久，甚至一度怀疑这道题目的答案就是 **不能**。直到在 stackoverflow 上面竟然真的发现了解法 [can-a-1-a-2-a-3-ever-evaluate-to-true](https://stackoverflow.com/questions/48270127/can-a-1-a-2-a-3-ever-evaluate-to-true)。

让这个表达式成为 `true` 的关键就在于这里的宽松相等，JS 在处理宽松相等时会对一些变量进行隐式转换。在这种隐式转换的作用下，真的可以让一个变量在一个表达式中变成不同的值。

<!-- more -->

## 宽松相等下的真值表

最高票答案给出的解法为：

```js
const a = {
  i: 1,
  toString: function () {
    return a.i++;
  }
}

if (a == 1 && a == 2 && a == 3) {
  console.log('Hello World!');
}
```

看到这个答案，我才恍然大悟，这道题目的考点原来是 JS 获取一个变量所需要做的操作以及其中一些细节。在 JS 中有 `===` 和 `==` 两种方式来判断两个变量是否相等。对 JS 稍有了解的人都知道，=== 是严格相等，不仅需要两个变量的值相同，还需要类型也相同，而 == 则是宽松下的相等，只需要值相同就能够判断相等，宽松相等是严格相等的子集。所以在 JS 中，严格相等的两个变量一定也是宽松相等的，但是宽松相等的两个变量，大多数情况下并不是严格相等的。例如：

```js
null == undefined // true
null === undefined // false

1 == '1' // true
1 === '1' // false
```

这也就出现了 JS 特有的，变量宽松相等判断的真值表，里面列举了所有在宽松相等比较的情况下，两种变量可能出现的类型：

![/images/truth-table.png](/images/truth-table.png)

在上面的表格中，ToNumber(A) 尝试在比较前将参数 A 转换为数字，这与 +A（单目运算符+）的效果相同。ToPrimitive(A) 通过尝试依次调用 A 的 A.toString() 和 A.valueOf() 方法，将参数 A 转换为原始值（Primitive）。

从上图中我们可以看到，当操作数 B 类型为 Number 时，如果希望在宽松相等的情况下整个表达式的结果返回 true，操作数 A 必须满足下面三个条件之一：

1. 操作数 A 类型为 String，并且调用 +A 的结果与 B 严格相等
2. 操作数 A 类型为 Boolean，并且调用 +A 的结果与 B 严格相等
3. 操作数 A 类型为 Object，并且调用 toString 或者 ValueOf 返回的结果与 B 严格相等

在这里，如果我们要改变 +A 操作的结果相对来说比较困难，因为我们很难在 JS 中去重载 + 操作符的运算。但是在第三种情况下，使 A 的类型为 Object，调用 toString 或 ValueOf 结果与 B 严格相等让我们自己实现就容易的多。

所以上面的答案就是新建了一个对象 `a` ，并有 `toString` 方法，当 JS 引擎每次读取 `a` 的值的时候，发现需要进行宽松判断一个对象和一个数字之间的结果，对于对象就会执行这里的 `toString` 方法，在这个方法内部，我们每次增加另一个变量的值并返回，就能够在这条表达式中使得 `a` 的结果有不同的值。

同理，换一种写法，`a` 为 `Object` ，使用 `valueOf` 也是可以完成目标的：

```js
cosnt a = {
  i: 1,
  valueOf() {
    return this.i++
  }
}

if (a == 1 && a == 2 && a == 3) {
  console.log('Hello World!');
}
```

## 宽松相等下的 Proxy 对象

有了上面的思路，下面实现起来就容易的多。在 ES6 中 JS 新增了 `Proxy` 对象，能够对一个对象进行劫持，接受两个参数，第一个是需要被劫持的对象，第二个参数也是一个对象，内部也可以配置每一个元素的 get 方法：

```js
var a = new Proxy({ i: 1 }, {
  get(target) { return () => target.i++ }
});

if (a == 1 && a == 2 && a == 3) {
  console.log('Hello World!');
}
```

同样的，Proxy 对象默认的 `toString` 和 `valueOf` 方法会返回这个被 getter 劫持过的结果，也能够在宽松相等的条件下满足题意。

## 严格相等下的实现

上面的这几种做法，都是利用了宽松相等条件下，JS 里的一些特殊表现来实现的，放在 === 这种严格相等的条件下就不能够满足，因为严格相等的条件下不会对两个操作数做任何处理，直接比较它们值的大小，这样上面的做法就不能成功。

但是这种做法给我们提供了很好的思路，在处理类似的问题的时候，就可以从 JS 获取一个变量执行过程中出发，来进行思考。那么接下来，如果题目中的宽松相等换成了严格相等，这样的例子还存在么？

```js
if (a === 1 && a === 2 && a === 3) {
  console.log('Hello World!');
}
```

答案是显然的，这一次当然不能用 hack 对象或者 Proxy 的 `toString` 或者 `ValueOf` 方法来做。从 JS 获取变量的过程入手，理所当然的立马能想到的就是数据的 getter 和 setter 方法，通过这样的 hack ，肯定也能达到题目的严格相等的要求。

在 ES5 之后，`Object` 新增 `defineProperty` 方法，它会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回这个对象，对于定义的这个对象有两种描述它的状态，一种称之为数据描述符，一种被称为存取描述符，分别举一个例子：

```js
var a = {}
Object.defineProperty(a, 'value', {
  enumerable: false,
  configurable: false,
  writable: false,
  value: "static"
})
```

这四个数据描述服分别作用是 enumerable 判断是否可以枚举，configurable 判断当前属性是否之后再被更改描述符，writable 判断是否可以继续赋值，value 判断这个结果的值。

经过这样的操作之后，a 对象下就有了 value 这个 key ，他被赋予不可继续赋值，不可继续配置，不能被枚举，值为 'static'，我们可以通过 a.value 拿到这里的 'static'，但是不能继续通过 `a.value = 'relative'` 来继续赋值。

同样的，设置存取描述符也是四个属性：

```js
var a = { i: 1 }
Object.defineProperty(a, 'value', {
  enumerable: false,
  configurable: false,
  get() { return a.i }
  set() { a.i++ }
})
```

这里设置时就没有配置 writable 和 value 属性，转而配置了 get 和 set 方法，在这两种配置中，get set 方法和 writable value 是不能共存的，否则就会抛出异常。类似上面这样的设置，当我们访问 a.value 时就会调用 get 方法，当我们通过 `a.value = 'test'` 时，就会执行 set 方法。

所以回归到题目中，当我们访问一个被设置了存取描述符的元素时，如果在 get 方法里面做一些操作，就能巧妙的使得最终的结果达到预期：

```js
var i = 1
Object.defineProperty(window, 'a', {
  get() { return i++ }
})

if (a === 1 && a === 2 && a === 3) {
  console.log('Hello World!');
}
```

同时，这种劫持 getter 和 setter 的方法本质上是执行了一个函数，内部除了用自增变量，还可以有更多的方法：

```js
const value = function* () {
  let i = 1
  while(true) yield i++
}()

Object.defineProperty(window, 'a', {
  get() {
    return value.next().value
  }
})

if (a === 1 && a === 2 && a === 3) {
  console.log('Hello World!');
}
```

## 总结

对于严格相等的情况，一般来说只能通过劫持数据的 getter 来进行操作，但是里面具体操作的方法在上面列举的就有很多。

对于宽松相等的情况，除了劫持 getter 以外，因为宽松相等 JS 引擎的缘故，还能用 Object ， Proxy 对象的 valueOf 和 toString 方法达到目的。

当然，在 stackoverflow 中有人提出了另一种做法，在 a 变量的前后用不同的字符达到目的，原理就在于某些字符在肉眼条件下是不可见的，所以虽然看起来都是 a ，但变量实际上的不同的，也能达到题目的要求，不过这就不在本文的讨论范围之内了。
