---
title: vue开发规约
date: 2017-05-28 17:50:55
tags: front-end
---
为了让vue项目代码更好的组织和架构，我们在开发vue前端项目时，我们最好在开发工具和代码书写方式上遵循一些基本的约定。

<!-- more -->

### 开发工具约定

1. 使用IDE或Webstorm
2. Vue.js单文件组件的高亮及语法支持<http://blog.csdn.net/Dcatfly/article/details/53123318>
3. 统一代码缩进格式：
	* 打开editor-code style-javascript，将tabsize、indent、continuation indet设置为2
	* 项目根目录中添加.editorconfig文件，配置代码格式
	  ```
		root = true
		[*]
		charset = utf-8
		indent_style = space
		indent_size = 2
		end_of_line = lf
		insert_final_newline = true
		trim_trailing_whitespace = true
	  ```

### js约定

1. 变量名和方法名采用驼峰命名法，尽量让命名具有识别性。
2. 保持代码干净，不必要的字段和方法记得删除
3. 尽量采用箭头表达式，去除掉`let that=this`之类的代码，箭头表达式可以根据上下文自动关联到外部的this对象
4. 一个方法内的代码，尽量不要超过30行
5. 同一段代码有两三处都使用了，一定要去抽取出来
6. js变量申明采用const或let，避免使用var

### css约定

1. 书写顺序　　
  * 位置属性(position, top, right, z-index,display, float等)　　
  * 大小(width, height, padding, margin)　　
  * 文字系列(font, line-height, letter-spacing,color- text-align等)　　
  * 背景(background, border等)　　
  * 其他(animation, transition等)
2. 如果设置的css属于只有两三个可采用内联样式（style），否则采用css类添加样式
3. css类命名一律小写，连词时用中槓（nav-image)，尽量不缩写，除非一看就明白的单词
4. 公共样式单独写入公共文件中，后期可用less做变量或者混合使用

### html约定

1. vue绑定属性v-bind:请简写成:(v-bind:class-->:class)
2. vue绑定时间v-on:请简写成@(v-on:click-->@click)
3. 当vue组件的文件名为InfoAuthItem，在vue2.x版本中可以直接使用，不需要转换成小写
```
<InfoAuthItem title="金融认证"
                  content="提供个人客户消费贷、车商信用贷等金融服务"
                  :image="authFinance" @clickAuth="clickFinanceAuth" :status="financeAuthStatus"/>
<Divider/>
```

### 其他约定

1. 如果一个效果或样式，会出现两次及以上请将其封装成组件
2. 如果一个组件属于属于某一种业务特有的，请将其放入该业务包中的components中
3. 所有vue组件文件命名采用PascalCase（InfoAuthPage.vue），页面级别的组件以Page结尾（InfoAuthPage.vue）
4. 确保代码能够在chrome上调试
5. Store中只存放公共数据（即该页面数据的生命周期不会随着页面destory而消亡），另外这些数据可采用module化进行管理，具体参考[vuex如何实现模块](https://vuex.vuejs.org/zh-cn/modules.html)
6. 文件命名：
  * vue组件文件命名采用PascalCase（如InfoAuthItem);其它js文件采用camelCase(commonUtil)
  * css或less独立文件命名采用小写，连词时用中槓
  * 图片等其它资源文件采用采用小写，连词时用中槓

#### 目录结构

```
    ├── README.md                       项目介绍
    ├── index.html                      入口页面
    ├── build                           构建脚本目录
    │   ├── build-server.js                 运行本地构建服务器，可以访问构建后的页面
    │   ├── build.js                        生产环境构建脚本
    │   ├── dev-client.js                   开发服务器热重载脚本，主要用来实现开发阶段的页面自动刷新
    │   ├── dev-server.js                   运行本地开发服务器
    │   ├── utils.js                        构建相关工具方法
    │   ├── webpack.base.conf.js            wabpack基础配置
    │   ├── webpack.dev.conf.js             wabpack开发环境配置
    │   └── webpack.prod.conf.js            wabpack生产环境配置
    ├── config                              项目配置
    │   ├── dev.env.js                      开发环境变量
    │   ├── index.js                        项目配置文件
    │   ├── prod.env.js                     生产环境变量
    │   └── test.env.js                     测试环境变量
    ├── mock                            mock数据目录
    │   └── hello.js
    ├── package.json                    npm包配置文件，里面定义了项目的npm脚本，依赖包等信息
    ├── src                             源码目录    
    │   ├── main.js                         入口js文件
    │   ├── app.vue                         根组件
    │   ├── components                      公共组件目录
    │   │   └── title.vue
    │   ├── assets                          资源目录，这里的资源会被wabpack构建
    │   │   ├── img
    │   │   │   └── logo.png
	│	│	└── css                          css文件目录 
    │   │       ├── common.less
	│	│		└── normalize.less
	│	├── utils                            工具类
	│	├── mixins                           混合类
	│	├── net                              网络相关
	│	├── plugins                          自定义插件
	│	│
    │   ├── routes                          前端路由
    │   │   └── index.js
    │   ├── store                           应用级数据（state）
    │   │   ├── index.js                    整个应用级别数据
	│	│	└── modules                     子模块数据
	│	│		└── home.js
    │   └── views                           页面目录
    │       ├── auth                        认证业务
	│		│	├── components              与认证业务相关联的特有组件
	│		│	│	└── InfoAuthItem.vue
	│		│	└── InfoAuthPage.vue	    认证状态列表页面
    │       └── notfound.vue                路由找不到时，跳转的页面
    ├── static                          纯静态资源，不会被wabpack构建。

```

### 参考
1. <http://www.tuicool.com/articles/za6rMfY>
2. <https://www.zhihu.com/question/19586885>
3. <http://blog.csdn.net/Dcatfly/article/details/53123318>
4. <https://vuex.vuejs.org/zh-cn/modules.html>





