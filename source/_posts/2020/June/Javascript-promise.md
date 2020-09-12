---
title: Promise 异步操作的九阳神功
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-06-30 23:00:37
tags:
  - javascript
categories:
  - 前端学习笔记
index_img: /img/promise.png
---

# Promise 异步操作的九阳神功

## 1. Promise中的值穿透

假如有下面这样的代码，结果会打印出什么呢？

```js
Promise.resolve(1).then(2).then(console.log)
```

答案是1，这是因为如果`then`的参数不是一个函数，就会把上一层传入的值直接传递给下一层 (类似直接 `return this`)，这就是值穿透现象。

通过具体的代码实现，可以比较容易地理解：

```js
successCallback =
  typeof successCallback === 'function'
    ? successCallback
    : (value) => value // 如果callback不是函数，则构造一个函数返回传入的值
failCallback =
  typeof failCallback === 'function'
    ? failCallback
    : (reason) => reason // 如果callback不是函数，则构造一个函数返回传入的值
```

## 2. await的执行顺序

### 2.1 理解await

在异步编程里面常见的知识点有两个，一个是微任务和宏任务的异步执行顺序，另一个就是以`Promise`为主的异步理解。其中个人觉得最容易被错误理解的不是`Promise`本身，而是`await`这个语法糖。众所周知，`await`关键字本质上还是生成器`generator`（而一个`async`函数默认会返回一个`Promise`）。

```js
async function fn1() {
  console.log('fn1 start')
}

let r = fn1() // fn1 start
console.log(r) // Promise { undefined }
```

如果没有`await`，在函数内部即使有异步操作也不会以异步的方式执行，`await`就好比`generator`中的`yield`关键字，我们下面用一个完整的例子来具体分析`await`。

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

我们可以注意到fn1 end和fn2 end的输出区别，在fn2中由于使用了await，后面的log语句被放到了`Promise`中执行，也就是进入了异步队列，因此最后主程序的finally打印之后才打印fn2 end。

而fn1 end是直接打印的，这正是因为我们没有在fn1中使用`await`，所以fn2()后面的log是以同步的方式运行的。理解了这一点之后，我们可以很容易发现`await`其实就是`Promise.resolve`，两者在语义上是一致的，任何在`await`之后的代码都会被放到`Promise`中，在`then`里面才会运行。

