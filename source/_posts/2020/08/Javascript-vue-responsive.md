---
title: Vue核心原理 - 自己实现一个Vue
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-08-28 22:08:38
tags:
  - javascript
  - Vue
categories:
  - 前端学习笔记
index_img: /img/vue.jpg
---

# Vue核心实现原理

## 1. 观察者模式和发布订阅模式

<center>
<img src="https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200722225714226.png" width="50%" height="50%"  />
</center>

观察者模式和发布订阅模式是最为常用的事件响应设计模式。

与观察者模式相比，发布订阅模式多了一层事件中心，隔离了发布者和订阅者，使其不需要相互依赖。

既然我们希望当数据改变的时候视图也自动更新，那么就需要一个发布者来跟踪数据，当数据改变的时候这个发布者能够通知所有订阅自己的观察者，观察者再负责改变视图。

所以简单来说：

- 发布者 - 跟踪数据变化，有变化发生时通知观察者
- 观察者 - 当被通知的时候更新视图

这个原理其实非常简单，剩下的就是如何实现里面的细节了，比如：

- 发布者
  - 如何跟踪数据？- 给数据设置setter/getter函数
  - 如何通知观察者？- 创建一个数组存放所有的观察者，遍历并调用每个观察者的`update`方法
- 观察者
  - 如何订阅发布者？- 被放到订阅者的观察者数组中就可以了
  - 如何更新视图？- 实现`update`方法：获取页面DOM元素并修改

Vue的响应式机制就使用了观察者模式，从源代码中的相应类的命名也能看出来（`Observer`，`Dependant(依赖管理者，就是发布者)`，`Watcher(观察者)`）

## 2. Vue原理

一个基本的响应式Vue由5个组件构成，每个组件都是一个class

<center>
<img src="https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/VueStructure.jpg" alt="VueStructure" style="zoom:50%;" />
</center>

### Vue

需要实现的功能

- 接收初始化的参数（选项）
- 把data中的属性注入到Vue实例，转换成getter/setter
- 调用observer监听data所有属性的变化
- 调用compiler解析指令和插值表达式

### Observer

把data中的属性变为响应式

### Compiler

需要实现的功能

- 编译模板，解析指令/插值表达式
- 页面的首次渲染
- 当数据变化后重新渲染视图

### Dep

通过Data的getter收集依赖，然后通过setter触发依赖，使用`notify`方法通知所有依赖自己的`Watcher`

### Watcher

需要实现的功能

- 当数据变化时触发依赖，Dep通知所有的Watcher实例更新视图
- 自身实例化的时候往Dep对象中添加自己

## 3. 自己实现一个Vue

> Talking is cheap, show me the code

下面我们就来自己实现一个简化版本的Vue，充分理解上面5个类是如何相互协同工作的

5个class的各自属性和方法成员如下所示：

![Vue](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/Vue.png)

### 3.1 `Vue`

首先我们都知道Vue的使用是以`new Vue()`的调用来开始的，所以`Vue`这个同名class当然是第一个要实现的：

```js
class Vue {
  constructor (options) {
    //* 1. 通过属性保存选项的数据
    this.$options = options || {}
    this.$data = options.data || {}
    this.$methods = options.methods || {}
    // 如果$el是字符串，就使用选择器查找这个元素，反之则认为传入的就是一个dom元素
    this.$el =
      typeof options.el === 'string'
        ? document.querySelector(options.el)
        : options.el

    //* 2.1 把data中的成员转换为getter/setter，注入到Vue实例中
    this._proxyData(this.$data)
    //* 2.2 把method中的函数成员注入到Vue实例中
    this._proxyMethod(this.$methods)

    //* 3. 调用observer对象，监听对象变化
    new Observer(this.$data)
  
    //* 4. 调用compiler对象，解析指令和插值表达式
    new Compiler(this)
  }

  _proxyData (data) {
    // 遍历data中的所有属性
    Object.keys(data).forEach((key) => {
      // 注入到Vue实例
      Object.defineProperty(this, key, {
        enumerable: true,
        configurable: true,
        get () {
          return data[key]
        },
        set (newValue) {
          if (newValue === data[key]) {
            return
          }
          data[key] = newValue
        }
      })
    })
  }

  _proxyMethod(methods) {
    Object.keys(methods).forEach((name) => {
      Object.defineProperty(this, name, {
        enumerable: true,
        configurable: true,
        get() {
          return methods[name]
        }
      })
    })
  }
}
```

