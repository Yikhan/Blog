---
title: Webpack进阶
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-07-17 11:59:38
tags:
  - javascript
  - webpack
categories:
  - 前端学习笔记
  - 自动化工具
index_img: /img/webpack.png
---

# Webpack进阶

## 1. `Dev-Server`

`webpack-dev-server`是使用率最高的webpack插件之一，其主要特点就是集成了打包和浏览器加载以及热更新这一套组合拳，特别是其打包后的文件是在内存中的，并不会真正生成到dist目录，避免了大量的重复磁盘读写。

安装：

```bash
yarn add webpack-dev-server --dev
```

启动方式：

```bash
yarn webpack-dev-server // 默认从根目录打包
```

老样子，安装后需要在`webpack.config.js`里配置

```js
module.exports = {
  devServer: {
    contentBase: './public',
    proxy: {
      '/api': {
        // http://localhost:8080/api/users -> https://api.github.com/api/users
        target: 'https://api.github.com',
        // http://localhost:8080/api/users -> https://api.github.com/users
        pathRewrite: {
          '^/api': ''
        },
        // 不能使用 localhost:8080 作为请求 GitHub 的主机名
        changeOrigin: true
      }
    }
  }
}
```

`contentBase`表示静态资源路径，这个目录下的文件不会被打包，有利于提高开发时的打包效率（静态文件一般只在最后打包上线时才应该被真正打包处理）

`proxy`表示代理转发，这是一个非常有用的配置。在本地开发阶段时，一般本地我们的网站都是以`localhost:8080`这类网址来运行的，当需要请求一些api接口时就会出现跨域问题，这时就要通过`proxy`来转发。

## 2. `source-map`

`source-map`是一种位置信息文件，主要保存了打包后的代码与源代码之间的对应关系，在调试的时候极为有用。

在webpack里配置`source-map`，主要使用devtool这个属性

```js
module.exports = {
  devtool: 'source-map' // 可以换成其他生成模式
}
```

然后就会生成`source-map`文件，比如`bundle.js.map`

webpack支持多种`source-map`的生成方式，其性能和效率各有不同

![image-20200715191338417](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200715191338417.png)

不同的模式从命名上就能看出基本的特点

- eval - 是否使用eval实行模块代码
- cheap - 是否包含行信息
- module - 是否能得到Loader处理之前的源代码
- inline - 将`source-map`以data-url的形式嵌入到代码中（会增大代码文件体积）
- hidden - 生成`source-map` 但是不在打包后的代码文件中引入（也就是提供`source-map`但是不用）
- nosources - 不显示源代码（会显示错误位置的行列信息），主要用于生产环境里保护源代码不暴露

------

在开发环境下首先推荐使用的是`cheap-module-eval-source-map`这个模式

其优势如下：

1. 因为一般只要代码风格控制得当，每行不长，行信息就足够了，不需要列信息。

2. 现在项目都会使用多个Loader多次打包，所以显示原始的源代码是有必要的。

3. 这个模式首次打包慢但是rebuild很快，符合我们经常调试时频繁重新打包的需要

而在生产环境下，建议选择`none`，不要提供`source-map`

## 3. Hot Module Replacement

HMR可以算是webpack中最为好用的功能之一，所谓HMR就是只替换有改动的文件而不用刷新整个页面，这极大地提高了开发效率。

`webpack-dev-server`本身就支持HMR，可以通过命令参数开启

```bash
yarn webpack-dev-server --hot
```

也可以通过配置，这时还需要引入webpack内置的`HotModuleReplacementPlugin`插件

```js
const webpack = require('webpack')

module.exports = {
  devServer: {
    hot: true
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ]
}
```

但是这还不够，这只能保证css等样式代码自动HMR，js代码却不行，页面依然会刷新。

这是因为js代码过于灵活，webpack无法判断要如何执行HMR，必须要用户来指定更新方法。

我们需要使用`HotModuleReplacementPlugin`提供的`module.hot`API

```js
import createEditor from './editor'
import background from './better.png'
import './global.css'

const editor = createEditor()
document.body.appendChild(editor)

const img = new Image()
img.src = background
document.body.appendChild(img)

// ============ 以下用于处理 HMR，与业务代码无关 ============
if (module.hot) {
  // js模块的HMR
  let lastEditor = editor // 保存之前的页面数据
  module.hot.accept('./editor', () => {
    console.log('editor 模块更新了，需要这里手动处理热替换逻辑')

    const value = lastEditor.innerHTML
    document.body.removeChild(lastEditor)
    const newEditor = createEditor()
    newEditor.innerHTML = value // 恢复更新前的页面数据
    document.body.appendChild(newEditor)
    lastEditor = newEditor
  })
  // 图片的HMR
  module.hot.accept('./better.png', () => {
    img.src = background
    console.log(background)
  })
}
```