[**值得注意的是，关于await具体的实现V8引擎也几经更改，可参考这个知乎回答**](https://www.zhihu.com/question/268007969)

```js
async function fn1() {
  console.log('fn1 start')
  await fn2()
  console.log('fn1 end')
}
// 等价于（注意仅仅是语义上等价，await并不是用Promise.resolve来实现的！）
async function fn1() {
  console.log('fn1 start')
  Promise.resolve(fn2())
    .then(console.log('fn1 end'))
}
```

当我们把`await`和`Promise.resolve`联系起来之后，就会很容易理解一个面试题的常考点，就是`await`同一行的函数会被立即执行，之后的代码才会被放到`then`里面，所以fn2 start和fn3 start都是马上就打印出来了。

如果我们在`fn2()`前面加个`await`的话，打印顺序就不一样了：

```text
fn1 start
fn2 start
fn3 start
finally
fn2 end
Promise
fn1 end
```

可以发现fn1 end现在最后才会打印，甚至晚于fn2 end。这是也因为`await`同一行的函数会被立即执行，就好比`Promise.resolve(fn2())`里的fn2是马上执行的，其返回值就是resolve出去的对象一样。

当执行fn2时，fn2内部也有一个`await`，所以fn3被马上执行，而后面的打印函数被放入了`then`里面，所以这里fn2中先产生了`Promise`，率先进入异步队列。而当执行栈再次回到fn1之后，才把fn1后面的log放入`then`，要晚于fn2。

最后为什么fn1 end要晚于Promise打印？因为fn2由于内部有`await`还没有执行完，主线程回到fn1的时候`await fn2()`产生的`Promise`还没能resolve，所以并不会把后面的代码放入异步队列（具体会在第4节的`Promise`源码部分），然后主线程继续执行后面的同步代码，于是Promise的打打印先进入异步队列。

### 2.2 await和Promise.resolve的区别

上一节我们提到了`await`和`Promise.resolve`在语义上的相似之处，但是也强调了两者并不完全等同，最重要的区别是由于`await`本质上是生成器，所以可以真正做到“暂停”程序，而`Promise.resolve`则没有这个能力。

来看一个例子：

```js
async function fn1 () {
  console.log('fn1 start')
  Promise.resolve(fn2()).then((value) => {
    console.log(value)
    console.log('fn1 end')
  })
}

async function fn2 () {
  console.log('fn2 start')
  Promise.resolve(fn3()).then(() => {
    console.log('fn2 end')
    return 2
  })
}

async function fn3 () {
  console.log('fn3 start')
}

fn1()

Promise.resolve('Promise').then((value) => console.log(value))

console.log('finally')
```

这个例子和2.1中的例题基本一样，区别在于我们不使用`await`而是用`Promise.resolve`代替，来看看结果：

```text
fn1 start
fn2 start
fn3 start
finally
fn2 end
undefined
fn1 end
Promise
```

这里可以发现顺序和2.1有所出入

1. 首先fn1 end先于Promise打印，也就是说其实fn1并没有真正“等待”fn2执行完毕就已经把后面的then放入异步队列了。
2. fn1中打印的fn2的返回值是`undefined`，从侧面佐证了fn1没有真正“等待”fn2，否则返回值应该是2

所以可以发现`Promise.resolve`并没有“暂停”代码执行的能力，一个函数内部如果有`Promise.resolve`也无法真正做到“延迟”返回，只不过是把返回放到了异步队列里，对于接受返回值的调用者来说返回仍然是同步代码决定的，而因为同步代码部分没有定义返回值，所以默认返回`undefined`，fn2就是这样的一个例子。

所以对于fn1中的`Promise.resolve(fn2())`而言，fn2的返回值仅仅取决于它的同步代码，`Promise.resolve`不会去等待fn2中的异步任务。

我们再把fn2用`await`改写一下，结果就会迥然不同：

```js
async function fn2 () {
  console.log('fn2 start')
  // Promise.resolve(fn3()).then(() => {
  //   console.log('fn2 end')
  //   return 2
  // })
  await fn3()
  return 2
}
```

现在的结果是：

```text
fn1 start
fn2 start
fn3 start
finally
Promise
2
fn1 end
```

`await`背后的生成器可以真正做到“暂停”代码。

我们知道`async`函数返回的是一个`Promise`，所以`Promise.resolve(fn2())`里面fn2本身返回了一个`Promise`，如果这个`Promise`状态是fulfilled了才会把后续的`then`放入异步队列，而这时由于fn2中使用了`await`，fn2在运行完之前这个返回的`Promise`一直是pending状态，所以`Promise.resolve(fn2()).then()`后面的`then`无法马上进入异步队列，只能等待fn2继续执行完`await`之后的所有代码。

这也是`await`强大的地方，因为它从真正意义上做到了“等待异步执行”，而不是仅仅把后续代码放入异步队列而已！这个区别非常重要，值得专门列出来：

- fn1里面使用

  ```js
  Promise.resolve(foo()).then(bar())
  ```

  把`bar()`放入异步队列就完事儿了，fn1会主动执行完剩下所有的同步代码然后返回，`then`里面的回调`bar()`不影响fn1是否执行完成，外部调用fn1的调用者会接收到一个已经fulfilled的`Promise`，并且这个`Promise`中resolve的值就是fn1同步代码部分的返回值（如果没有就是`undefined`）

- fn2里面使用

  ```js
  await foo()
  bar()
  ```

  fn2会等待`foo()`执行完毕，剩下的所有同步代码包括`bar()`被阻塞，fn2处于“暂停”状态，没有执行完毕，外面调用fn2的调用者会接收到一个pending中的`Promise`。

最后值得一提的是，`Promise.resolve`有一个特性，如果resolve的值本来就是一个`Promise`，就会直接返回这个`Promise`而不是新生成一个，否则`Promise.resolve(fn2()).then`后面`then`接收到的只会是fn2返回的`Promise`，而不是里面resolve的值。

## 3. 深入理解then

下面我们来看更复杂一些的例子

```js
new Promise((resolve, reject) => {
  console.log('外部promise')
  resolve()
})
  .then(() => {
    console.log('外部第一个then')
    new Promise((resolve, reject) => {
      console.log('内部promise')
      resolve()
    })
      .then(() => {
        console.log('内部第一个then')
        return Promise.resolve()
      })
      .then(() => {
        console.log('内部第二个then')
      })
  })
  .then(() => {
    console.log('外部第二个then')
  })
  .then(() => {
    console.log('外部第三个then')
  })
  .then(() => {
    console.log('外部第四个then')
  })

console.log('外部结束')
```

我们知道`then`方法里面的回调函数会被放入微任务队列，当主线任务执行完毕后下一次事件轮询才会执行，大多数人在没有真正从源码层面理解的情况下，对`then`的认知就止步于此了，于是上面这道题目马上就会掉坑。

这道题目的正确答案（以chrome浏览器运行环境为准）

```text
外部promise
外部结束
外部第一个then
内部promise
内部第一个then
外部第二个then
外部第三个then
外部第四个then
内部第二个then
```

诡异的地方来了，内层的第二个`then`居然是最后才执行的！

我们来分析一下代码的运行过程：

1. 第一个`Promise`内部打印 `外部promise`并直接调用`resolve()`（这里要注意的一点就是`Promise`构造时传入的函数是同步执行的），状态变成fulfilled，外层的第一个`then`被放入微任务队列，外层的其余`then`都要等待第一个`then`完成后才会被执行，所以不用考虑。然后主任务直接执行到剩下的最后一行，打印`外部结束`。

   - 此时微任务队列`[外部第一个then]`

2. 由于主任务没有代码要执行了，微任务队列的第一个任务进栈处理，外部第一个`then`执行，内部又声明了一个`Promise`，并执行里面的函数，打印`内部promise`，然后这个`Promise`也直接被resolve了，内部的第一个`then`进入微任务队列，内部第二个`then`要等待内部第一个`then`执行结束，暂时不会进入微任务队列。

   此时外部第一个`then`的同步代码部分就执行结束了，由于它没有定义返回值，相当于就是返回了一个`undefined`，这个值就会被作为这个`Promise`的返回值被resolve（这个部分在下面的源码部分我们详细来看），于是外部第二个`then`也被放入微任务队列。

   - 此时微任务队列`[内部第一个then，外部第二个then]`

3. 主任务清空，开始执行微任务队列，先取出内部第一个`then`处理，它的回调函数直接返回了一个   `Promise.resolve()`，也就是一个处于fulfilled状态的`Promise`。接下来就是真正考察功力的时候了，我们需要思考的问题是`Promise`在resolve一个普通值和一个`Promise`的时候，有什么差别呢？这里面的差别非常大，如果resolve的是普通值，直接就会注册后面的`then`方法，也就是把`then`当中的回调函数放入微任务队列。而当resolve的对象也是一个`Promise`的时候，并不会直接返回这个`Promise`，而是会返回一个`NewPromiseResolveThenableJob`，这是ECMA-262标准里规定的，这个Job内部会去执行要返回`Promise`的`then`方法，同时这个Job本身会被放入到微任务队列里。

   这里的细节一下子多了很多，我们分别列举出来一个个看：

   1. 如果`Promise1`要返回一个`Promise2`（也可以理解成`Promise2`会resolve`Promise1`，即把`Promise1`的状态变成fulfilled），会生成一个`NewPromiseResolveThenableJob`放入微任务队列。
   2. 这个Job里会调用`Promise2.then(resolve, reject)`，要注意这里传入的`resolve`和`reject`是`Promise1`自己的！它们处理的是`Promise1`，也就是说`Promise2.then`里面的执行结果会决定`Promise1`的状态。
   3. 等到这个Job被执行了，`Promise2.then`的回调被放入微任务。
   4. 等到`Promise2.then`的回调被执行了，`Promise1`终于状态改变，它自己的`then`才能开始被处理。
   5. 从1-4，一共经过了3次事件轮询！所以当`then`返回一个`Promise`的时候，后面的链式`then`并不会马上被放入微任务，第一次放入的是`NewPromiseResolveThenableJob`，第二次放入的是`Promise2.then`，第三次才是链式的`then`。

   回到3我们刚刚讨论到的地方：

   - 回到此时微任务队列`[外部第二个then，NewPromiseResolveThenableJob]`

4. 执行外部第二个`then`，打印`外部第二个then`。这个`then`没有返回值也没有异步代码，所以直接返回`undefined`，和2里面我们讨论过的一样，外部第三个`then`顺利被放入微任务队列。

   - `[NewPromiseResolveThenableJob，外部第三个then]`

5. 执行`NewPromiseResolveThenableJob`，把`Promise.resolve().then`放入微任务队列。

   - 此时微任务队列`[外部第三个then，Promise.resolve().then]`

6. 执行外部第三个`then`，打印`外部第三个then`，把外部第四个`then`放入微任务队列。

   - 此时微任务队列`[Promise.resolve().then，外部第四个then]`

7. 执行`Promise.resolve().then`，没有任何返回值（`undefined`），但是把内部第一个`then`返回的`Promise`状态变为fulfilled，相当于内部第一个`then`最后返回了`undefined`，这时把内部第二个`then`放入微任务队列。

   - 此时微任务队列`[外部第四个then，内部第二then]`

8. 后面就很好理解了，依次执行剩下的两个`then`，分别打印`外部第四个then`和`内部第二个then`。

## 4. Promise源码理解

我们来看下模拟部分Promise源码的简单实现

```js
then (successCallback, failCallback) {
  // 判断传入的回调是不是函数
  successCallback =
    typeof successCallback === 'function' ? successCallback : (value) => value
  failCallback =
    typeof failCallback === 'function' ? failCallback : (reason) => reason

  let promiseReturned = new MyPromise((resolve, reject) => {
    // 判断状态，如果已经变化了就立即调用回调函数
    if (this.status === FULFILLED) {
      setTimeout(() => {
        try {
          let x = successCallback(this.value)
          // 检查x是否是也是一个MyPromise对象
          resolvePromise(promiseReturned, x, resolve, reject)
        } catch (error) {
          reject(error)
        }
      }, 0)
    } else if (this.status === REJECTED) {
      setTimeout(() => {
        try {
          let x = failCallback(this.reason)
          // 检查x是否是也是一个MyPromise对象
          resolvePromise(promiseReturned, x, resolve, reject)
        } catch (error) {
          reject(error)
        }
      }, 0)
    } else {
      // 等待，状态还没有变化
      // 将成功和失败的回调函数储存起来
      this.successCallback.push(() => {
        setTimeout(() => {
          try {
            let x = successCallback(this.value)
            // 检查x是否是也是一个MyPromise对象
            resolvePromise(promiseReturned, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      })
      this.failCallback.push(() => {
        setTimeout(() => {
          try {
            let x = failCallback(this.reason)
            // 检查x是否是也是一个MyPromise对象
            resolvePromise(promiseReturned, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      })
    }
  })
  return promiseReturned
}
```

从上面的代码中可以明显的看到几个特点：

一是果然所有的`then`都会在内部声明并返回一个`Promise`： `promiseReturned`

二是`then`是如何把回调函数放入异步队列的（代码里使用了`setTimeout`来模拟），包括如果当前`Promise`还处于pending状态时，会把回调函数包裹在一个异步任务里再放入数组中保存。

三是回调函数的返回值就是`Promise`要resolve的值，也就是`x`，但是具体怎么resolve或者reject还要经过一个函数`resolvePromise`。

首先我们先看看`resolve`函数是怎么工作的（要注意这里的`resolve`方法不是`Promise.resolve`！后者是一个独立的静态方法，这里没有讨论）：

```js
resolve = (value) => {
  if (this.status != PENDING) return
  this.status = FULFILLED
  this.value = value
  // 判断成功回调是否存在，如果存在就调用
  // this.successCallback && this.successCallback(this.value)
  // 现在回调函数是一个数组
  while (this.successCallback.length) {
    let callback = this.successCallback.shift()
    callback()
  }
}
```

当调用`resolve`的时候，不仅会改变`Promise`的状态，同时也会把数组中保存的回调任务按顺序全部执行了，`reject`函数的原理完全一样。

最后我们再来看看`resolvePromise`和`resolve`有什么不同：

```js
function resolvePromise (promiseReturned, x, resolve, reject) {
  //! 不允许返回自身 会造成无限then嵌套
  if (promiseReturned === x) {
    reject(new TypeError('Chaining cycle detected for promise'))
  }
  if (x instanceof MyPromise) {
    // 是一个 MyPromise对象
    // 这里模拟了ECMA标准里的NewPromiseResolveThenableJob(promiseToResolve, thenable, then)
    setTimeout(() => {
      x.then(resolve, reject)
    }, 0)
  } else {
    // 普通值
    resolve(x)
  }
}
```

可以看到，`resolvePromise`相当于包裹了`resolve`，只不过处理了返回值的各种情况，而且要注意其参数设计别有用意：

- `promiseReturned`： 当前`then`要返回的`Promise`（还记得吗，所有的`then`默认都返回一个`Promise`）
- `x`：当前`then`内部回调函数的返回值，用于resolve当前这个`promiseReturned`
- `resolve`：`resolvePromise`自己的resolve方法
- `reject`：`resolvePromise`自己的reject方法

当返回值是一个普通值的时候，直接调用`resolve`；而如果也是一个`Promise`，就会出现我们之前分析的那种情况，构造一个新的异步任务，在这个里任务里面去调用x的`then`方法，在这个`then`方法里再去调用传入的`resolve`和`reject`，也就是说`x`会决定`promiseReturned`的状态，但不是这一时刻同步决定的，而是下一次异步轮询。

最后还有一点是，由于`resolve`和`reject`函数被这么传来传去，所以为了保持它们内部的`this`指向，必须要通过箭头函数的方法来定义，否则还得各种使用闭包去记录原本的`this`。

## 5. 总结

最后我们再来稍微改造一下2里面的例子，把fn3改为：

```js
async function fn3() {
  console.log('fn3 start')
  return Promise.resolve()
}
```

现在运行的结果是什么样的呢？我们现在已经知道了`Promise`内部如果resolve另一个`Promise`要花费三个微任务才能真正完成，所以最后的打印结果：

```text
fn1 start
fn2 start
fn3 start
finally
Promise
fn2 end
fn1 end
```

外部的Promise打印会最先进入异步队列，fn2和fn1变得更晚了。

经此一役，相信你对`Promise`的理解已经远超凡人，不会再有什么题目能轻易难道你了！
