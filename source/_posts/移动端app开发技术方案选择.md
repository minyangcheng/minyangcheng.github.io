---
title: 移动端app开发技术方案选择
date: 2017-05-06 11:02:21
tags: android front-end
---
目前移动端app开发大体的技术方案有：
1. 纯原生
2. 混合模式（主体原生+部分h5页面）
3. 运用react native或weex开发 
4. 混合模式（部分原生+主体前端spa页面）

我个人比较看好前端的发展，因此在项目中用的就是是混合模式（部分原生+主体前端spa页面），采用此方案时原生代码只负责书写webview和js桥接层，其它的全部由前端方式书写。

<!-- more -->

### 原生部分

原生部分主要负责书写webview相关的东西：
1. webview的壳
2. 对js开放的webview接口，例如照相、从相册选择图片等。

当用户打开app的时候，实际上是打开带有webview的原生页面，然后webview从网络加载指定的前端的spa页面。之后所有的操作就在前端spa页面中进行。

### 前端部分

大前端时代，前端框架层出不穷，我这里主要采用webpack+vue+vuex+vue-router+mint来搭建前端项目。

1. webpack前端项目构建打包工具，可以轻松使你能够在项目中使用es6语法。
2. vue是渐进式JavaScript框架，可以使你能够轻松实现组件化开发和数据绑定。
3. vuex采用集中式存储管理应用的所有组件的状态。相当于facebook提出的flux模式的简化版。
4. vue-router路由管理库，可以使你轻松实现spa项目。
5. mint采用vue开发的ui组件库，包含一些常用的组件。

### 此方案的技术点

* 原生部分：webview和js互相调用
* 前端部分：
  1. html、css、js、dom节点操作
  2. es6语法
  3. webpack打包技术
  4. vue、vuex、vue-router套件
  5. jquery
  6. nodejs、npm

其实这种技术方案，无外乎用的还是前端基础和android端基础。技术永远都在发展，我们只有打好技术才能更好的应对未来。

#### 参考
1. <http://www.w3school.com.cn/b.asp>
2. <https://developer.mozilla.org/zh-CN/>
3. <https://cn.vuejs.org/>
4. <https://router.vuejs.org/zh-cn/>
5. <https://vuex.vuejs.org/zh-cn/>