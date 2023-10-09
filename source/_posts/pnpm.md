---
title: 幽灵依赖与包管理工具
date: 2023-05-05 17:05:37
tags: [pnpm,幽灵依赖,包管理工具]
categories: [前端工程化]
---

## 包管理工具

最近接触了 vben-admin 这个开源项目，发现使用的包管理工具是 pnpm，而我之前一直都是用的 npm，想来不知他们有何差别，便去网上找了些资料和文档学习一下。

目前流行的包管理工具：npm，yarn，pnpm。pnpm的主要优势在于节省磁盘空间，install命令执行速度快，解决了幽灵依赖。

## node_modules 的目录结构

### 嵌套式

在早期的npm@2版本中，他的 node_modules 目录是嵌套式的。因此当一个依赖包内部依赖另一个包时，外部的依赖包目录里会再嵌套一层node_modules，里面存放着内部依赖包。

一个例子，demo-foo 和 demo-baz 都依赖了demo-bar 这个包，它被同时安装在demo-foo和demo-baz的node_modules下。

```
node_modules
└─ demo-foo
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ demo-bar
         ├─ index.js
         └─ package.json
└─ demo-baz
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ demo-bar
         ├─ index.js
         └─ package.json
```

虽然这种方式目录结构比较清晰，但它的缺点显而易见，如果多个依赖包都在内部依赖同一种依赖，由于没有复用机制，会造成磁盘空间的浪费；且如果嵌套层级太深，那么在windows中无法识别长度超过255个字符的路径，造成严重的问题。



### 扁平式

为了解决上述问题，后来的 npm@3+ 版本和新出的 yarn 对 node_modules 目录结构做出了改变，将原来嵌套式的目录结构拍平，所有的依赖包和依赖的依赖都存放在根node_modules目录下，再通过链接的方式，**共享**依赖的依赖。

跟上一个例子相同的场景，目录结构被拍平了，demo-foo和demo-baz共同依赖同一个demo-bar

```text
node_modules
└─ demo-bar
   ├─ index.js
   └─ package.json
└─ demo-baz
   ├─ index.js
   └─ package.json
└─ demo-foo
   ├─ index.js
   └─ package.json
```

这种方式解决了嵌套式目录路径过长和依赖不能复用的问题。但同时也引入了新的问题：幽灵依赖和分身问题

## 幽灵依赖

幽灵依赖是指：在 JS 代码中可以导入并没有在 `package.json `中出现的包。这是扁平化的node_modules目录引起的，它把所有的依赖和子依赖都置于最顶层。根据模块的加载机制，demo-bar即使没有被显式的install，但仍可以通过`import bar from 'demo-bar'`导入这个子依赖。

幽灵依赖会产生严重的问题：当有一天demo-foo和demo-bar被卸载，那么它们的子依赖也会不复存在，这时项目中引入的demo-bar及其API就会出现问题；再者，当更新父依赖的版本时，它的子依赖的版本也会被更新，一旦这次子依赖的更新导致当前项目中用到的子依赖API失效，那么这个问题是非常难以排查的。



## 分身问题

NPM分身问题是指：对于相同依赖的不同版本，npm只会将其中的一个版本提升到最顶层的node_modules，而剩下的其他版本则可能会被重新安装，并嵌套地安装在父依赖目录下。

> 具体提升哪个父依赖的相同子依赖，取决于哪个父依赖最早被安装

比如下面这个例子，当某一个其他依赖demo-bar的父依赖C更新版本时，C所需要的demo-bar是1.0.1版本，且C是最早被安装的demo-bar的父依赖，那么1.0.1版本的demo-bar会提升到顶级的node_modules，其他的父依赖重新安装1.0.0版本的demo-bar

```text
node_modules
└─ demo-bar // v1.0.1
   ├─ index.js
   └─ package.json
└─ demo-baz
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ demo-bar // v1.0.0
         ├─ index.js
         └─ package.json
└─ demo-foo
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ demo-bar // v1.0.0
         ├─ index.js
         └─ package.json
```

## pnpm的解决方案

### 硬链接和软链接

硬链接（hard link）和软连接，又称符号链接（symbolic link）是操作系统中两种共享文件的链接方式。

硬链接通过直接指向文件的索引块从而实现文件共享；软连接则创建一个新的文件（eg. 快捷方式），文件内容是要共享的文件的硬链接，示意图如下：



![软链接和硬链接](https://imgnothing.top/images/f1e85fbacc206f75936444f322c83c96.png)

### pnpm的目录结构

pnpm通过软链接、硬链接以及全局依赖库来组织目录结构，从而解决了分身问题和幽灵依赖。

pnpm将所有的包都安装在全局的 .pnpm-store 目录中（具体路径通过 `pnpm store path`查询），这是全局依赖库，以下称为store。所有项目的所需依赖都通过硬链接的方式从store链接到node_modules的.pnpm目录中，再用软连接，将.pnpm中的顶层依赖链接到node_modules目录下。

一个简单的例子：demo-baz@1.0.0和demo-foo@1.0.1是经显式install的顶层依赖，demo-bar是它们的子依赖。

```
node_modules
└─ .pnpm
   └─ demo-bar@1.0.0
      └─ node_modules
         └─ demo-bar -> <store>/demo-bar
   └─ demo-bar@1.0.1
      └─ node_modules
         └─ demo-bar -> <store>/demo-bar
   └─ demo-baz@1.0.0
      └─ node_modules
         ├─ demo-bar -> ../../demo-bar@1.0.0/node_modules/demo-bar
         └─ demo-baz -> <store>/demo-baz
   └─ demo-foo@1.0.1
      └─ node_modules
         ├─ demo-bar -> ../../demo-bar@1.0.1/node_modules/demo-bar
         └─ demo-foo -> <store>/demo-foo
└─ demo-baz -> ./pnpm/demo-baz@1.0.0/node_modules/demo-baz
└─ demo-foo -> ./pnpm/demo-foo@1.0.1/node_modules/demo-foo
```

好处：

+ .pnpm目录中采用硬链接，所以可以达到包复用的目的
+ .pnpm中依赖的node_modules中采用的仍然是扁平式目录，无论demo-foo中嵌套多少层子依赖，它的文件目录的深度依然不变
+ 由于顶层依赖中没有了子依赖，所以就避免了幽灵依赖
+ 不同版本的相同子依赖都会存在于顶层.pnpm下，因此某个依赖更新子依赖并不会影响其他依赖的子依赖，依然可以通过硬链接实现依赖的复用，这就解决了分身问题。



## pnpm的局限性

1. 由于符号连接（symbolic link）在一些场景下有兼容性问题，目前在 Eletron 以及 labmda 部署的应用上无法使用 pnpm
2. 由于全局公用同一份store，因此当某个项目修改node_modules中的内容时，会直接影响全局store中对应的内容，这会对其他的项目造成影响。关于这个问题，最好的解决方法是clone（copy-on-write写入前复制）：修改前，创建一个新的引用指向当前的文件。
3. 并不是所有的命令pnpm都很快，例如pnpm run 就比较慢 
