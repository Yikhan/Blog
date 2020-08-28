---
title: Javascript性能优化
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-06-30 23:18:03
tags:
  - javascript
categories:
  - 编程技巧
  - 前端学习笔记
index_img: /img/performance.png
---

# Javascript性能优化

## GC常见算法

常用的内存回收(Garbage Collection)算法整理：

![GC回收算法](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/GC回收算法.png)

## 代码优化

可以使用Jsperf来进行Javascript代码的性能测试和对比

<https://jsperf.com/yif-global-variable>

### 1. 慎用全局变量

- 全局变量定义在全局执行上下文，是所有作用域链的顶端（根作用域）
- 由于一直存在于根作用域，难以被GC（内存回收）清理，会一直存在知道程序退出
- 容易被局部作用域的同名变量遮蔽或者污染

### 2. 将使用中无法避免的全局变量放入缓存

例如为了获取获取dom元素节点，在需要大量获取document节点的函数中缓存全局变量document可以增加性能

```js
// 不使用对象缓存
function getBtnWithoutCache() {
    let btn1 = document.getElementById('btn1')
    let btn2 = document.getElementById('btn2')
    let btn3 = document.getElementById('btn3')
}

// 使用对象缓存
function getBtnWitCache() {
    let obj = document
    let btn1 = obj.getElementById('btn1')
    let btn2 = obj.getElementById('btn2')
    let btn3 = obj.getElementById('btn3')
}
```

其原理是通过缓存加快对document全局变量的访问速度，不需要每次都从根作用域重新查找

### 3. 通过原型链添加方法

原型链是所有对象共享的，比起在每个对象的this上增加方法，使用原型链的性能会更好

![image-20200630155359315](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200630155359315.png)

### 4. 避免闭包陷阱

闭包会产生额外的对象引用，会使得本身应该被释放回收的对象由于闭包内的引用而无法被回收

```js
function foo() {
  let el = document.getElementById('btn')
  el.onclick = function() {
    console.log(this.a)
  }
}

foo()
```

上面的el由于onclick函数的闭包会一直存在，使得即便btn节点已经从dom中被移除了也不会被回收，因为el一直对其进行引用

### 5. 避免属性访问方法

Javascript中没有传统面向对象语言的属性访问限制，一个对象里所有属性都是对外暴露的（这里没有讨论ES2015之后的class），在这种情况下，使用对象访问方法反而会降低性能

```js
function Person() {
    this.name = 'Me'
    this.age = 18
    this.getAge = function() {
        return this.age
    }
}
```

上面的`getAge`就是一种属性访问方法，我们可以和直接的属性访问对比：

![image-20200630162233394](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200630162233394.png)

可以发现直接访问性能明显更好，这也是符合直觉的

### 6. For循环优化

如果for循环的终止条件是一个定值，最好直接把它取出来缓存成一个变量

```js
// 使用len变量来直接获取数组长度，避免重复访问
for (let i = 0, len = arr.length; i < len; i++) {
    console.log(i)
}
```

![image-20200630164130926](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200630164130926.png)

再对比一下常用的四种for方法

<https://jsperf.com/yif-for-loop-performance>

- forEach (ES5引入)
- for
- for in
- for of (ES2015引入，只能遍历可迭代对象，如数组，不能直接遍历对象，可以遍历`Object.values()`或者`Object.keys()`)

测试的数组

```js
const arrList = []
for (let i = 0; i < 10000; i++) {
    arrList.push(i)
}
```

![image-20200630180407937](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200630180407937.png)

可以发现forEach的速度是最快的（但这个结果或许并不准确，关于forEach和for的对比争论在stackoverflow上很多，两者其实在不同场景各有胜负，不过从语义角度而言，forEach作为比较新的语法表达更简洁易读，但要注意的是forEach不能使用break/continue来中断，而ES2015/ES6新加入的for of则可以）

### 7. 使用文档碎片优化节点添加

```js
// 直接往body上添加节点元素
for (let i = 0; i < 10; i++) {
    let oP = document.createElement('p')
    oP.innerHTML = i
    document.body.appendChild(oP)
}

// 使用documentFragment
const fragEle = document.createDocumentFragment()
for (let i = 0; i< 10; i++) {
    let oP = document.createElement('p')
    oP.innerHTML = i
    fragEle.appendChild(oP)
}
document.body.appendChild(fragEle)
```

早期批量操作dom的开销比较大，使用`Fragment`可以很好的降低开销，因为是先把文档操作放到虚拟的`Fragment`上再一次性插入到document，`Fragment`就相当于占位符，在插入之后本身就会销毁

现代浏览器已经对这类批量文档操作做了优化，实际的运行速度上使用文档碎片并不会有很大的优势了，但是从语义上讲文档碎片依然是很好的选择，因为它表明了不需要马上对页面dom进行更新操作

### 7. 使用克隆节点代替创造节点

当需要插入新节点时，另一个优化的方法是使用`cloneNode`函数，复制一个已有的同类节点再进行更改

```js
for (let i = 0; i < 3; i++) {
    let newP = oldP.cloneNode(false) // 参数表示是否进行深拷贝
    newP.innerHTML = i
    document.body.appendChild(newP)
}
```

`cloneNode`比使用`createElement`性能更好，尤其批量创造同类节点

### 8. 直接量替换new Object

```js
// 使用new
let a1 = new Array(3)
a1[0] = 1
a1[1] = 2
a1[2] = 3

// 直接量
let a2 = [1, 2, 3]
```

这个非常好理解，使用直接量的性能更好