这些代码在webpack打包后会被移除，所以并不用担心影响生产环境。

但依然会发现原生的HMR虽然强大但是要编写这些替换逻辑比较繁琐，而这也是开发者现在更愿意使用`vue-cli`，`create-react-app`这些现成的集合框架，它们已经内置了替换逻辑，可以直接进行HMR

## 4. 多配置文件

在实际项目中，往往需要针对不对的环境配置相应的`webpack.config`文件

```reStructuredText
-- webpack.common.js
-- webpack.dev.js
-- webpack.prod.js
```

比较常用的方法是将通用的部分抽出为`webpack.common.js`文件，然后在各自的配置里使用合并，这时我们可以使用`webpack-merge`这个插件

```js
const merge = require('webpack-merge')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')
const CopyWebpackPlugin = require('copy-webpack-plugin')
const common = require('./webpack.common')

module.exports = merge(common, {
  mode: 'production',
  plugins: [
    new CleanWebpackPlugin(),
    new CopyWebpackPlugin(['public'])
  ]
})
```

其提供的`merge`函数可以把不同配置的属性针对性地合并，比如这里的`plugins`，我们并不想覆盖掉common里的插件数组，而是合并。

然后运行webpack的时候就要指定配置文件

```bash
yarn webpack --config webpack.prod.js
```

也可以把这个命令集成到`package.json`的`scripts`里简化

```json
{
  "scripts": {
    "builid": "webpack --config webpack.prod.js"
  }
}
```

## 4. Tree-Shaking

Tree-Shaking可以在打包是去除未被使用到的代码，有效减小代码体积。webpack默认在production会自动进行Tree-Shaking，其他模式下要手动开启，主要就是使用`optimazation`属性。

```js
module.exports = {
  mode: 'none'， // 非production模式
  entry: './src/index.js',
  output: {
   filename: 'bundle.js'
  },
  optimization: {
    usedExports: true, // 在打包时仅导出使用到的成员变量
    minimize: true // 压缩移除掉未被引用到的代码
  }
}
```

`usedExports`可以在打包阶段找到哪些是被引用到的成员，将其导出，而剩下的未被引用成员就会在`minimize`阶段被移除。

`optimization`里还有一个`concatenateModules`选项，可以尽可能把代码合并到一个模块函数内，提升运行效率和再次减小代码体积，这个功能是webpack3加入的，也被称作`scope hoisting`

```js
optimization: {
  concatenateModules: true
}
```

------

Tree-Shaking只能对ES Module起效，这点尤其需要注意，如果使用了Babel的话可能会因为js被编译成了CommonJS而导致Tree-Shaking失效。

目前新版的Babel不会导致这个问题，我们也可以通过强制指定编译模式来确保：

```js
rules: [
  {
    test: /\.js$/,
    use: {
      loader: 'babel-loader',
      options: {
        presets: [
          ['@babel/preset-env', { modules: false }]
        ]
      }
    }
  }
]
```

注意presets的配置，添加选项对象时要嵌套两层数组

## 5. SideEffect

Side-Effect顾名思义表示副总用，如果一个模块没有副作用，就表示除了export，其余代码不会造成任何额外变化（类似纯函数的概念），我们可以在模块对应的`package.json`里标注`sideEffects: false`来表示这点，然后webpack就会进一步移除掉未被引用的代码（这个功能在开发npm库时可能使用更多）

production环境下是默认开启的

```js
optimization: {
  sideEffects: true
}
```

这个功能的意义在于Tree-Shaking是通过代码是否被引用来判断优化的，但有些时候代码可能被import了却并没有真正使用，比如下面这种整合了局部组件的情况，虽然最后可能只有Button被真正使用了，但Link由于也被引入了所以也会被一起打包进来。

```js
export { default as Button } from './button.js'
export { default as Link } from './link.js'
```

## 6. 代码分割/分包

在项目越来越大以后，我们不希望把所有代码都打包到一个bundle.js里面，这会严重拖慢加载速度。理想情况是用户打开了哪个页面就加载哪个部分，因此代码分割打包就势在必行。

### 6.1 多入口打包

很简单，从多个js文件入口开始打包，生成各自的打包文件。

```js
module.exports = {
  entry: {
    index: './src/index.js',
    album: './src/album.js'
  },
  output: {
    filename: '[name].bundle.js'
  },
}
```

这样之后还有个问题，假如我们使用了`HtmlWebpackPlugin`，每个生成的html都会把所有的打包js文件都导入一次，这显然不是我们所期望的。

所以我们还设置一个属性，就是chunks

