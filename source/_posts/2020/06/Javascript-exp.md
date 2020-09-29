---
title: 函数式编程
date: 2020-06-16 19:34:24
tags:
  - javascript
categories:
  - 前端学习笔记
index_img: /img/Javascript.jpg
banner_img: https://wallroom.io/img/1920x1080/bg-02fdff7.jpg
---

# 函数式编程

## 1. 柯里化

函数式编程的基础，使用了高阶函数的思想，利用闭包把接受多个参数的函数封装成单参数函数，配合组合使用函数compose构成了函数式编程的重点技术

如何实现curry

```js
function curry(fn) {
return function curriedFunc(...args) {
    //* 使用fn.length获得fn定义时的参数数量，fn通过闭包被缓存
    if (args.length < fn.length) {
      //* 参数数量不够，应该返回柯里化后的函数
      return function(...localArgs) {
        //* 继续递归返回
        //* 把上一层递归的args和这个函数会接收到的参数合并
        return curriedFunc(...args.concat([ ...localArgs ]))
      }
    } else {
      //* 参数数量够了，可以运行fn了
      return fn(...args)
    }
  }
}
```

## 2. 组合函数

把一组需要组合的函数按照从右到左的顺序依次执行，注意要组合的函数必须都是单参数函数，也就是一元函数，所以常常需要和curry化配合使用

```js
//* 组合函数原理模拟
function compose(...funcs) {
  return function(value) {
    return funcs.reduceRight(function(acc, func) {
      return func(acc)
    }, value)
  }
}

const reverse = (arr) => arr.reverse()
const first = (arr) => arr[0]
const toUpperCase = (str) => str.toUpperCase()

const f = compose(toUpperCase, first, reverse)
console.log(f([ 'ant', 'bat', 'cab' ])) // C B A
```

## 3. Point-free

在组合函数的基础上，为了使函数更方便组合，可以约定让函数的第一个参数接受方法，第二个参数接受数据，这种方法优先的模式使得组合函数非常简单直接

当我们组合一系列的函数时，我们实际上就是在按一定的步骤组合方法，并没有提及数据，这和传统的过程式编程完全不同，也是函数式编程个人觉得最有魅力的地方，其简洁和老程序猿嗤之以鼻的面条代码刚好是两个极端

```js
// 可以很直观地知道我们先要执行的步骤，而完全没有提及变量或者数据本身
const firstLetterToUpper = fp.flowRight(
  fp.join('. '),
  fp.map(fp.flowRight(fp.toUpper, fp.first())),
  fp.split(' ')
)
```

lodash/fp这个库里实现了lodash里功能函数的Point-free版本(所以专门放在了fp这个模块下)

## 4. 函子Functor

### 4.1 基本概念

函子是我以前比较陌生的概念，基本从未看到其真正的使用

函子的本质类似面向对象里面的私有变量封装，传统的函数式编程要求函数为纯函数(Pure Function)，也就是没有任何副作用的函数

>副作用：函数执行时不引起其他变量的变化，函数没有中间中间状态

在实际应用中，除了功能单一的功能性函数，大部分业务函数都不是纯函数，副作用无可避免，所以函子的概念应运而生

函子把要被改变的变量和改变它的方法封装为一个class，和面向对象不同的是，这个改变它的方法并不是一个确定的函数，而是从外部传入一个函数

同时每次操作都会返回一个新的函子，这样就可以进行链式操作

```js
//* SECTION 容器 - 用于封装会被改变的值，以及改变它的方法
class Container {
  constructor(value) {
    this._value = value
  }

  //* 使用静态函数封装构造函数，这样外面调用的时候就不需要使用new
  static of(value) {
    return new Container(value)
  }

  //* map封装了任何试图改动_value的行为，并且总是返回一个新的容器
  map(fn) {
    return Container.of(fn(this._value))
  }
}
//* !SECTION

//* 测试 基本的函子
let r = Container.of(5) // 初始化
  .map((x) => x + 1) // map会返回一个新的容器，所以可以链式调用
  .map((x) => x * x) // 结果应该是36
```

### 4.2 Maybe函子

基本的函子里面没有判断value和传入的修改方法是否兼容，所以我们可以在容器内加入判断，比如value为空值的时候直接返回一个新函子(value值还是原来的)，并不进行计算

```js
class MayBe {
  static of(value) {
    return new MayBe(value)
  }

  constructor(value) {
    this._value = value
  }
  // 如果对空值变形的话直接返回 值为 null 的函子
  map(fn) {
    return this.isNothing() ? MayBe.of(null) : MayBe.of(fn(this._value))
  }
  isNothing() {
    return this._value === null || this._value === undefined
  }
}
// 传入具体值
MayBe.of('Hello World').map((x) => x.toUpperCase())
// 传入 null 的情况
MayBe.of(null).map((x) => x.toUpperCase())
// => MayBe { _value: null }
```

### 4.3 其他函子

常用的函子还有Either函子，IO函子，Task函子等

尤其是Task函子也是一种异步机制，具体可参考folktale函数式编程库

[folktale](https://folktale.origamitower.com/)

## 5. 一些题目

### 5.1 map函数和parseInt之间的故事

这也是一道经典的题目了

```js
const array = ['23', '8', '10']
console.log(array.map(parseInt))
//! 结果是[ 23, NaN, 2 ]
```

这是因为map函数在传递形参时，一共会传递三个参数 `callback(currentValue[, index[, array]])` 第二个是索引，第三个是数组本身
而parseInt其实可以接受两个参数string, radix，第二个radix表示进制，取值范围是2-36。

有意思的地方来了，radix大部分时候都没有人用，一般都会被留空，而当radix为 `0, undefined, null` 时，并不是默认以10进制来解析的：

[根据MDN的资料](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/parseInt#%E6%8F%8F%E8%BF%B0)

- 如果输入的 string以 "0x"或 "0x"（一个0，后面是小写或大写的X）开头，那么radix被假定为16，字符串的其余部分被解析为十六进制数。

- 如果输入的 string以 "0"（0）开头， radix被假定为8（八进制）或10（十进制）。具体选择哪一个radix取决于实现。ECMAScript 5 澄清了应该使用 10 (十进制)，但不是所有的浏览器都支持。因此，在使用 parseInt 时，一定要指定一个 radix。

- 如果输入的 string 以任何其他值开头， radix 是 10 (十进制)。

所以上面的例子，其真实的执行情况是：

```js
parseInt('23', 0) // parseInt没有第三个参数，所以map传递进来第三个参数array可以被忽略
parseInt('8', 1)
parseInt('10', 2)
```

第一个23是字符串，且不是以0x开头的特殊情况，所以使用默认10进制解析，答案为23
第二个因为parseInt的radix取值是2-36，1是非法值，所以结果为NaN
第三个10是字符串，以2进制解析，答案是2

这里的细节确实是普通程序猿绝对不会留意到的，这个题目出的的确刁钻
