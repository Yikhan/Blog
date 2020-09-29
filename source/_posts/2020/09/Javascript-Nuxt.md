---
title: Nuxt服务端渲染项目部署到阿里云
banner_img: 'https://wallroom.io/img/1920x1080/bg-02fdff7.jpg'
date: 2020-09-29 21:35:12
tags:
  - javascript
  - Vue
  - Nuxt
categories:
  - 前端学习笔记
index_img: /img/nuxt.jpg
---

# Nuxt服务端渲染项目部署到阿里云

## asyncData方法

### 1. 基本使用

>官方文档：<https://zh.nuxtjs.org/guide/async-data>

`asyncData`方法是Nuxt自带的一个异步数据获取方法，是Nuxt实现服务端渲染最重要的api

- 它会将返回的数据合并到Vue组件的data里面。
- 调用时机：服务端渲染期间和客户端路由更新之前。

注意事项

- 只能在页面组件（pages）中使用，普遍组件不会触发该方法。
- 没有`this`，因为它是在组件初始化前被调用的。
- 官方推荐和`axios`一起使用。

```js
export default {
  async asyncData() {
    const res = await axios({
      method: 'GET',
      // 注意要使用完整的路径，服务端渲染时访问的根路径和客户端不同
      url: 'http://localhost:3000/data.json'
    })
    return res.data
  }
}
```

### 2. Context上下文

> 官方文档：<https://zh.nuxtjs.org/api/context/>

`asyncData`方法同样可以接受一个自带的默认参数`context`，这个参数里包含了很多有用的内置信息。

比如路由信息，由于`asyncData`内部不能使用`this`，也就无法通过常用的`this.$route.params`获取路由参数，所以要通过`context`来获取。

```js
export default {
  async asyncData(context) {
    const res = await axios({
      method: 'GET',
      url: 'http://localhost:3000/data.json'
    })
    const id = Number.parseInt(context.params.id) // 从context中获取动态路由参数
    return {
      article: data.posts.find(item => item.id === id)
    }
  }
}
```

## 部署Nuxt到阿里云服务器

### 1. 设置阿里云服务器

