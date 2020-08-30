---
title: Webpack基础
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-07-17 11:58:38
tags:
  - javascript
  - webpack
categories:
  - 前端学习笔记
  - 自动化工具
index_img: /img/webpack.png
---

# Webpack基础

## 1. 安装

很简单，主要就是webpack和webpack-cli两个模块

```bash
yarn add webpack webpack-cli --dev
```

然后就可以使用命令打包了

```bash
yarn webpack
```

默认情况下webpack会把`src/index.html`作为打包入口，打包结果存放在dist文件夹下

## 2. 配置

在项目根目录下添加`webpack.config.js`，注意这个文件默认是在node环境下运行，所以采用CommonJS规范

```js
const path = require('path')

module.exports = {
  entry: './src/main.js', // 打包入口文件，可以是相对路径
  output: {
    filename: 'bundle.js', // 输出文件名称
    path: path.join(__dirname, 'dist') // 输出文件目录，必须是绝对路径
  }
}
```

webpack默认的打包模式为production，会自动进行代码合并压缩等优化操作

可以通过`--mode`来设置打包模式

除了production还有devlopment和none两种模式

```bash
yarn webpack --mode development/none
```

- development模式下webpack会优化打包速度和添加一些辅助调试的代码

- none模式是最原始基础的打包

当然也可以在配置中预先设置打包模式

```js
module.exports = {
  mode: 'development'
}
```

## 3. loader加载器

**webpack默认只能处理js文件**，其他文件需要安装对应的loader来处理，这也是webpack的核心所在。

loader的工作机制类似管道，所以可以把不同的loader组合起来使用，但要注意的是webpack要求最后的Result一定要是js代码。

![image-20200714154649412](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200714154649412.png)

### 3.1 loader分类

基本上webpack的loader有三类

- 把代码转换为js格式，比如`css-loader`
- 处理拷贝文件，导出文件路径，比如`file-loader`
- 辅助功能，代码校验，比如`eslint-loader`

### 3.2 `css-loader`/`style-loader`

比如要处理css文件就需要安装`css-loader`

```bash
yarn add css-loader --dev
```

然后需要在module属性的rules里配置规则，rules是一个数组，一个rule需要包括两个属性：

- `test` 匹配目标文件路径，一般使用regex
- `use` 需要使用的loader，不仅可以是名字，也可以是路径，和require类似

```js
module.exports = {
  module: {
    rules: [
      {
        test: /.css$/,
        use: 'css-loader'
      }
    ]
  }
}
```

然后还需要安装`style-loader`，因为`css-loader`只是把css转换为js数组，我们还需要一个loader来把转换好的css加载到页面上。

```bash
yarn add style-loader --dev
```

然后更改use为一个数组

```js
rules: [
  {
    test: /.css$/,
    use: [
      'style-loader',
      'css-loader'
    ]
  }
]
```

注意当use为数组时，执行顺序为从下往上

这里我们希望先执行`css-loader`之后再执行`style-loader`，所以把`style-loader`放上面。

webpack推荐以js文件为打包入口，在js里import引入css等其他所需的文件，这样有利于tree-shaking清理未被使用的代码，同时也符合webpack的设计思想---`围绕js来进行打包服务，当js需要使用其他资源时，应该就地引入`。这更贴近于现代组件化的前端开发模式（比如React，Vue等都是把css样式和组件放在一起，甚至直接使用css-in-js）

### 3.3 `file-loader`

打包图片等静态文件资源需要安装`file-loader`

```bash
yarn add file-loader --dev
```

然后同样的，需要设置rule

```js
{
  test: /.png$/,
  use: 'file-loader'
}
```

要注意的是webpack默认的资源路径是项目的根目录，并不是我们设置输出目录dist。也就是说，webpack默认打包文件输出的地方就应该是项目运行的根目录（很好理解，理想情况下dist就是最后部署运行的唯一目录）

所以一些情况下如果dist不是根目录，比如本地开发时我们可能仍然在原来的目录下去引用dist里面的打包文件，这时就要告诉webpack这个打包目录相对于真正的根目录在哪。

否则一些需要使用这些资源的地方就会报错。

打包过的bundle.js里可以看到图片文件的路径：

```js
module.exports = __webpack_require__.p + "aaa0e8af..3e18.png" // 文件名被替换成了hash，可以防止名字重复
```

里面的`__webpack_require__.p`就是我们需要设置的资源路径publicPath

```js
module.exports = {
  output: {
    filename: 'bundle.js',
    path: path.join(__dirname, 'dist'),
    publicPath: 'dist/' // 注意斜杠不能省略，因为会直接被拼接到路径里
  }
}
```

publicPath是很有用的属性，可以结合env来设置不同的值，比如有些时候我们希望发布后托管资源到云端或者其他地方，就可以根据需求修改publicPath

### 3.4 `url-loader`

`url-loader`可以把资源文件直接转换为url，这样就可以避免使用http的方式去请求文件。

这是因为url本身也是可以保存数据的

```html
data: [<mediatype>][;base64],<data>
 协议     媒体类型			编码		文件内容
```

比如下面的一个url，就直接传输了一个html页面数据

```html
data:text/html;charset=UTF-8,<h1>html content</h1>
```

把小文件转换为url是很有用的（大文件不应该使用url，因为转换后的js文件也会过大），比较好的方式是把10kb以下的文件转化为url，超过的则照常使用`file-loader`

首先安装`url-loader`

```bash
yarn add url-loader --dev
```

然后配置rule，不过这次use既不是字符串也不是一个数组，我们要使用一个对象，这样就可以在里面使用options来配置细节。

