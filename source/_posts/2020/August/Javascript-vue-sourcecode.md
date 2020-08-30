---
title: Vue源码理解心得
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-08-13 12:06:05
tags:
  - javascript
  - Vue
categories:
  - 前端学习笔记
index_img: /img/vue.jpg
---

# Vue源码理解心得

## 1. 响应式原理

先上一张图作为总纲心法：

![Vue响应式处理过程](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/Vue响应式处理过程.png)

里面有很多实现细节，但大概的原理都和[自己实现的简化版的Vue](https://yikhan.github.io/2020/August/Javascript-vue-responsive/)类似。

其中比较有意思的几个地方，也是面试里面常常考察的点，通过理解源码之后就再无任何疑问了，一通百通。

### 1.1 数组响应式劫持

众所周知，Vue对于数组的响应式处理是一个被经常讨论的话题，比较关键的源码如下：

>源码位置：vue\src\core\observer\index.js 是响应式处理的核心

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 判断 value 是否是对象
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 如果 value 有 __ob__(observer对象) 属性 结束
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建一个 Observer 对象
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

上面代码一开始的判断非常重要，如果传入的`value`不是对象或者VNode，**就不会进行响应式处理了**！

从一开始的总纲可以看到，`observe`方法是开始响应式处理的关键入口，其中会创建`Observer`这个响应式类，然后我们再到`Observer`的源码中看看：

```js
constructor (value: any) {
  this.value = value
  this.dep = new Dep()
  // 初始化实例的 vmCount 为0
  this.vmCount = 0
  // 将实例挂载到观察对象的 __ob__ 属性
  def(value, '__ob__', this)
  // 数组的响应式处理
  if (Array.isArray(value)) {
    if (hasProto) {
      protoAugment(value, arrayMethods)
    } else {
      copyAugment(value, arrayMethods, arrayKeys)
    }
    // 为数组中的每一个对象创建一个 observer 实例
    this.observeArray(value)
  } else {
    // 遍历对象中的每一个属性，转换成 setter/getter
    this.walk(value)
  }
}
```

这里的构造函数比较直白，马上可以看到对于数组是有额外的判断处理的，需要调用`observeArray`这个方法。

```js
observeArray (items: Array<any>) {
  for (let i = 0, l = items.length; i < l; i++) {
    observe(items[i])
  }
}
```

这个方法非常简单，就是遍历数组的每个元素，然后再次递归调用一开始的`observe`方法。

然后再回到`Observer`中，在10行的位置判断了当前环境是否有prototype可以用，然后分别调用`protoAugment`或者`copyAugment`两个方法，其目的都是劫持数组自带的方法，关键的源码如下：

> 源码位置：vue\src\core\observer\array.js

```js
const arrayProto = Array.prototype
// 使用数组的原型创建一个新的对象
export const arrayMethods = Object.create(arrayProto)
// 修改数组元素的方法
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
```

具体的劫持方法：

```js
/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  // 保存数组原方法
  const original = arrayProto[method]
  // 调用 Object.defineProperty() 重新定义修改数组的方法
  def(arrayMethods, method, function mutator (...args) {
    // 执行数组的原始方法
    const result = original.apply(this, args)
    // 获取数组对象的 ob 对象
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 对插入的新元素，重新遍历数组元素设置为响应式数据
    if (inserted) ob.observeArray(inserted)
    // notify change
    // 调用了修改数组的方法，调用数组的ob对象发送通知
    ob.dep.notify()
    return result
  })
})
```

这样串联起来思考一下，我们很容易就能通过源代码的逻辑知道：

1. 数组的每个元素都是响应式的吗？

   -- 不是，注意`observe`方法一开始的判断，当`observeArray`遍历数组元素的时候，如果元素不是对象或者VNode，就不会被响应式处理。这也非常合理，因为数组有可能非常大，都转化成响应式毫无意义，性能极低。

2. 什么情况下数组元素会变成响应式？

   -- 当元素本身是对象或者VNode的时候！这时`observe`方法就会尽心尽力地开始遍历元素的成员并转换成响应式了。

这就是为什么直接修改数组成员，比如`arr[0] = 100`这种或者修改数组自带成员`arr.length = 0`都无法触发视图更新，因为本来就不是响应式的。

那如果要响应式地改变数组成员如何处理？

1. 使用`splice`方法，比如`arr.splice(1, 1, 100)`，删除原来索引为1的元素然后替换为新值，`splice`方法因为被劫持了，所以是响应式的。
2. 使用`$set`方法，比如`vm.$set(vm.arr, 1, 100)`，在JS里数组的索引等同于对象的键，所以`$set`方法不但能用于设置对象的响应式成员，也能设置数组元素。有意思的是，如果看`$set`方法的源码，可以发现内部也是通过调用被劫持的`splice`方法来改变数组的。

### 1.2 `set`和`del`方法

> 源码位置：vue\src\core\observer\index.js

上面提到了`$set`方法，其实就是静态方法`Vue.set`的别名，在初始化的时候会被挂载到Vue实例上，然后通过`vm.$set`的方式被使用。

`set`方法内部调用了核心的`defineReactive(obj, key, val)`来实现响应式地新增成员。

`set`和`del`方法代码十分相似，其中有意思的一段源码是（两个函数都有）：

```js
// 如果 target 是 vue 实例或者 $data 直接返回
if (target._isVue || (ob && ob.vmCount)) {
  process.env.NODE_ENV !== 'production' && warn(
    'Avoid adding reactive properties to a Vue instance or its root $data ' +
    'at runtime - declare it upfront in the data option.'
  )
  return val
}
```

上面的代码表明了`set`和`del`都不能直接对Vue实例或者`$data`对象使用，如果要操作`$data`，应该要在声明阶段定义好，不允许在运行时动态增删。

比较有趣的地方在于代码里检查`$data`的方式，因为`observe`方法在初始化传入的`data`并赋值给`$data`时，会新增一个`vmCount`属性到`ob`，并设置为1，其他普通对象则为0，所以`(ob && ob.vmCount)`可以判断传入的要删改对象是否为`$data`，代码简化后类似如下：

```js
ob = value.__ob__ || new Observer(value)
ob = new Observer(value) // vmCount初始化为0
if (asRootData && ob) { // asRootData表示为根数据对象，即$data
  ob.vmCount++
}
```

### 1.3 `nextTick`方法

> 源码位置：vue\src\core\util\next-tick.js

`nextTick`也是常用的一个方法，主要用于在DOM元素更新后立即执行某些操作，因为DOM的更新是异步的，所以同步代码无法获得更新之后的DOM元素内容，所以要使用`nextTick`，这个函数名字也表示了**下一时刻**执行之意。

用法如下：

```js
mounted() {
  this.msg = 'Hello' // 通过响应式数据改变DOM内容
  this.$nextTick(() => {
    console.log(this.$refs.p1.textContent) // 通过$refs获取DOM元素内容
  })
}
```

`nextTick`本身是一个静态方法，和`set`以及`del`函数一样，会在Vue初始化的时候被注入到实例中，然后通过`vm.$nextTick()`的方式来调用，其参数是一个callback函数。

源码实现：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 把 cb 加上异常处理存入 callbacks 数组中
  callbacks.push(() => {
    if (cb) {
      try {
        // 调用 cb()
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    // 调用
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    // 返回 promise 对象
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

了解JS的异步知识之后，很容易想到`nextTick`必然也是借助了`Promise`等异步api来实现。

源码中的细节会更加复杂一些，因为要考虑到环境中不支持`Promise`的情况，也就是19行的那个`timerFunc`

Vue考虑到了各种环境里面的异步api情况，总结一下就是以下面的顺序来决定使用哪个api实现异步延时：

1. `Promise` （微任务）
2. `MutationObserver` （微任务）
3. `setImmediate` （宏任务，主要是IE10以上，还有Node环境，虽然也是宏任务但性能优于`setTimeout`）
4. `setTimeout` （宏任务）

要注意的时，此时DOM并没有真正的更新完毕（稍微想想也能知道DOM更新是浏览器来完成的，速度必然远远滞后于JS，也根本不受JS控制），所以`nextTick`实际上等待的是虚拟DOM的更新而不是真实页面上的DOM！因为虚拟DOM是在JS代码中更新的，所以才能保证我们能在异步任务中获取更新后的虚拟DOM树。

### 1.4 computed和 watcher的区别

computed属性：

- 不支持异步，因为计算属性一般要绑定到模板中
- 会缓存结果以提高性能
- 一定要有返回值

watcher：

- 可以执行异步操作

- 不需要返回值

## 2. Key的作用

几乎所有的文档都会说，在使用`v-for`等指令渲染节点时最好使用key，这和Vue使用的diff算法关系很大。

> 源码位置：vue\src\core\vdom\patch.js

```js
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

Vue的diff算法类似Snabbdom，主要进行同层级的VNode比较，基本上所有文档都会提到diff的5种比较情况：

1. oldStartNode = newStartNode
2. oldStartNode = newEndNode
3. oldEndNode = newStartNode
4. oldEndNode = newEndNode
5. 其他

具体的比较过程不在这里赘述，其篇幅足够单独撰文阐述了，但既然要比较节点，就必然需要一个比较函数，也就是上面的`sameVnode`函数。

可以很清楚地看到，进行节点比较的时候第一个比较对象就是key，如果设置了key，只要两个key不同就能马上判断节点不同，反之如果没有key，第一个判断为true（`undefined === undefined`），所以就要进行后续的比较，这时就有问题了，我们可以举例说明：

> 假设原始DOM元素节点数组为[A, B, C , D]，新的是[A, E, B, C , D]

使用key和不使用key的更新过程：

<img src="https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/key的作用.jpg" alt="key的作用" style="zoom:67%;" />

可以发现，没有key的情况下要更新三次并插入一个新的节点D，而如果有key的话，只需要插入一个新节点E，效率明显高多了。

这里的关键原因就是上面的`sameVnode`函数，如果不看源码的话，大部分人都会想当然地认为判断两个节点是否相同，应该要比较节点里面的内容（`textContent`）才对，然而源码告诉我们，Vue在比较节点的时候并不考虑节点里面的内容，事实上`textContent`的比较和更新是放到最后`patchVnode`方法被调用时，真正改变节点的时候才进行的。

所以当没有key时，`sameVnode`会直接比较两个节点的类型（`tag`），假设它们都是`<li>`元素，就会直接被判定相同，因此老的B节点和新的E节点被认为相同，然后调用`patchVnode`方法更新，后面的C更新成B，D更新成C同理，最后发现新元素还多了一个D，于是再新建一个D。

而如果设置了key，`sameVnode`比较B和E的时候就会返回`false`，知道它们不是同一个节点，然后老的B节点就会继续与后面的节点比较，接着发现B节点可以复用，C，D亦然，最后只用插入一个新的E节点就行了，如此一来效率就高得多了。

## 3. 模板和render函数谁优先

假设有下面的代码，既有`template`又有`render`函数，谁会优先被使用呢？

```js
 const vm = new Vue({
   el: '#app',
   template: '<h1>Hello Template</h1>',
   render(h) {
     return h('h1', 'Hello Render')
   }
 })
```

如果是使用run-time版本的Vue，由于没有模板编译器，`template`自然会被忽略。在完整版的Vue里面，存在如下的判断代码，位于入口文件`entry-runtime-with-compile.js`（也就是带有编译器的版本）

```js
if (!options.render) {
  // 这里的代码是转换template代码为render函数
}
return mount.call(this, el, hydrating)
```

所以`render`函数优先级更高，有`render`函数的时候`template`就不会被编译了。

## 4. el不能是body或者html根标签

在初始化的代码中有如下的判断

```js
if (el === document.body || el === document.documentElement) {
  process.env.NODE_ENV !== 'producation' && warn(
  	`Do not mount Vue to <html> or <body> - mount to normal element instead.`
  )
  return this
}
```

`document.documentElement`就是文档的根元素，一般就是html根标签

## 5. Vue入口文件

Vue的文件打包入口在

> `vue\src\platforms\web\*.js`

在初始化的过程中，有四个模块比较重要

- `src\platforms\web\entry-runtime-with-compiler.js`
  - web平台相关的打包入口
  - 重写了平台相关的`$mount()`方法（增加了编译模板的功能）
  - 注册了`Vue.compile()`方法，传递一个html字符串，返回render函数
- `src\platforms\web\runtime\index.js`
  - web平台相关
  - 注册和平台相关的全局指令：`v-model`，`v-show` -> `Vue.options.directives`
  - 注册和平台相关的全局组件：`v-transition`，`v-transition-group` -> `Vue.options.comoponents`
  - 全局方法：
    - `__patch__`：把虚拟DOM转换为真实的DOM
    - `$mount`：标明渲染到哪里的挂载方法
- `src\core\index.js`
  - 与平台无关
  - 设置了Vue的静态方法，`initGlobalAPI(Vue)`
    - `Vue.set`
    - `Vue.delete`
    - `Vue.nextTick`
    - `Vue.observable` (Vue 2.6新增)
- `src\core\instance\index.js`
  - 与平台无关
  - 定义了构造函数，调用了`this._init(options)`
  - 给Vue中混入了常用的实例成员
