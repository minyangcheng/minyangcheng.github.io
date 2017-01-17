---
title: fiddler抓包体验
date: 2016-11-13 13:26:18
tags: 工具
---
fiddler是一款非常方便的http或https抓包工具。

<!-- more -->

## 下载并安装fiddler

根据安装过程中的提示操作既可。

> fiddler官网: <http://www.telerik.com/fiddler> 

## 配置fiddler

* 配置支持http： 点击<Tool>,选中<Telerik Fiddler Options>,然后在Telerik Fiddler Options界面，将对话框切换到Connections选项卡，然后勾选“Allow romote computers to connect”后面的复选框，然后点击“OK”按钮。
![](/images/fiddler_1.jpg)
* 配置支持https： 在Telerik Fiddler Options界面，将对话框切换到Https选项卡，然后勾选“Allow romote computers to connect”后面的复选框，然后点击“OK”按钮。
![](/images/fiddler_2.jpg)

## 手机网络连接配置

以Android系统为例：打开android设备的“设置”->“WLAN”，找到你要连接的网络，在上面长按，然后选择“修改网络”，弹出网络设置对话框，然后勾选“显示高级选项”。在“代理”后面的输入框选择“手动”，在“代理服务器主机名”后面的输入框输入电脑的ip地址，在“代理服务器端口”后面的输入框输入8888，然后点击“保存”按钮。
![](/images/fiddler_3.jpg)

## https抓包出现的问题

1. 当抓https包时只会出现host为tunnel to，可以用一个叫FiddlerCertMaker.exe的工具重新打了一个证书。[点击下载](http://pan.baidu.com/s/1kV0jr5X)
2. 手机上需要先访问一个https的网址，然后会提示你下载安装证书，默认同意就行了。

## 参考
1. <http://blog.csdn.net/htdeyanlei/article/details/52874248>