```js
{
  test: /.png$/,
  use: {
    loader: 'url-loader',
    options: {
      limit: 10 * 1024 // 单位是字节，这里表示10kb
    }
  }
}
```

现在`url-loader`就回自动对10kb以下的文件进行url转换了，不过要注意的是，`url-loader`对超过大小的文件默认还是会调用`file-loader`，所以仍然需要安装`file-loader`

### 3.5 `babel-loader`

由于webpack本身并不能识别ES6的新特性（除了import/export这两个打包必需的），当需要转换兼容js代码时，就又轮到了大名鼎鼎的`babel`出场（`bable`可能会迟到，但永远不会缺席...）

`babel-loader`还需要依赖`babel`的核心组件和插件，所以我们要安装以下三个：

```bash
yarn add babel-loader @babel/core @babel/preset-env --dev
```

然后当然还是要配置

```js
{
  test: /.js$/,
  use: {
    loader: 'babel-loader',
    options: {
      presets: ['@babel/preset-env']
    }
  },
  include: [ path.resolve(__dirname, 'src') ]
}
```

这里的配置基本和模块化一章中一样，关键依然是要指定presets，也就是告诉`babel`要使用什么插件（`preset-env`本身就是包含了新特性的一组插件集合）

注意还要使用`include/exclude`限定要处理的js文件路径，官方推荐的配置里使用了`exclude`来排除node_modules。在实际测试中，不使用限定的话运行打包后的代码时会报错。

> exclude: /(node_modules|bower_components)/

### 3.6 `html-loader`

`html-loader`用于打包html文件

```bash
yarn add html-loader --dev
```

要注意的是，webpack的任何loader或者多个loader管道式运行后最后生成的一定是js代码，所以`html-loader`的目的是把js中引用的html文件打包到js里面。

我们还需要配置如何处理html里面的资源引用问题。

```js
{
  test: /.html$/,
  use: {
    loader: 'html-loader',
    options: {
      attr: ['img:src', 'a:href']
    }
  }
}
```

默认情况下，`html-loader`只会对img的src属性进行处理，href里面引用的资源是不会被打包的，所以需要按照上面的格式来配置。

### 3.7 Webpack资源打包原理

webpack的资源打包也是有规则的，它类似一个爬虫，从入口文件开始爬取需要加载进来的资源。

默认情况下webpack会自动打包：

- ES Modules的import声明
- CommonJS的require
- AMD的define和require
- 样式代码(css)中的@import和url函数
- html代码中图片标签的src属性

### 4. 插件Plugin

插件是让webpack变得如此强大的另一个重要原因。插件可以解决打包任务之外的很多自动化操作需求，让webpack开始抢占gulp等传统自动化工具的地盘。

#### 4.1 `clean-webpack-plugin`

我们先来试试`clean-webpack-plugin`，它的作用是在每次打包前清理dist目录

```bash
yarn add clean-webpack-plugin --dev
```

在`webpack.config.js`里进行配置

```js
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

module.exports = {
  plugins: [
    new CleanWebpackPlugin()
  ]
}
```

现在每次打包前dist目录就会被自动清理了

#### 4.2 `html-webpack-plugin`

`html-webpack-plugin`可以在打包目录下生成一个自动导入所有打包模块的html文件，

```bash
yarn add html-webpack-plugin --dev
```

然后配置

```js
const HtmlWebpackPlugin = require('html-webpack-plugin') // 直接使用default导出即可

module.exports = {
  plugins: [
    new HtmlWebpackPlugin()
  ]
}
```

在打包后，dist目录下会自动生成一个html文件，有一点要注意的是，生成的html里对资源的引用路径和配置里的`publicPath`有关，对这个生成的html来说，因为已经在dist目录下了，就不需要`publicPath`了，需要删除这项配置。

```html
<script type="text/javascript" src="<publicPath> + bundle.js"></script>
```

这个插件的好处在于可以动态生成导入了bundle.js的html文件，方便直接从dist目录启动项目。

如果想要进一步配置这个html文件，可以传参给`HtmlWebpackPlugin`函数

```js
module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      title: 'Webpack Plugin Sample',
      meta: {
        viewport: 'width=device-width'
      },
      template: './src/index.html' // 可以使用自定义的html模板
    })
  ]
}
```

在自定义的模板里可以使用模板语法添加属性

```html
<h1><%= htmlWebpackPlugin.options.title %></h1>
```

同时，如果想要生成多个html文件也是可以的，只要再增加一个`HtmlWebpackPlugin`插件实例就行了。

```js
module.exports = {
  plugins: [
    // 默认的index.html文件
    new HtmlWebpackPlugin({
      title: 'Webpack Plugin Sample',
      meta: {
        viewport: 'width=device-width'
      },
      template: './src/index.html'
    }),
    // 再生成一个about.html文件
    new HtmlWebpackPlugin({
      filename: 'about.html' // 默认值就是index.html
    })
  ]
}
```

其他配置详见官网

<https://github.com/jantimon/html-webpack-plugin#options>

#### 4.3 `copy-webpack-plugin`

这个插件用于拷贝静态文件到dist目录

```bash
yarn add copy-webpack-plugin --dev
```

进行配置，要注意的是`CopyWebpackPlugin()`接受的是一个数组，成员是要拷贝文件的路径，可以是通配符或者文件夹，也可以是相对路径。

```js
const CopyWebpackPlugin = require('copy-webpack-plugin')

module.exports = {
  plugins: [
   new CopyWebpackPlugin({
      patterns: [ { from: 'public/**/*' } ]
    })
  ]
}
```