首先当然是接受传入的参数进行初始化，然后就是第一个使用了`Object.defineProperty`的地方：`_proxyData`，它把`data`里的所有成员都挂载到了`this`，也就是`Vue`实例上，这也是为什么我们能够在模板里直接访问`data`里的属性的原因（比如`{{ user }}`），下面的`_proxyMethod`原理一样。

### 3.2 `Observer`

然后到了`new Observer(this.$data)`这一行，我们就要实现`Observer`这个类了：

```js
class Observer {
  constructor(data) {
    this.walk(data)
  }

  walk(data) {
    // 判断data是否为对象
    if (!data || typeof data !== 'object') {
      return
    }

    // 遍历data所有属性
    //! 注意compiler内部没有实现针对新对象内部的属性创建watcher
    Object.keys(data).forEach(key => {
      this.defineReactive(data, key, data[key])
    })
  }

  defineReactive(obj, key, val) {
    // 把当前的this记录下来
    let self = this

    // dep负责收集依赖，发送通知
    let dep = new Dep()

    // 如果val是对象，把val内部的属性也变成响应式
    this.walk(val)

    // defineProperty内部的this发生了变化，不再指向Observer，所以不能直接使用this
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get() {
        //* 收集依赖，watcher初始化的时候会绑定到Dep.target这个静态属性上
        Dep.target && dep.addSub(Dep.target)

        //! 这里不能使用obj[key]，否则会死循环，因为obj[key]又会触发get
        //! 所以val要作为参数传入，会通过闭包被缓存下来
        return val
      },
      set (newValue) {
        if (newValue === val) {
          return
        }
        val = newValue
        // 如果新的值是对象，把其内部的属性也变成响应式
        self.walk(newValue)
        // 发送通知
        dep.notfiy()
      }
    })
  }
}
```

`Vue`里面的`_proxyData`只是把`data`里面的属性挂载到了实例上，还没有实现响应式，而这里的`defineReactive`才是真正的重头戏，也就是Vue的响应式核心所在。

1. 首先我们调用了`walk`方法来遍历`data`，而且注意`defineReactive`和`walk`相互调用形成递归，只要一个属性是对象就会一直深入下去把里面对象内部的属性也变成响应式，知道遇见非对象的属性为止。
2. `defineReactive`先声明了一个`Dep`，也就是发布者，用于收集依赖。
3. 再次使用了`Object.defineProperty`方法来对属性设置`setter`和`getter`，于是属性值的改变和读取都被劫持了。
4. 在`getter`里面先判断当前`Dep`上有没有需要被收集（想订阅发布者）的`Watcher`，这个判断是通过`Dep.target`这个静态属性来实现的，非常巧妙，我们后面在`Watcher`内部可以看到为什么要这么做。如果有，就把它放入订阅者（观察者）数组，这样“订阅”这个目的就实现了。
5. 当`setter`被触发时，通知所有订阅了`Dep`的`Watcher`，于是视图被更新了。

这里要注意的一点是，`defineReactive(obj, key, val)`接受三个参数，第三个`val`就是被劫持属性本身的值，在`getter`里面直接返回这个值给外部的访问者，而不是像`Vue`里面一样直接返回`obj[key]`，这是为了避免`obj[key]`再次触发`getter`导致无限自我递归。

- 那为什么`Vue`里面的`_proxyData`使用`obj[key]`不会无限递归？

