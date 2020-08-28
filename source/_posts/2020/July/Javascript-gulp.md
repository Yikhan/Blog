---
title: 自动化构建工具Gulp
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-07-09 21:48:19
tags:
  - javascript
categories:
  - 编程技巧
  - 前端学习笔记
index_img: /img/glup.png
---

# 自动化构建工具Gulp

Gulp作为当下最流行的前端自动化构建工具，功能强大，插件丰富而且简单易上手。虽然最近Gulp的功能被全能的Webpack蚕食了不少，很多项目已经完全依赖Webpack构建了，但Gulp本身还是非常好用的，值得一学。

## 1. 基本使用

通过yarn/npm安装gulp之后，需要配置的有两个地方，一个是gulp的入口文件gulpfile.js，另一个则是安装gulp的一系列插件

gulp自带的最常用的命令：

| 方法       | 作用                                            |
|------------|-----------------------------------------------|
| `src`      | 从指定目录构建一个输入流                        |
| `dest`     | 构建一个输出流到指定目录                        |
| `parallel` | 并行执行任务                                    |
| `series`   | 顺序执行任务                                    |
| `watch`    | 跟踪一个路径内的文件变化，有变化时触发指定的任务 |

我们来构建一个最基本的gulp任务：

```js
const style = () => {
  return src('src/assets/styles/*.scss', { base: 'src' }) // base可以指定要忽略的路径符，之后的路径会被完整复制到目标路径
    .pipe(plugins.sass({ outputStyle: 'expanded' })) // outputStyle控制编译后的css文件风格
    .pipe(dest('temp'))
}
```

一个任务就是一个函数，需要return一个函数执行，一般我们使用gulp的目的就是自动化打包文件，所以常常先使用`src`构造一个文件输入流，从目录读取文件，经过处理后再通过`dest`输出到指定的打包目录。gulp对文件流使用了类似管道pipe的处理概念，使用`pipe()`可以使用gulp的插件或者自定义方法对文件流进行任意操作。

同理，下面这样也是一个gulp任务

```js
const clean = () => {
  return del([ 'dist', 'temp' ])
}
```

使用node自带的del库对目标文件夹进行清理

## 2. 自动加载插件

一般安装插件后我们需要`require`来使用，但通过安装一个专门用于加载插件的插件`gulp-load-plugins`，我们可以把这个过程进一步简化，因为gulp的插件都有固定的命名方式`gulp-<name>`，通过`gulp-load-plugins`可以使用`plugins.name`的方法来直接调用（命名转为驼峰式）

```js
const loadPlugins = require('gulp-load-plugins')
const plugins = loadPlugins() // 之后便可在pipe里使用plugins.sass/plugins/babel这样来使用
```

当然不要忘了先安装这些插件，`gulp-load-plugins`本身并不会帮你安装任何插件，只是把加载方式简化了。

## 3. 使用Brower-Sync调试

在gulp打包编译完成之后一般我们会打开浏览器进行调试预览，通过`brower-sync`这个库可以很好地简化这个流程，`brower-sync`本身并不依赖于gulp，二者是完全独立的库，gulp可以非常轻松地和其结合使用。

![image-20200709221730499](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200709221730499.png)

```js
const browserSync = require('browser-sync')
const bs = browserSync.create()

bs.init({
    notify: false, // 不弹窗提示
    port: 2080, // 端口号
    open: true, // 是否马上打开浏览器页面
    server: {
        baseDir: [ 'temp', 'src', 'public' ], // 项目目录
        // 额外配置特定路由的转发
        // 比如一些html文件里通过src='path'导入的路径/node_modules并不在上面三个目录里面
        // 所以要转发到我们开发根目录下的node_modules
        routes: {
            '/node_modules': 'node_modules'  
        }
    }
})
```

配置可以见官网

https://browsersync.io/docs/options

## 4. 使用watch跟踪文件变化

非常简单，watch的第一个参数是要跟踪的路径，可以使用通配符，第二个就是要执行的任务。

```js
watch('src/assets/styles/*.scss', style)
```

要注意的是watch的函数签名为

```js
watch(globs, [options], [task])
```

所以可以最多接受三个参数，路径可以是一个数组，比如`['input/*.js', '!input/something.js']`，task是一个定义好的任务，可以用在这里用`series`和`parallel`来组合任务。

## 5. Useref替换资源路径

在开发过程中，一些资源有时候我们可能是分散引入的，比如从node_modules下面引入，或者从多个js/css文件中引入。我们在打包时希望把这些用到的文件全部整合到一起然后放到一个指定的目录下面，useref插件就是做这个的。

```js
const useref = () => {
  return (
    src('temp/**/*.html', { base: 'temp' })
      .pipe(plugins.useref({ searchPath: [ 'temp', '.' ] })) // 引用资源搜索的目录，useref会先到temp下面找，找不到就到下一个路径，以此类推
      // html js css
      .pipe(plugins.if(/\.js$/, plugins.uglify())) // 在进行打包的过程中还可以使用其他插件来压缩转化
      .pipe(plugins.if(/\.css$/, plugins.cleanCss())) // 这里的plugins.if也是一个gulp插件
      .pipe(
        plugins.if(
          /\.html$/,
          plugins.htmlmin({
            collapseWhitespace: true,
            minifyCSS: true,
            minifyJS: true
          })
        )
      )
      .pipe(dest('dist'))
  )
}
```