首先当然是到[阿里云官网](https://cn.aliyun.com/price/product#/ecs/detail)购买服务器了

这里我选的是`突发性能型ECS.T5` CentOS系统 一月的价格约100RMB，如果按周买的话更便宜，适合测试和练习用

购买服务器之后，首先需要知道服务器的公网ip地址，然后打开端口，一般应用会被部署到80或者3000端口。

在`本实例安全组` -> `配置规则`菜单里面可以打开指定端口：

![image-20200929140818321](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200929140818321.png)

### 2. 连接服务器

打开本地的CMD或者任何其他Terminal，然后通过ssh命令连接服务器

```bash
ssh root@47.74.68.207
```

这里使用的是管理员账号root，如果在服务器里创建了其他账号当然也可以使用

连接需要输入密码，为了简化过程避免每次连接都要输入一次密码，我们可以配置ssh秘钥

```bash
# 生成公钥 秘钥名字是自定义的 一个为私钥 另一个为公钥.pub
# 私钥 key_rsa
# 公钥 key_rsa.pub
cd C:\Users\Administrator\.ssh
ssh-keygen

# 把公钥拷贝到服务器 
scp key_rsa.pub root@47.74.68.207:root/.ssh
```

这里`root/.ssh`是CentOS系统在根目录下的一个隐藏文件夹，需要用`ls -a`命令才能看到

然后在服务器端我们需要配置authorized_keys文件，让系统知道这个公钥

```bash
cd ~/.ssh
# 找到 authorized_keys 文件
# 把 nllcoder_com_rsa.pub 文件内容追加到 authorized_keys 文件末尾
cat >> authorized_keys < nllcoder_com_rsa.pub
# 重启 ssh 服务
systemctl restart sshd
```

最后在本地我们还需要修改本机的`.ssh/config`文件，这个文件可以配置很多组连接和对应使用的私钥，加入如下一组即可：

```bash
Host 47.74.68.207
HostName 47.74.68.207
User root
PreferredAuthentications publickey
IdentityFile C:\Users\Administrator\.ssh\key_rsa
```

### 3. 安装Node.js

如果直接在CentOS上手动安装Node.js，一个常见的问题就是装了之后node命令依然用不了，因为还需要设置环境变量，所以更方便的方法是使用nvm安装Node.js

[https://github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm)

```bash
# 查看环境变量
echo $PATH

# nvm官方的安装脚本
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
nvm --version

# 查看环境变量
echo $PATH

# 安装 Node.js lts
nvm install --lts

# 如果使用yarn的话还需要安装
curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
sudo yum install yarn
```

yarn的安装和npm类似，也非常简单

[yarn安装文档](https://classic.yarnpkg.com/en/docs/install/#centos-stable)

然后我们还需要安装pm2，这个工具可以管理Node进程，使得我们可以同时开启多个Node进程而不会使得控制台阻塞。

```bash
i pm2 -g
```

有了pm2之后我们就可以使用pm2来启动项目

```bash
# 以前的运行方式 start指令是我们在package.json里面配置好的 等于nuxt start
npm run start
yarn start

# Linux系统
# 使用pm2启动npm
pm2 start npm -- run start
# 使用pm2启动yarn
pm2 start yarn -- start

# Windows系统
pm2 start "C:\Program Files\nodejs\node_modules\npm\lib\npm.js" -- run start
pm2 start "C:\Users\fresh\AppData\Roaming\npm\node_modules\yarn\bin\yarn.js" -- start
```

`--`后面（注意有个空格）的参数就是传递给前面npm/yarn的运行参数。

这里有个坑，如果在windows系统上运行pm2的话，会找不到npm/yarn命令，需要手动指定npm/yarn的路径。

所幸pm2一般我们也不需要在本地的windows系统上运行（本地可以随便开多个Terminal）

最后为了方便运行，我们还可以在根目录下建立一个配置文件[`ecosystem.config.js`](https://pm2.keymetrics.io/docs/usage/application-declaration/)来记录pm2的配置信息：

```js
module.exports = {
  apps: [
    {
      name: 'RealWorld',
      script: 'yarn',
      args: 'start',
      watch: '.',
      interpreter: '/usr/bin/bash'
    }
  ]
}
```

注意如果使用yarn的话要设定`interpreter`这个选项，因为yarn命令是使用bash而不是node来执行的，而pm2的默认运行环境是node

### 4. 部署Nuxt项目

#### 手动部署

首先在项目根目录下执行`nuxt build`生成`.nuxt`文件夹，里面包含了完整的项目打包结构。

我们需要把以下的所有文件复制到服务器上，最简单的办法就是先把它们打包成一个压缩包realworld.zip

```bash
.nuxt
nuxt.config.js
package.json
package-lock.json
ecosystem.config.json
```

具体操作

```bash
# 服务器根目录下建立home/realworld文件夹
mkdir realworld

# 本地执行
scp ./realworld.zip root@47.74.68.207:home/realworld

# 服务器 进入路径
cd realworld

# 解压
unzip realworld.zip

# 安装依赖 使用npm或者yarn都可以
yarn install

# 运行
pm2 start ecosystem.config.js
```

### 自动部署

手动显然效率很低，实际项目里必须使用自动化构建来简化操作，我们这里可以使用免费的Github Actions

Actions的配置非常简便，直接在项目根目录下创建一个`.github/workflows`文件夹，其中可以创建任意名字的`.yml`文件，每个文件就是一个workflow

例如我们新建一个`main.yml`文件：

```yml
name: Publish And Deploy Demo
on: 
  push:
    tags:
      - v*

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:

    # 下载源码
    - name: Checkout
      uses: actions/checkout@master

    # 打包构建
    - name: Build
      uses: actions/setup-node@master
   
    - run: yarn install
    - run: yarn build
    - run: tar -zcvf release.tgz .nuxt nuxt.config.js package.json yarn.lock ecosystem.config.js

    # 发布 Release
    - name: Create Release
      id: create_release
      uses: actions/create-release@master
      env:
        GITHUB_TOKEN: ${{ secrets.TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false

    # 上传构建结果到 Release
    - name: Upload Release Asset
      id: upload-release-asset
      uses: actions/upload-release-asset@master
      env:
        GITHUB_TOKEN: ${{ secrets.TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: ./release.tgz
        asset_name: release.tgz
        asset_content_type: application/x-tgz

    # 部署到服务器
    - name: Deploy
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        password: ${{ secrets.PASSWORD }}
        port: ${{ secrets.PORT }}
        script: |
          cd home/realworld
          wget https://github.com/Yikhan/RealWorld_Nuxt/releases/latest/download/release.tgz -O release.tgz
          tar zxvf release.tgz
          yarn install --production
          pm2 reload ecosystem.config.js
```

[workflow官方文档](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions)

一开始的on表示在什么事件时触发actions，这里我们选择在push任意tag的时候触发

可以看到workflow的文件模式很简单易读，每个step都有固定的语法格式

| 关键字 | 意思                                            |
| ------ | ----------------------------------------------- |
| name   | 步骤的名字，会显示在构建时的页面上              |
| uses   | 使用什么actions，这些是github actions提供的指令 |
| env    | 环境变量，主要是token，用户操作repo             |
| with   | 运行actions时提供的参数                         |

其中花括号里面的是我们自定义的一些变量，主要是通过`secrets`来设置，在项目的Settings里面可以找到，我们连接阿里云服务器所需的用户名，密码，端口号（ssh端口默认是22），还有操作我们自己的repo所需的github token都可以在这里设置。github token可以在自己账号的Settings/Developer settings里面选择Personal access tokens来生成。

![image-20200929212453768](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200929212453768.png)

然后我们直接在本地代码设置好tag之后就可以推送了

```bash
git tag v0.1.1
git push
```

![image-20200929212950523](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200929212950523.png)

action被触发后我们设定好的workflow就会依次执行，然后代码就会被自动部署到阿里云服务器上了。

![image-20200929213121446](https://cdn.jsdelivr.net/gh/Yikhan/ImageHost/blog/image-20200929213121446.png)