因为`_proxyData`是在`this`，也就是Vue实例上新增了属性，这个属性被读取时返回`data[key]`。而`defineReactive`是直接在`data`上再次定义了同样的`key`，所以`data[key]`会触发`getter`，然后`getter`内部如果又是`return data[key]`就又会触发`getter`了。

### 3.3 `Dep`

接着我们来实现`Dep`这个class

```js
class Dep {
  constructor() {
    this.subs = []
  }

  //* 添加观察者
  addSub(sub) {
    // 所有的观察者都必须有一个update方法
    if (sub && sub.update) {
      this.subs.push(sub)
    }
  }

  //* 发送通知
  notfiy() {
    this.subs.forEach(sub => {
      sub.update()
    })
  }
}
```

这个类很简洁，`addSub`方法用于添加订阅自己的`Watcher`，`nofity`方法用于通知所有订阅自己的`Watcher`

### 3.4 `Watcher`

```js
class Watcher {
  constructor(vm, key, cb) {
    this.vm = vm
    // data中的属性名
    this.key = key
    // 回调函数负责更新视图
    this.cb = cb
    //* 把watcher注册到Dep的静态属性target
    Dep.target = this
    //* 触发get方法，在get方法中调用addSub
    this.oldValue = vm[key]
    //* 注册成功之后把target重置为null，避免重复注册
    Dep.target = null
  }

  //* 数据变化时使用cb更新视图
  update() {
    let newValue = this.vm[this.key]
    if (this.oldValue === newValue) {
      return
    }

    this.cb(newValue)
  }
}
```

`Watcher`同样不复杂，里面最关键的地方有两个

第一个就是如何通过订阅`Dep`：

1. 给`Dep`这个类上增加一个静态属性`target`，这个`target`就指向自己
2. 然后调用`vm[key]`，触发`getter`
3. 回到`Observer`里面，精髓的地方就是这一句`Dep.target && dep.addSub(Dep.target)`，也就是如果`Dep`上有`target`（也就是新的`Watcher`自己），那么就注册这个`target`成为订阅者。于是最关键的订阅功能就完成了！
4. 在成功注册自己之后，把`target`的值重设为`null`

通过`Dep.target`这个静态属性，我们可以在不进行任何多余操作的情况下成功把`Watcher`注册到了`Dep`上面。

第二个关键地方就是`update`函数：

如果新旧值不同，就要调用`Watcher`初始化时传入的`callback`函数，这个函数应该实现真正更新视图的逻辑，比如访问DOM。

### 3.5 `Compiler`

最后尚缺两块拼图，一个是上面`Observer` - `Dep` - `Watcher`这个发布订阅模式的最后一环，就是在哪里初始化`Watcher`，第二个是模板指令的解析，比如`v-model`，`v-on`，`v-html`和插值表达式等。

这两块其实是互相关联的，其实就是在解析html模板的时候知道哪些变量需要监视，然后生成对应的`Watcher`。

`Compiler`涉及到DOM节点的创建，代码相对较多，只取最关键的地方。

当解析到特定的模板指令时，我们就调用相应的`updater`函数去创建`Watcher`：

```js
update (node, key, attrName, event) {
  // 通过拼接函数名找到处理函数，避免if语句
  let updateFn = this[attrName + 'Updater']
  // 注意调用时this的指向
  updateFn && updateFn.call(this, node, key, this.vm[key], event)
}
```

先声明好一系列的模板`updater`，然后通过函数名拼接的方式`this[attrName + 'Updater']`来调用，这样可以避免用一大堆的`if`语句。

比如`v-model`就对应一个`modelUpdater`，同理还有`v-html`对应的`htmlUpdater`，`v-on`对应的`onUpdater`等等。

```js
modelUpdater (node, key, value) {
  node.value = value
  new Watcher(this.vm, key, (newValue) => { // 传入改变视图的callback函数
    node.value = newValue
  })
  // 双向绑定
  node.addEventListener('input', () => {
    this.vm[key] = node.value
  })
}
```

`updater`里面会声明一个`Watcher`，这样发布订阅模式就完整实现了。
