---
title: ES2015
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-06-30 23:12:13
tags:
  - javascript
categories:
  - 前端学习笔记
index_img: /img/ES6.png
---

# ES2015

## 1. let和const的块作用域

let和const的块级作用域是JS很大的一个进步，尤其是let在循环中的使用很有意思

```js
for (let i = 0; i < 3; i++) {
    for (let i = 0; i < 3; i++) {
      console.log(i)
    }
}
```

像上面那样在循坏的内外层使用同名变量i但却不会相互覆盖

同时我们也可以这样

```js
let elements = [{}, {}, {}]
for (let i=0;i<elements.length;i++) {
  elements[i].onclick = function() {
    console.log(i)
  }
}
elements[0].onclick() // 0
```

这里通过闭包每个i都被单独地记住了

另外一个值得注意的地方是

```js
for (let i = 0; i < 3; i++) {
  let i = 'foo'
  console.log(i)
} // 输出三次foo
```

这里循环逻辑内部的i也不受影响，我们可以这样来理解，上面的代码其实等同于：

```js
let i = 0

if (i < 0) {
  let i = 'foo' // 块作用域内部的i的独立的
  console.log(i)
}

i++
// 以上重复三次
```

最后let和const都不会自动提升，必须声明后才能使用，否则抛出RefError

但要特别注意的是：**所谓的不会提升(hoist)指的是初始化不会提升，而不是声明不会提升**

let和const的声明依然会被提升，这是Javascript的底层机制决定的，Javascript引擎会在进入每个作用域时寻找该作用域内部的所有变量声明并创建它们

唯一的区别就是，var还会执行初始化（undefined），而let和const不会，这就是为何在赋值前使用let和const会抛出异常的真正原因

