---
title: 使用Hexo搭建个人博客
date: 2016-10-28 12:57:13
tags: 工具
---
hexo在github上搭建免费的个人博客。

<!--more-->

## 搭建步骤

1. 安装node和git（可使用Chocolatey)
2. 申请github账户
3. 执行 `npm install -g hexo`
4. 创建一个文件夹，并在该文件夹下执行 `hexo init`
5. 执行 `hexo generate`
6. 执行 `hexo server`，浏览器输入http://localhost:4000预览页面
7. 在github上建立名为your_user_name.github.io的仓库
8. 修改_config.yml：
```
deploy:
  type: git
  repository: git@github.com:yourname/yourname.github.io.git
  branch: master
```
9. 执行 `npm install hexo-deployer-git --save`
10. 部署(每次部署最好执行以下三步)：
```
hexo clean
hexo generate
hexo deploy
```

## hexo常用命令
```
hexo help #查看帮助
hexo init #初始化一个目录
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成网页，可以在 public 目录查看整个网站的文件
hexo server #本地预览，'Ctrl+C'关闭
hexo deploy #部署.deploy目录
hexo clean #清除缓存，**强烈建议每次执行命令前先清理缓存，每次部署前先删除 .deploy 文件夹**
```
对于文章和页面的删除，只需要去hexo_site\source中找到指定的文件，收到删除即可。

#### 参考
1. [http://baixin.io/2015/08/HEXO%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/](http://baixin.io/2015/08/HEXO%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/)
2. [http://www.jianshu.com/p/5e0ca2b14815](http://www.jianshu.com/p/5e0ca2b14815)
3. [http://www.jianshu.com/p/701b1095da11](http://www.jianshu.com/p/701b1095da11)
4. [https://chocolatey.org/](https://chocolatey.org/ "强大的包管理器")