```js
plugins: [
  new CleanWebpackPlugin(),
  new HtmlWebpackPlugin({
    title: 'Multi Entry',
    template: './src/index.html',
    filename: 'index.html',
    chunks: ['index']
  }),
  new HtmlWebpackPlugin({
    title: 'Multi Entry',
    template: './src/album.html',
    filename: 'album.html',
    chunks: ['album']
  })
]
```

这样每个html都只会引入自己的bundle.js打包文件了。

我们还可以让webpack帮我们提取各个模块的公共部分，避免分割以后重复打包。

```js
optimization: {
  splitChunks: {
    chunks: 'all'
  }
}
```

这样webpack会自动把公共部分打包到一个新文件里。

### 6.2 动态导入

也就是真正意义上的按需加载，使用ES Module的`import`函数，而不是在文件开头使用`import from`

```js
// import posts from './posts/posts'
// import album from './album/album'

const render = () => {
  const hash = window.location.hash || '#posts'

  const mainElement = document.querySelector('.main')

  mainElement.innerHTML = ''

  if (hash === '#posts') {
    import(/* webpackChunkName: 'components' */'./posts/posts').then(({ default: posts }) => {
      mainElement.appendChild(posts())
    })
  } else if (hash === '#album') {
    import(/* webpackChunkName: 'components' */'./album/album').then(({ default: album }) => {
      mainElement.appendChild(album())
    })
  }
}
```

上面代码里在`import `内部还使用了注释代码

> /* webpackChunkName: 'components' */

这样可以指定把动态导入的模块打包到哪里，不指定的话生成的bundle文件会以数字为序号。如果指定名一样的话就会打包到同一个文件里。

------

然后我们还可以让css文件也按需加载，通过安装使用一个插件`MiniCssExtractPlugin`把css从打包中抽取出来单独构成一个文件。

```js
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = {
  plugins: [
    new MiniCssExtractPlugin()
  ]
}
```

使用这个插件之后原来的`style-loader`就不需要了（因为css已经不在打包后的js里面了），我们要使用`MiniCssExtractPlugin`自带的loader

```js
{
  test: /\.css$/,
  use: [
    MiniCssExtractPlugin.loader,
    'css-loader'
  ]
}
```

要注意的，如果css不是很大（未超过150kb），就没有必要单独提取出来，和js打包在一起的性能会更好一些。

最后我们还需要压缩这个独立的css文件（webpack默认只能压缩js文件）

需要使用官方推荐的`optimize-css-assets-webpack-plugin`

安装后按照惯例配置插件：

```js
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const OptimizeCssAssetsWebpackPlugin = require('optimize-css-assets-webpack-plugin')

module.exports = {
  plugins: [
    new MiniCssExtractPlugin(),
    new OptimizeCssAssetsWebpackPlugin()
  ]
}
```

有意思的是，webpack其实建议把这个插件配置到minimizer数组里，而不是plugins数组。所有的压缩有关的插件都可以放到minimizer里面，使用参数统一控制是否压缩。

```js
module.exports = {
  optimization: {
    minimizer: {
      new OptimizeCssAssetsWebpackPlugin()
    }
  }
}
```

production打包时minimizer会自动开启

不过这样一来js就没法自动压缩了，因为minimizer这个自定义压缩配置会覆盖默认，所以还需要配置一个js的默认压缩插件

安装`terser-webpack-plugin`，这是webpack默认使用的js压缩插件

```js
const TerserWebpackPlugin = require('terser-webpack-plugin')

module.exports = {
   optimization: {
    minimizer: {
      new TerserWebpackPlugin(),
      new OptimizeCssAssetsWebpackPlugin()
    }
  }
}
```

## 7. Hash命名

我们可以看到很多项目打包后的bundle名字里都带了一串hash，这主要是为文件缓存服务的，hash编号可以区分文件是否有更新，如果不同则当做新文件缓存。这样可以解决新老文件同名时文件缓存不更新的问题。

一般常用的有三种hash命名方式：

|             命名方式             |                 效果                 |
| :------------------------------: | :----------------------------------: |
|    '[name]-[hash].bundle.js'     |         每次打包整体重设hash         |
|  '[name]-[chunkhash].bundle.js'  | 以chunk为单位，一个chunk共享一个hash |
| '[name]-[contenthash].bundle.js' |    文件级hash，最适合解决缓存问题    |

基本上配置里配置文件输出名的地方都可以使用hash:

```js
// 打包输出
output: {
  filename: '[name]-[contenthash:8].bundle.js'
}
// 插件里面也可以
plugins: [
  new MiniCssExtractPlugin({
    filename: '[name]-[contenthash:8].bundle.css'
  })
]
```

可以在hash属性后面增加`:number`来控制生成hash的长度，默认是20位。