[参考资料: let存在变量提升吗？](https://www.jianshu.com/p/0f49c88cf169)

## 2. Proxy

Proxy专门用于对对象进行代理操作，是比Object.defineProperty更灵活便捷的方法。Proxy内置了13个代理方法handler，又称为捕捉器，比如最常用的set和get

```js
const person = {
    name: 'Tom',
    age: 20
}

const personProxy = new Proxy(person, {
    get(target, property) {
        return property in target ? target[property] : 'default'
    },
    set(target, property, value) {
      // 捕捉set进来的属性和值，可以进行任意额外的操作
      if (property === 'age') {
          if(!Number.isInteger(value)) {
              throw new TypeError(`${value} is not an integer`)
          }
      }
      target[property] = value
      return true // set方法需要返回一个bool值表示是否成功
    }
})
```

Proxy比Object.defineProperty更为强大，事实上Proxy本身就自带了一个defineProperty的捕捉器

更重要的，Proxy可以方便地劫持数组操作，这也是Vue3.0使用Proxy代替了defineProperty的原因之一

```js
const list = []

const listProxy = new Proxy(list, {
  set(target, property, value) {
    console.log('set', property, value)
    target[property] = value
    return true
  }
})

listProxy.push(100)
// set 0 100
// set length 1
```

第一行的0表示数组下标0， 第二行的length表示对数组长度的操作。可见Proxy自己内部知道数组是如何被操作的，不再需要我们进行干涉，这比defineProperty方便了很多

Proxy对于对象的劫持是非侵入性的，可以任意代理一个已经被定义好的对象和其中的变量，而defineProperty则是一开始就需要声明好

## 3. Reflect

Reflect本身提供了一整套对象操作的拦截函数，可以说和Proxy有点相辅相成的感觉(特别是两者的自带方法都是13个)

```js
const person = {
    name: 'Tom',
    age: 20
}

const personProxy = new Proxy(person, {
    get(target, property) {
      console.log(target, property)
      return Reflect.get(target, property)
    }
}) 
```

Reflect可以统一对象的操作方式。传统的JS里针对对象的操作种类繁多，而且语法差异很大，非常不规范，如下例

```js
const person = {
    name: 'Tom',
    age: 20
}

console.log('name' in person) // 判断是否存在属性
console.log(delete person['age']) // 删除特定的属性
console.log(Object.keys(person)) // 获取所有属性名

// 使用Reflect统一操作
console.log(Reflect.has(person, 'name'))
console.log(Reflect.deleteProperty(person, 'age'))
console.log(Reflect.ownKeys(person))
```

## 4. Set

Set类似Python等语言中的集合，是数学概念集合的实现，最有用的地方就是去重

```js
const s = new Set()
s.add(1).add(2).add(2).add(3)
console.log(s) // Set {1, 2, 3}

s.forEach(v => console.log(v))

// 注意Set是一个对象，并没有索引，所以不能使用for in
for (let v of s) {
  console.log(s)
}

console.log(s.size) // 获取大小
console.log(s.has(100)) // 是否具有某个元素
console.log(s.delete(3)) // 成功删除返回true，否则false
console.log(s.clear()) // 清空集合

const arr = [1, 1, 1, 2, 2, 2]
const result = [ ... new Set(arr)]
console.log(result)
```

## 5. Map

即便有计算属性这个对象字面量的增强版，可以给对象添加动态的属性名，比如

```js
const obj = {
  [Math.ramdom()] : 10,
  [1 + 1] : 20
}
```

这种机制仍然有一个很大的限制，就是无法使用对象作为变量名

```js
const obj = {
  [{a: 2}] : 10,
  [{b: 3}]: 30
}

console.log(obj) // { '[object Object]': 30 }
console.log(obj[{a: 2}]) // 30 错误值
```

可以看到两个属性名其实都被toString之后再当做属性名，导致彼此覆盖掉了，因为任何对象toString = `[object Object]`。也就是说，传统的JS里对象的属性名只能为字符串。

Map就可以解决这个问题

```js
const m = new Map()
const tom = {a: 10}

m.set(tom, 10)
console.log(m) // Map { { a: 10 } => 10 }
console.log(m.get(tom)) // 10 正确值

// 和Set一样具有常用的基本操作
// m.has()
// m.delete()
// m.clear()
```

## 6. Symbol

Symbol是很有意思的一个数据类型，类似传统静态语言中常用的UID，专门用于生成一个不会重复的变量

```js
let x = Symbol.for('Hi') // 使用for来注册一个全局Symbol
console.log(Symbol.keyFor(x)) // Hi
```

如果不使用Symbol.for来注册，每次Symbol都会生成一个不同的新值

```js
let x = Symbol.for('Hi')
let y = Symbol.for('Hi')

console.log(Symbol('x') === Symbol('x')) // false
console.log(x === y) // true
```

Symbol的特性可以用来创造对象内部的私有成员

```js
const name = Symbol()
const person = {
  [name]: 'Yikhan',
  say() {
    console.log(this[name])
  }
}

person.say() // Yikhan
```

进一步来说，因为这个功能Symbol也成为了JS的一些内置接口名称，比如`[Symbol.toStringTag]`

```js
const obj = {
  [Symbol.toStringTag]: 'Awesome Name',
  foo: 10
}

console.log(obj.toString()) // [object Awesome Name]
```

通过Symbol定义的属性，无论是使用for in还是`Object.keys()`，亦或是`JSON.stringify()`都是获取不到的，会直接被这些方法忽略掉

## 7. Iterator

迭代器接口直接和for of操作符相关，只有实现了这个接口才能调用for of来遍历元素

要注意的是for of不能直接遍历普通的Object，只能遍历`Object.values(obj)`

迭代器接口的关键字就是在对象中实现`[Symbol.iterator]`

```js
const obj = {
  store: [ 'foo', 'bar', 'baz' ],
  [Symbol.iterator]: function() {
    let index = 0

    return {
      next: () => {
        const result = {
          value: this.store[index],
          done: index >= this.store.length
        }
        index++
        return result
      }
    }
  }
}

for (const item of obj) {
  console.log(item) // foo bar baz
}
```

如果结合生成器generator，还可以进一步简化这个方法

```js
const obj = {
  store: [ 'foo', 'bar', 'baz' ],
  [Symbol.iterator]: function*() {
    for (const item of this.store) {
      yield item
    }
  }
}
```
