---
title: Promise
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-06-30 23:00:37
tags:
  - javascript
categories:
  - 编程技巧
  - 前端学习笔记
index_img: /img/promise.png
---

# Promise

## 1. 为什么不能把Promise.resolve当做函数变量来使用

如果试图直接调用：

```js
FunctionThatNeedsCallback(Prmose.resolve) // TypeError: PromiseResolve called on non-object
```

最简单的复现方法是：

```js
Promise.resolve(1).then(Promise.resolve) // TypeError: PromiseResolve called on non-object
```

因为Promise.resolve是一个需要context的函数，好比一个里面使用了this的函数，是不允许脱离context使用的。
所以要么直接在一个对象上使用 `v => Prmose.resolve(v)`，要么就要手动绑定 `Promise.resolve.bind(Promise)`

## 2. Promise中的值穿透

和上面那个例子相关，假如有下面这样的代码，结果会打印出什么呢？

```js
Promise.resolve(1).then(2).then(console.log)
```

答案是1，这是因为如果then的参数不是一个函数，就会把上一层传入的值直接传递给下一层 (类似直接 `return this`)，这就是值穿透现象。

通过具体的代码实现，可以比较容易地理解：

```js
successCallback =
  typeof successCallback === 'function'
    ? successCallback
    : (value) => value // 如果callback不是函数，则返回传入的值
failCallback =
  typeof failCallback === 'function'
    ? failCallback
    : (reason) => reason // 如果callback不是函数，则返回传入的值
```

## 3. await的执行顺序

在异步编程里面常见的知识点有两个，一个是微任务和宏任务的异步执行顺序，另一个就是以Promise为主的异步理解。其中个人觉得最容易被错误理解的不是Promise本身，而是await这个语法糖。众所周知，async-await关键字本质上还是generator和Promise，一个async函数默认会返回一个Promise

```js
async function fn1() {
  console.log('fn1 start')
}

let r = fn1() // fn1 start
console.log(r) // Promise { undefined }
```

如果没有await，在函数内部即使有异步操作也不会以异步的方式执行，await就好比generator中的yield关键字，我们下面用一个完整的例子来具体分析await

```js
async function fn1() {
  console.log('fn1 start')
  fn2()
  console.log('fn1 end')
}

async function fn2() {
  console.log('fn2 start')
  await fn3()
  console.log('fn2 end')
}

async function fn3() {
  console.log('fn3 start')
  return 1
}

fn1()

Promise.resolve('Promise').then((value) => console.log(value))

console.log('finally')
```

当fn1当中不使用await时，执行顺序为：

```text
fn1 start
fn2 start
fn3 start
fn1 end
finally
fn2 end
Promise
```

我们可以注意到fn1 end和fn2 end的输出区别，在fn2中由于使用了await，后面的log语句被放到了Promise中执行，也就是进入了异步队列，因此最后主程序的finally打印之后才打印fn2 end

而fn1 end是直接打印的，这正是因为我们没有在fn1中使用await，所以fn2()后面的log是以同步的方式运行的。理解了这一点之后，我们可以很容易发现await其实就是Promise.resolve，两者在语义上是一致的，任何在await之后的代码都会被放到Promise中，在then里面才会运行

[**值得注意的是，关于await具体的实现V8引擎也几经更改，可参考这个知乎回答**](https://www.zhihu.com/question/268007969)

```js
async function fn1() {
  console.log('fn1 start')
  await fn2()
  console.log('fn1 end')
}
// 等价于
async function fn1() {
  console.log('fn1 start')
  Promise.resolve(fn2())
    .then(console.log('fn1 end'))
}
```

当我们把await和Promise.resolve联系起来之后，就会很容易理解一个面试题的常考点，就是await同一行的函数会被立即执行，之后的代码才会被放到then里面，所以fn2 start和fn3 start都是马上就打印出来了

如果我们在`fn2()`前面加个await的话，打印顺序就不一样了

```text
fn1 start
fn2 start
fn3 start
finally
fn2 end
Promise
fn1 end
```

可以发现fn1 end现在最后才会打印，甚至晚于fn2 end。这是也因为await同一行的函数会被立即执行，就好比`Promise.resolve(fn2())`里的fn2是马上执行的，其返回值就是resolve出去的对象一样，当执行fn2时，fn2内部也有一个await，所以fn3被马上执行，而后面的打印函数被放入了then里面，所以这里fn2中先产生了Promise，率先进入异步队列。而当执行栈再次回到fn1之后，才把fn1后面的log放入then，要晚于fn2
