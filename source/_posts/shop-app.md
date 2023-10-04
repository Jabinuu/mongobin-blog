---
title: vue2电商练手项目
date: 2023-04-09 15:07:15
tags: [vue2,电商项目,vuex,vue-router,mock,代理]
categories: [vue2]
---

*完成了一个 Vue2 的的练手项目，记录部分项目开发流程和技术点*

### 创建 Vue 项目

+ 创建vue项目：在根目录cmd `vue create xxx`
+ 创建vue项目时就勾上Router，免得开发时再手动安装 vue-router 插件包。同理把 css preprocess也勾选上，免得手动安装 less loader 包



### eslint常见报错类型

+ 组件的命名必须是**多个**单词的驼峰命名
+ 组件通过 `import` 引入后必须注册，注册后必须使用，变量、方法同理
+ `import` 引入组件的文件时**不能**带上 .vue 后缀



### 文件树规范

+ node_modules 是包裹项目依赖的各个包的文件夹
+ public 中存放的是**静态页面** index.html （vue和react一般用来开发单页面应用，单页面应用自然要有一个html文件）和一些**静态资源**（图片）。放在public文件夹的资源在 webpack 打包时会原封不动的放到dist文件夹中

+ src 意为**源码**，顾名思义该目录是开发者写代码的文件夹
  + assets 也是放置**静态资源**（一般放置多个组件共用的静态资源）。与 public 不同的是，放置在 assets 文件夹中的静态资源，在webpack打包时会被当做一个模块，打包在 js 文件里
  + components 一般放置的是**非路由组件**（全局组件），不管跳转到哪个 page 页面上都会有的组件，通常以 html 标签的方式使用
  + pages 放置**路由组件**，即经路由跳转到的组件，通常用 `router-view` 来使用
  + router 项目当中配置的**路由‘器’**放置在 router 文件夹中。
  + App.vue 是项目中的**唯一根组件**
  + main.js 是程序的入口文件，也是整个程序当中最先执行的文件
+ babel.config.js 是 babel 的配置文件。可以把 babel 理解为翻译官，负责把 ES6 语法翻译为 ES5 兼容性更好
+ vue.confg.js 是 vue 项目的可选配置文件，运行时会被自动加载。用于配置打包dist文件夹的路径、webpack、Babel、eslint的设置等等，详见[https://cli.vuejs.org/zh/config/#](https://links.jianshu.com/go?to=https%3A%2F%2Fcli.vuejs.org%2Fzh%2Fconfig%2F%23)
+ package.json 可以理解为是项目的身份证，记录着项目的信息（项目叫什么，版本是多少，运行脚本是什么，依赖有哪些）
+ package-lock.json 缓存性文件，告诉我项目的依赖包是从哪里下载的
+ readme.md 说明性的文件



### 路由相关

**使用路由的流程**

​		在 index.js 引入 vue-router 插件 **=>** Vue.use() 注册引用的插件 **=>** 定义路由配置 **=>** 在main.js中引入定好的路由 **=>** 把引入的路由作为根组件的选项添加进去。

+ 前端的路由表现为 url : component 的键值对，这种 component 被称为路由组件。

+ 在使用时一定要**注册路由**：在 main.js 中用 `import router from '@/router/index.js' ` 引入 `router` ，并将其添加到 Vue 根组件的选项上。

+ 注册完路由，会发生神奇的事情，不管是非路由组件还是路由组件，他们身上都会获得 `$route` 和 `$router` 属性。`$route` ：一般获取**路由信息**（路径、query、params，meta），`$router` 是**路由操作**对象，用于实现编程式导航进行路由跳转（push | replace）。

+ **路由重定向**`{path: '/', redirect: '/home'}  ` ，打开域名直接跳转到首页

+ 路由的跳转分两种：

  + 声明式导航，用 `router-link` 代替 `<a>...</a>` 用户点击，直接进行路由跳转	

  + 编程式导航，用 `$router.push()` 和 `$router.replace()` 进行路由跳转。比如，用户点击登录时，我们不希望直接进行跳转，而是先进行账户验证后再发生跳转。  在跳转发生前后做一些动作，这就是编程式导航的典型应用场景

+ 路由重定向：将一个 url 重定向为指向另一个 url。

+ **路由元信息**：他是一个Object。有时，你可能希望将任意**信息附加**到路由上，如过渡名称、谁可以访问路由、在该路由下那些组件隐藏等。这些事情可以通过接收属性对象的`meta`属性来实现，并且它可以在路由地址和导航守卫上都被访问到。

+ **路由传参**分两种：

  + params参数：实现**动态路由**，必须在路由的 path 处添加 `:/xxx` 的占位符。
    + 当路由的path里设置了占位符，默认是必须传 params 参数的。如果路由跳转时却没有提供 params 参数，则跳转后的 url 中就会有 url 路径缺失的现象，这当然不是我们希望看到的。所以，尽量在路由中定义占位符时加上一个问号 `:/xxx?` 代表着 params 参数可传可不传。
  + query参数：即`path?xxx=1&xxx=2`这样用于搜索的参数。

  > 当需要路由传参时，推荐使用命名路由的name进行路由检索，当然这**只能**用于push的**对象参数**（另一种原始的参数是字符串参数）。
  >
  > 另外要注意：push的对象参数里，path 和 params是**不可同时使用**的，如果要传params，必须用name来指定路由

  + 给**路由组件传递props**：在路由定义处设置，常用函数式传递，既能传递 params 和query参数又能传递自定义数据。



### 页面组件结构

典型的上中下结构

![组件结构](http://182.44.49.100:34/images/2023/05/06/99a042307be2e6e58cd703214d0c07a4.png)



### 技术点

![技术点](http://182.44.49.100:34/images/2023/05/06/7801634e91006adb2b0a329401f8cbba.png)
