---
title: 前端学习笔记 (chap.2.2) - JS模块化
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-07-17 11:57:38
tags:
  - javascript
categories:
  - 编程技巧
  - 前端学习笔记
index_img: /img/module.png
---

# 模块化开发

## 1. ES Module基本特性

- 自动采用严格模式，忽略`use strict`
- 每个ESM模块都是单独的私有作用域
- ESM是通过CORS去请求外部JS模块的（所以使用链接请求远程JS时要注意对方是否支持）
- ESM的script标签会延迟执行脚本（类似html的defer属性）

如下就是在html里使用ES Module，就是把type定义为module

```html
<script type="module">
	var foo = 100
    console.log(foo)
</script>
```

## 2. ES Module导入导出

基本的export/import用法网上很多，这里仅总结几个容易用错的地方。

### 2.1 export导出的并非对象字面量

最常见的错误就是认为export就是导出一个对象：

```js
var name = 'Yihan'
var age = 10

export {name, age}
```

其实`export {}`只是一种语法，同理`import {}`也是，并非对象字面量，**导入的时候也不是ES6的对象解构**

所以如果试图像下面那么用就会报错：

```js
export { a:11 } // Uncaught SyntaxError: Unexpected token ':'
```

但如果使用export default就可以，这时相当于导出了default这个对象，而default等于后面跟的那个对象：

```js
export default { a:11 } // it works
```

### 2.2 default作为导出名

```js
// 基本的default使用
export default { foo }

// 或者使用as
export {
	foo as default,
    bar as baz
}
```

如果希望在导入时重命名默认导出default，那么就需要在`import {}`括号内使用as

```js
import { default as foo } from './module.js'
```

### 2.3 导出的是变量引用，而且只读

import进来的变量或者方法只是对模块内导出对象的引用，而不是拷贝。所以当模块内发生变化时，这些import进来的变量也会改变，同时要注意它们都是只读的，无法在模块外部被修改。

```js
import { age } from './module.js'

age = 5
console.log(age) // Uncaught TypeError: Assignment to constant variable.
```

### 2.4 export from

这个用法可以直接导出刚导入的变量，一般可以在一个组件或模块的汇总文件里使用。

```js
export { Button } from './button.js'
export { Page } from './page.js'
```

### 2.5 导入

ES模块在导入时必须使用完整的路径和文件名。

```js
import x from './module' // 不行，文件后缀名不能省略
import x from 'module.js' // 不行，相对路径不能省略./ ES会认为你要加载第三方模块

import x from '/package/module.js' // 可以，从项目根目录开始
import x from 'http://localhost:3000/package/module.js' // 可以，允许使用地址
import {} from './module.js' // 可以，不引入只是执行模块
```

import只能出现在最顶层，不允许直接动态导入。

```js
var path = './module.js'
import x from path // 不行
```

想动态加载就必须使用import函数。

```js
import(path).then(function (module) { // ES模块加载是异步的
    console.log(module)
})
```

## 3. 在Node里使用ES模块

### 3.1 Babel

最常用的方式就是通过万能的Babel来转换

Babel通过插件来转换每种ES新特性，所以一般需要安装：

```bash
@babel/node  @babel/core  @babel/preset-env
```

其中preset-env是包含了新特性的一组插件，Babel本身就是通过不同的插件来完成代码转换的。

然后就可以通过`babel-node`命令来运行js文件了。

```bash
yarn babel-node index.js --presets=@babel/preset-env
```

如果不想手动传入presets参数，可以在根目录下创建一个`.babelrc`文件来配置。

```json
{
    "presets": ["@babel/preset-env"]
}
```

注意我们其实也可以自己来安装特定的插件然后使用

比如可以单独安装`@babel/plugin-transform-modules-commonjs`

然后在`.babelrc`里面配置，这时候可以不使用presets，而是plugins直接配置要用的每个插件。

```json
{
    "plugins": ["@babel/plugin-transform-modules-commonjs"]
}
```

### 3.2 Node内置的功能

其实Node较新的版本（高于8.9.0）已经提供了对ES模块的支持，只不过截止目前为止仍然是作为实验特性。

```bash
node --experimental-modules index.mjs
```

要注意的是需要把模块文件重命名为`.mjs`文件，如果不想这么麻烦可以在`package.json`里配置type项：

```json
{
    "type": "module"
}
```

但这样以后所有的js文件都会默认以ES Module的规范加载，也就是说不能再使用CommonJS的那套语法了（比如`require`）

这个时候如果还想保留CommonJS的用法，需要把使用CommonJS的模块重命名为`.cjs`文件。