要注意的是到底合并哪些文件其实是需要在html内指定的

```html
<!-- build:js assets/scripts/vendor.js -->
<script src="/node_modules/jquery/dist/jquery.js"></script>
<script src="/node_modules/popper.js/dist/umd/popper.js"></script>
<script src="/node_modules/bootstrap/dist/js/bootstrap.js"></script>
<!-- endbuild -->
<!-- build:js assets/scripts/main.js -->
<script src="assets/scripts/main.js"></script>
<!-- endbuild -->
```

上面的注释非常重要，`build`和`endbuild`之间就是一个build block，其中的所有文件都会被useref提取打包整合成一个文件，而整合后的文件的文件名和存放路径也在注释中，比如上面的`vendor.js`和`main.js`

## 6. 一个完整的gulpfile.js文件

最后我们来看一个完整的定义文件

```javascript
const { src, dest, parallel, series, watch } = require('gulp')

const del = require('del')
const browserSync = require('browser-sync')
const loadPlugins = require('gulp-load-plugins')

const plugins = loadPlugins()
const bs = browserSync.create()

const clean = () => {
  return del([ 'dist', 'temp' ])
}

const style = () => {
  return src('src/assets/styles/*.scss', { base: 'src' })
    .pipe(plugins.sass({ outputStyle: 'expanded' }))
    .pipe(dest('temp'))
    .pipe(bs.reload({ stream: true }))
}

const script = () => {
  return src('src/assets/scripts/*.js', { base: 'src' })
    .pipe(plugins.babel({ presets: [ '@babel/preset-env' ] }))
    .pipe(dest('temp'))
    .pipe(bs.reload({ stream: true }))
}

const page = () => {
  return src('src/**/*.html', { base: 'src' })
    .pipe(plugins.swig({ data, defaults: { cache: false } }))
    .pipe(dest('temp'))
    .pipe(bs.reload({ stream: true }))
}

const image = () => {
  return src('src/assets/images/**', { base: 'src' })
    .pipe(plugins.imagemin())
    .pipe(dest('dist'))
}

const font = () => {
  return src('src/assets/fonts/**', { base: 'src' })
    .pipe(plugins.imagemin())
    .pipe(dest('dist'))
}

const extra = () => {
  return src('public/**', { base: 'public' }).pipe(dest('dist'))
}

const start = () => {
  // 代码文件
  watch('src/assets/styles/*.scss', style)
  watch('src/assets/scripts/*.js', script)
  watch('src/**/*.html', page)
  // 资源文件
  watch(
    [ 'src/assets/images/**', 'src/assets/fonts/**', 'public/**' ],
    bs.reload
  )

  bs.init({
    notify: false,
    port: 2080,
    open: true,
    server: {
      baseDir: [ 'temp', 'src', 'public' ],
      routes: {
        '/node_modules': 'node_modules'
      }
    }
  })
}

const useref = () => {
  return (
    src('temp/**/*.html', { base: 'temp' })
      .pipe(plugins.useref({ searchPath: [ 'temp', '.' ] }))
      // html js css
      .pipe(plugins.if(/\.js$/, plugins.uglify()))
      .pipe(plugins.if(/\.css$/, plugins.cleanCss()))
      .pipe(
        plugins.if(
          /\.html$/,
          plugins.htmlmin({
            collapseWhitespace: true,
            minifyCSS: true,
            minifyJS: true
          })
        )
      )
      .pipe(dest('dist'))
  )
}

// 并行执行文件编译
const compile = parallel(style, script, page)

// 打包上线之前执行的任务
const build = series(clean, parallel(compile, image, font, extra), useref)

// 开发编译并打开Broswer
const serve = series(compile, start)

module.exports = {
  clean,
  build,
  start,
  serve
}
```

### 基础任务

| 任务名   | 功能                                                                                                  |
|----------|-----------------------------------------------------------------------------------------------------|
| `clean`  | 清理temp和dist文件夹                                                                                  |
| `style`  | 编译scss样式文件为css文件                                                                             |
| `script` | 编译js文件，通过pipe先进行eslint语法校验再进行babel转译                                                |
| `page`   | 编译html模板文件，渲染swig内容                                                                         |
| `image`  | 打包静态图片资源                                                                                      |
| `font`   | 打包静态字体资源                                                                                      |
| `extra`  | 打包public目录下的其他静态资源                                                                        |
| `start`  | 启动浏览器显示项目，并使用watch跟踪src目录下的html/js/css文件变化进行热更新                            |
| `useref` | 对html文档中build block注释内的js/css资源导入进行处理，把引用到的文件进行合并打包，并在生成的html内替换 |

### 高级任务

| 任务名    | 功能                                                                            |
|-----------|-------------------------------------------------------------------------------|
| `compile` | 并行执行style/script/page任务，编译scss/js/html文件                              |
| `build`   | 先进行clean，然后并行执行compile/image/font/extra任务，最后执行useref进行引用替换 |
| `serve`   | 先执行compile在执行start，用于开发调试                                           |

### 开发

使用`yarn serve`命令即可编译项目并自动打开浏览器预览

### 发布

使用`yarn build`命令即可编译并打包所有资源到发布目录dist
